# CLAUDE.md — mm-plugins-Sycros 개편 프로젝트

> 이 파일은 Claude Code가 본 저장소에서 작업할 때 항상 참조하는 컨텍스트다.
> 요구사항 원본: `cowork-requirements` repo의 01~04 문서 (AS-IS 분석 / TO-BE 아키텍처 / 인터페이스·데이터모델 / Open Questions).

## 프로젝트 개요

미래에셋증권 사내 Mattermost용 알람 라우팅 플러그인. Sycros(인프라 모니터링) DB를 폴링하고 Tmax 알람을 웹훅으로 수신하여, 조직(팀/본부) 단위 채널로 M:N 라우팅한다. Slack + 차세대 모니터링 전환 전 프로토타입 — 단순함 > 완벽함, 단 알람 유실·중복은 타협 불가.

- plugin id: `dmove-sycros` — **절대 변경 금지** (KV 네임스페이스·설정 단절됨)
- 대상 환경: 온프레미스 Mattermost, **2노드 HA 클러스터**, 폐쇄망
- Go 1.24 / mattermost-plugin-starter-template 구조 / server 중심 (webapp은 관리 UI 한정)

## 빌드·테스트 명령

```bash
make dist          # 플러그인 번들 빌드 → dist/dmove-sycros-*.tar.gz
make all           # 린트 + 테스트 + 빌드
make test          # go test
make check-style   # 린트
MM_DEBUG=1 make dist  # 디버그 빌드
```

로컬 배포: `MM_SERVICESETTINGS_SITEURL`, `MM_ADMIN_TOKEN` 환경변수 후 `make deploy`.

## 아키텍처 원칙 (TO-BE)

```
Source Adapter (Sycros Poller | Tmax Webhook)
  → Normalize(AlarmEvent) → Filter(critical 이상만) → Route(mapping) → ThreadGroup(슬라이딩 120초)
  → Dispatch → (+ Jira Sink: ScriptRunner 커스텀 endpoint, 스레드 루트당 1티켓, 비동기 best-effort)
```

1. **어댑터는 수신·정규화만.** 라우팅 이후는 소스 무관 공용 파이프라인. 새 소스 추가 시 어댑터 1개 + 매핑 rule만 늘어나야 한다.
2. **매핑은 KV가 단일 진실** (`mapping:v1` JSON blob). config에 매핑을 넣지 말 것 — config 저장은 폴링 재시작을 유발한다(OnConfigurationChange → restartPolling).
3. **HA 필수 패턴**: 공유 상태 선점은 전부 `KVSetWithOptions{Atomic: true, ExpireInSeconds: N}`. in-memory 캐시는 TTL(30~60s) 수렴 허용. 주기 작업은 리더 선출(cluster.Schedule). 두 노드가 동시에 같은 작업을 한다고 가정하고 작성할 것.
4. **유실 < 중복**: KV 오류 등 판단 불가 상황에서는 중복 발송을 택한다 (fail-open). 단 반드시 로그를 남긴다.
5. 발송은 bounded 동시성 (semaphore, 설정값). 개별 실패 격리 — 한 채널 실패가 다른 채널 발송을 막지 않는다.
6. 미매칭 알람은 버리지 않는다 → unmatched 채널 + 구조화 로그.

## 절대 규칙 (보안)

- **크레덴셜·IP·비밀번호를 코드/로그에 절대 하드코딩하지 않는다.** 기존 configuration.go의 하드코딩 Crowd DB 접속정보와 password_hint 로그는 발견 즉시 제거 대상. 커밋 전 diff에서 secret 검사.
- SQL 식별자(테이블/컬럼명)는 화이트리스트 검증 또는 quoting 후 사용. 값은 항상 파라미터 바인딩.
- ingest 웹훅은 토큰 인증 필수. 관리 API는 `Mattermost-User-Id` + System Admin 검증 필수.
- 폐쇄망: 신규 외부 Go 모듈·npm 패키지 추가는 사전 협의 (사내 레지스트리 반입 절차 필요). 현 go.mod 범위 내 우선 해결.

## 알려진 지뢰 (AS-IS 결함 — 수정 대상)

- 자정 롤오버의 어제 테이블 잔여분 스킵은 **의도적으로 동결된 동작** — 수정 금지 (결정 A17)
- event_id는 18자리 zero-padded 고정폭(적재시각+초당 시퀀스 4자리) — 문자열 비교 안전, 교정 불필요. 뒷 4자리는 호스트 그룹핑이 아니라 같은 초 시퀀스
- 폴링 N+1 쿼리 (목록 후 row별 SELECT *) → 단일 쿼리화
- 일별 테이블(event_text_hist_YYYYMMDD)은 자정 생성 확정 — 부재 시 에러가 아닌 정상 대기로 처리
- 스레드 윈도우 시간 기준은 **create_date** (결정 A35). 판정은 항상 이벤트 create_date끼리 비교 — 서버 시계와 혼용 금지

## 코드 컨벤션

- 기존 로그 스타일 유지: `p.API.Log{Info,Warn,Error}(msg, k, v, ...)` 구조화 키-값. 알람은 event_id로 전 구간 추적 가능해야 한다.
- plugin.go 단일 파일 비대화 금지 — 신규 코드는 `server/adapter/`, `server/pipeline/`, `server/mapping/`, `server/admin/` 등으로 분리.
- 주석·로그 메시지는 기존 관례대로 한국어 주석 허용, 로그 키는 영문 snake_case.
- 테스트: 라우팅·정규화·윈도우 판정 등 순수 로직은 반드시 단위 테스트. KV/API는 plugintest mock 사용.
- DM 파이프라인(tb_kko_tran 폴링)과 기존 state 기반 채널 라우팅은 **영구 동결(무변경)** — 신규 기능은 순수 추가 레이어로만 구현하고, 기존 동작·설정(notification_channels)·System Console UI를 변경하지 않는다. 기존 state 채널과 신규 매핑 채널은 별도 채널로 분리 운영(결정 A34) — 이중발송 가드는 구현하지 않으며, 같은 채널 중복 등록 시 이중 발송은 문서화된 동작이다.

## 작업 시 확인 습관

1. 설계 변경이 Open Questions(04 문서)의 미결 항목과 충돌하면 구현 전에 질문할 것.
2. HA 시나리오("다른 노드가 동시에 이 코드를 실행하면?")를 PR 설명에 한 줄 명시.
3. 매핑 스키마(`mapping:v1`) 변경 시 version 마이그레이션 경로 포함.
