# Open Questions & 결정 이력 (v4 — 2026-07-08)

> 상태: ✅ 결정 / ❓ 미결. **설계상 미결 항목 0건 — 잔여는 외부 의존성만.**

## A. 결정 완료 (이력)

| # | 항목 | 결정 |
|---|---|---|
| A1 | Tmax 수신 방식 | 별도 웹훅 endpoint |
| A2 | fallback | unmatched 전용 채널 신설 (매핑 백로그 작업큐 겸용) |
| A3 | Control-M 중복 정책 | exclude 필터 구조만 구현, 정책값 빈 상태 시작 (event_name='CTRLM' 축 권장) |
| A4 | 매핑 저장소 | KV 단일 JSON blob + 캐시. KV 단일 진실, 웹뷰가 유일 쓰기 경로 |
| A5 | 기존 베이스 | Go MM 플러그인(dmove-sycros), Pull(DB 폴링) 구조 |
| A6 | Tmax payload | 봉투 고정 + body 유동 |
| A7 | Tmax 심각도 | 없음 → 전건 critical |
| A8 | HA | 2노드. TTL 캐시 수렴, KV 선착순 선점, 리더 선출 |
| A9 | Sycros 볼륨 | 일 최대 1만 / 피크 분당 1,000 (원천 발생량) |
| A10 | 심각도 분포 | critical/fatal 저빈도 |
| A11 | 스레드 요건 | 호스트 단위 묶음 |
| A12 | 개발 주체 | 자체 개발(Claude Code), PRD는 cowork-requirements repo |
| A13 | 폴링 테이블 | event_text_hist_YYYYMMDD, 자정 생성 |
| A14 | 신규 기능 성격 | 기존 DM·state 라우팅 영구 유지(무변경), 신규는 순수 추가 레이어 |
| A15 | event_id 포맷 | 18자리 zero-padded — 문자열 비교 안전 |
| A16 | state 코드 | 1 normal / 2 warning / 3 critical / 4 fatal |
| A17 | 자정 롤오버 | 현행 동작 수용(동결) |
| A18 | 크레덴셜 | 테스트 더미 확인 — 위생 정리로 하향 |
| A19 | DM 파이프라인 | 무변경 확정 |
| A20 | 관리화면 | B(웹뷰) 단독. 기존 System Console UI 무변경 |
| A21 | 스레드 reply 간소화 | 동의 |
| A22 | Tmax event_id | 원천에 없음 → 플러그인 채번 |
| A23 | event_id 뒷 4자리 | 같은 초 적재 시퀀스 — 스레딩은 플러그인이 구현 |
| A24 | Jira 티켓 생성 | 채널 옵트인(우선 IT부문) |
| A25 | 스레드 윈도우 | 슬라이딩 120초 |
| A26 | 미달 심각도 | warning 이하·회복 이벤트 완전 미표시 |
| A27 | reply 포맷 | 루트 포맷에서 헤더 라인만 제거, 뱃지 유지 |
| A28 | Jira 생성 방식 | 스레드 루트당 1티켓, 비동기 best-effort, 링크를 스레드 reply |
| A29 | Jira 호출 경로 | ScriptRunner 커스텀 endpoint (기존 taskList 예제와 동일 방식, 자체 작성) |
| A30 | Tmax 업무코드 | body 2번째 라인 첫 2글자 대문자 치환 (lnhb0003u0 → LN) |
| A31 | exception 컬럼 | 무시 |
| A32 | severity 정책 | critical 이상 고정으로 시작 |
| A33 | Tmax 볼륨 | 일 1,000건 이내 |
| A34 | 이중발송 가드 | **불채택** — 기존 state 채널과 신규 매핑 채널은 별도 채널로 분리 운영. 겹침 등록 시 이중 발송은 문서화된 동작 |
| A35 | 스레드 시간 기준 | **create_date** — 판정은 이벤트 create_date끼리 비교(서버 시계 혼용 금지) |
| A36 | 스레드 캡 | 30분 또는 reply 200건 도달 시 새 루트 (B7 승인) |
| A37 | Jira 인증·설정 | API 계정 PAT(보안 협의 후 발급). endpoint URL·PAT는 **관리화면 설정 탭 입력** → 매핑과 분리된 KV(integration:jira) 저장, export/import 제외 |
| A38 | MM→Jira 방화벽 | 열려 있음 (확인 완료) |

## B. 외부 의존성 (구현과 병행 가능)

| # | 항목 | 상대 | 차단 여부 |
|---|---|---|---|
| D1 | exclude 정책값 (event_name='CTRLM' 축 기준) | 민석 책임 | 비차단 — 구조만 구현, 값은 추후 |
| D2 | ingest 토큰/IP 정책 | 정보보안팀 | 비차단 — 토큰 방식 우선 구현 |
| D3 | 조직 채널 생성·멤버 운영 주체 | 미정 | 비차단 — 도입 단계 이슈 |
| D4 | Tmax body 2번째 라인(잡코드 시작) 포맷 불변 합의 + 봉투 계약 | Tmax 인터페이스 담당 | **Tmax 기능 배포 전 필수** |
| D5 | API 계정 PAT 발급 (보안 협의) + ScriptRunner endpoint 작성 | 정보보안팀 / 본인 | **Jira 기능 배포 전 필수** (개발은 mock으로 병행 가능) |
