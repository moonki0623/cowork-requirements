# Sycros MM 플러그인 개편 TO-BE 아키텍처 설계서 (v2)

> 전제: AS-IS 분석서(01) 및 2026-07-08 질의응답 결정 반영.
> 상태 표기 — ✅ 합의 완료 / 🔶 제안(승인 대기) / ❓ 미결(04 문서 참조)

## 1. 설계 목표 및 기본 원칙

**기본 원칙: 본 개편은 순수 추가(additive) 피처다.** 기존 DM 파이프라인과 state 기반 채널 라우팅은 무변경으로 영구 유지되며 ✅, 신규 기능은 그 옆에 독립 레이어로 얹힌다. 기존 코드 경로의 동작 변경은 금지.

1. **조직 단위 M:N 라우팅 신규 추가** (source+key 매핑) — 기존 state 라우팅과 병렬 공존
2. 알람 소스 확장: Sycros(DB 폴링, 기존) + **Tmax(웹훅 중계, 신규)** — 3번째 소스 추가에 스키마 변경 없음
3. **호스트 단위 스레드 묶음** (슬라이딩 1분 윈도우)
4. **매핑 관리 웹뷰** (B안 단독 ✅ — 슬래시 커맨드 불채택, 기존 System Console UI는 무변경 유지)
5. unmatched 전용 채널로 미매핑 알람 유실 방지 ✅
6. **Jira 티켓 자동 생성** (채널 단위 옵트인, 우선 IT부문 채널) — 신규 요구 ✅
7. HA 2노드 정합성 유지 (기존 row 선점 패턴 계승)

## 2. 파이프라인 아키텍처

```
[Source Adapters]                 [신규 라우팅 파이프라인]              [Sinks]
┌───────────────────┐
│ Sycros Poller     │──┬──▶ 기존 state 라우팅 (무변경 유지) ──────────▶ 기존 state 채널
│ (event_text_hist) │  │
└───────────────────┘  └──▶ ┌────────────────────────────────┐
┌───────────────────┐       │ Normalize → AlarmEvent         │      ┌──────────────┐
│ Tmax Webhook      │──────▶│ Filter (severity/exclude)      │─────▶│ Dispatcher   │─▶ 매핑 채널들
│ (신규 HTTP 수신)   │       │ Route  (mapping: source+key)   │      │ (bounded)    │─▶ unmatched 채널
└───────────────────┘       │ Group  (thread window)         │      │              │─▶ Jira 티켓
                            └────────────────────────────────┘      └──────────────┘─▶ 감사/heartbeat
DM 파이프라인(tb_kko_tran): 완전 동결, 위 그림과 무관 ✅
```

핵심: 어댑터는 수신·정규화만, 라우팅 이후(스레딩·발송·fallback·Jira·로깅)는 소스 무관 공용.

## 3. 컴포넌트 설계

### 3-1. Sycros Poller (기존 개선)

- 폴링 루프·위치 관리·HA row 선점·자정 롤오버 동작은 AS-IS 그대로 계승 ✅ (롤오버 동결 결정 A17)
- 개선: 목록+데이터 단일 쿼리화(N+1 제거) 🔶. event_id 비교는 현행 문자열 비교 유지 (18자리 zero-padded 확인, A15)
- state → severity 정규화: 1 normal / 2 warning / 3 critical / 4 fatal ✅ (A16)
- 신규 라우팅으로의 분기는 기존 발송 경로와 독립 — 기존 state 채널 발송 코드는 건드리지 않음

### 3-2. Tmax Webhook Adapter (신규)

- `ServeHTTP` 훅 연결 + gorilla/mux (기존 api/ 스텁 골격 재활용)
- `POST /plugins/dmove-sycros/api/v1/ingest/tmax` — 소스별 별도 endpoint ✅
- 인증: 공유 토큰 헤더(config secret) + 선택적 IP allowlist 🔶
- 봉투 고정 + body 유동 ✅. **Tmax는 event_id 없음 → 플러그인이 수신 시 채번** ✅ (A22)
- Tmax는 심각도 없음 → 전건 critical ✅
- **service_code 획득 방법 미결** ❓ (N1): 중계기 필드 송신 vs 잡코드 앞 2글자 추출 vs 본문 파싱

### 3-3. Router (신규)

- `(source, key) → [channel_ref...]` M:N ✅. 키 정규화: lower + trim (+Sycros FQDN strip 여부 실데이터 확인)
- 미매칭 → unmatched 채널 + 구조화 로그 ✅
- 매핑 캐시: in-memory + TTL 30~60초 자연 수렴 ✅
- exclude 필터: **필드 지정 문법** 지원 (`event_name: CTRLM` 등) 🔶 — 첨부 실데이터에서 Control-M은 event_name='CTRLM'으로 정확 식별됨. 정책값은 빈 상태 시작(민석 책임 협의 후, A3)
- severity 필터: 채널/전역 `severity_min` (기본 critical) ❓ (N5)
- 파이프라인 간 이중발송 가드: **불채택** ✅ (A34) — 기존 state 채널과 신규 매핑 채널은 **별도 채널로 분리 운영**(운영 규칙). 같은 채널을 양쪽에 등록하면 이중 발송되며, 이는 버그가 아닌 문서화된 동작

### 3-4. Thread Grouper (신규)

- 묶음 키: `(channel_id, source, routing_key)` ✅ — 호스트 단위, 세분화는 Phase 2 설정값
- 윈도우: **슬라이딩 120초** ✅ (B1 확정 — 마지막 이벤트 기준 연장) + 무한 성장 방지 캡(30분 또는 reply 200건) ✅ (B7 확정)
- 기준 시각: **create_date** ✅ (B6 확정) — 120초 판정은 항상 이벤트 create_date끼리 비교하며 서버 시계와 혼용하지 않음. 스레드 키가 호스트 단위라 호스트 내 시계 일관성은 유지됨
- **미달 심각도 처리** ✅ (B4 확정): severity_min(critical) 미달 알람(warning·normal)은 발송·스레드 부착 모두 하지 않음 — 완전 미표시. 회복 이벤트 특수 처리 없음
- reply 포맷: 루트와 동일 포맷에서 헤더 라인만 제거, 심각도 뱃지 유지 ✅ (상세 03 문서 §5)
- HA 경합: 루트 post_id를 KV 선착순 도장(set-if-absent)으로 선점 ✅. 스레드 상태는 채널별 독립 ✅

### 3-5. Dispatcher

- bounded 동시성(semaphore, 설정값 기본 16) 🔶, 개별 실패 격리
- 채널 아카이브/삭제 실패 시 unmatched 채널 우회 + 관리 알림 🔶
- 발송 결과 구조화 로그 (event_id 추적)

### 3-6. Jira Ticket Sink (신규 ✅ — A24, B5 승인)

- 채널 속성 옵트인 — 우선 IT부문 채널만 활성
- **생성 단위: 스레드 루트당 1티켓** ✅ — 새 인시던트(루트 post) 시점에만 생성, 티켓 링크를 스레드에 reply
- 격리: MM 발송 완료 후 비동기 goroutine, 재시도 1회, 실패 시 감사 채널 알림 — Jira 장애가 알람 발송을 막지 않음
- **호출 대상: ScriptRunner 커스텀 endpoint** ✅ (자체 작성 — 기존 taskList 예제와 동일 방식). 계약은 03 문서 §6
- 인증: **API 계정 PAT(Bearer)** ✅ (N9 — 보안 협의 후 발급 예정). endpoint URL과 PAT는 **관리화면 설정 탭에서 입력**, 매핑과 분리된 KV(`integration:jira`)에 저장하고 export/import에서 제외
- project_key·issue_type은 요청 파라미터로 유지(범용형) — 채널 확장 시 endpoint 재작성 불필요
- MM 서버→Jira 방화벽: **확인 완료(열려 있음)** ✅ (N10)
- Phase 2: reply 코멘트 미러링, 자동 상태 전환

### 3-7. 매핑 관리 웹뷰 (B안 단독 ✅ — A20)

- 저장소: KV `mapping:v1` 단일 JSON blob (config 금지 — 재시작 부작용), version 필드 optimistic lock ✅
- 화면: 매핑 규칙(조회/검색/CRUD) / 채널 관리(채널 실체·priority·exclude·Jira 설정·역할) / 상태·테스트(health 카드, 라우팅 dry-run + 테스트 발송, 변경 이력)
- import/export(JSON) ✅, 감사 로그는 지정 채널 bot post ✅
- 권한: `Mattermost-User-Id` + System Admin (Phase 1) ✅
- 기존 System Console ChannelSettings UI(state 라우팅용)는 무변경 유지 ✅ (A20)

### 3-8. 관측성

- Heartbeat 일 1회 (cluster.Schedule 리더 선출 🔶), tick당 수신/매칭/발송/실패/미매핑 카운트 로그

## 4. HA 정합성 체크리스트 (2노드 ✅)

| 상태 | 방식 |
|---|---|
| 폴링 위치 / row 선점 | AS-IS 유지 (KVSetWithOptions Atomic+TTL) |
| 매핑 캐시 | TTL 자연 수렴 (30~60초) |
| 스레드 루트 | KV set-if-absent 선점 |
| Jira 생성 | 스레드 루트 선점 승자 노드만 수행 (별도 경합 없음) |
| 주기 작업 | cluster.Schedule 리더 선출 |

## 5. 단계 범위

**Phase 1**: source+key M:N 라우팅(state 라우팅과 공존, 채널 분리 운영), Tmax 웹훅 수신, unmatched 채널, 스레드 묶음, 매핑 KV + 관리 웹뷰 + 감사로그 + import/export, Jira 티켓(스레드 루트당 1건, IT부문), bounded 발송, heartbeat, 구조화 로깅, N+1 쿼리 개선.

**Phase 2**: 팀별 셀프서비스 권한, 스레드 키 세분화, Jira 코멘트 미러링·자동 종결, dedup 고도화, 폭주 시 채널당 collapse.

**범위 외(동결)**: DM 파이프라인 ✅, state 라우팅 로직 ✅, 자정 롤오버 동작 ✅.

## 6. 공존 및 도입 전략

1. plugin id `dmove-sycros` 유지 (KV 네임스페이스 보존).
2. **기존 state 채널은 정리 대상이 아니다** — 신규 매핑 라우팅과 영구 공존 ✅ (A14). 마이그레이션·컷오버 개념 없음, 신규 채널을 점진 등록하며 확장.
3. 기존 state 채널과 신규 매핑 채널은 별도 채널로 분리 운영(A34) — 겹침 등록 시 이중 발송은 알려진 동작.
4. DM 파이프라인은 회귀 테스트에서 "동작 불변" 검증만 수행.

## 7. 비기능 요구사항

- 폐쇄망: 신규 외부 의존성 금지, 현 go.mod 범위 내 구현
- 성능: 피크 분당 1,000건 유입 가정, 배치 크기·tick·발송 동시성 설정값 노출
- 보안: ingest 토큰 인증, SQL 식별자 quoting, 크레덴셜 config(secret) 주입(더미 하드코딩 정리 포함)
- 로깅: event_id 기준 전 구간 추적 가능
