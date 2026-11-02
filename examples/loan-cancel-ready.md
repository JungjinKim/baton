> 설명하려고 지어낸 예시다. 도서 대출 시스템, 파일 경로, 줄 번호, 명령 결과는 모두 지어낸 것이다. 읽는 법은 [README](README.md)에 있다.

# loan — 대출 준비 취소

단계: 완료
규모: 정규 (상태 전이·동시성 있음 — 대출 준비 → 예약됨, 두 사서가 같은 예약을 동시에 취소할 수 있음) → 검증 정규
유형: 기능
대상: `src/loans/`
검증 수단: `npm run build` (출처: package.json scripts)
준비: 없음
동기화: 최신
갱신일: 2026-11-02

## 배경
- 예약 상태는 `RESERVED`(예약됨) → `READY`(대출 준비) → `ON_LOAN`(대출 중) 세 값이고, 앞 상태로 돌아가는 전이는 없다 — `src/loans/reservation-status.ts:3-7`
- 대출 준비는 `markReady()`가 상태를 바꾸고 이력 한 줄을 남긴다 — `src/loans/ready.ts:12-30`
- 상태 갱신은 `updateStatus(id, from, to)`가 `WHERE status = from` 조건으로 한다. 0행이면 `AlreadyProcessedError` — `src/loans/reservation-repo.ts:20-34`
- 예약 상세 화면은 상태별로 버튼을 그린다. 대출 준비 상태에는 "대출 처리" 버튼만 있다 — `src/loans/LoanDetail.tsx:30-52`
- `AlreadyProcessedError`는 화면에 "이미 처리됨"으로 보인다 — `src/loans/LoanDetail.tsx:60-66`

## Spec
### AC
1. 대출 준비 상태의 예약에서 사서가 "준비 취소"를 누르면 상태가 예약됨으로 돌아가야 한다
2. 준비 취소 시 취소 사유(필수)가 이력에 남아야 한다
3. 이미 대출 중인 예약에는 "준비 취소" 버튼이 보이지 않아야 한다
### 사람 테스트
| # | 상황 | 기대 |
|---|---|---|
| 1 | 두 창에서 같은 예약을 동시에 취소 | 한쪽만 성공, 다른 쪽은 "이미 처리됨" |
### 전제
- 취소 사유는 자유 입력(선택지 없음). 선택지 방식은 운영 쪽 코드표가 없어 기각
- 준비 취소 뒤 예약 순번은 그대로 둔다(사람의 답). 맨 뒤로 보내는 방식은 사서가 잘못 취소했을 때 회원이 순번을 잃어 기각
### 범위 밖
- 취소 알림(이메일·문자) 발송
### OQ
- 취소 사유의 길이 제한이 필요한가 — 정책이라 이번에 넣지 않는다(`## 검증`의 과잉)

## 계획
### Task 1 — 준비 취소 저장
- AC: 1, 2
- Files: `src/loans/cancel-ready.ts`(새 파일), `src/loans/cancel-ready.test.ts`(새 파일), `src/loans/LoanDetail.tsx`
- Verify: `npm test -- cancel-ready` → 통과 3건 / `grep -c "cancelReady" src/loans/LoanDetail.tsx` → 1

### Task 2 — 대출 준비가 아니면 버튼 숨김
- AC: 3
- Files: `src/loans/LoanDetail.tsx`
- Verify: 확인: `src/loans/LoanDetail.tsx`의 "준비 취소" 버튼 → 대출 준비 상태 조건 블록 안에만 있다(블록 밖 0곳)

## 검증
doubter 1사이클.
- 블로커: Task 2의 Verify가 처음에는 `grep -c "준비 취소" src/loans/LoanDetail.tsx` → 1이었다. 버튼을 조건 없이 그려도 1이 나와 AC 3을 재지 못한다 → `확인:`으로 바꿨다
- 보강: 동시 취소에서 진 쪽의 처리가 계획에 없었다. 기존 `updateStatus()`와 "이미 처리됨" 표시를 그대로 쓰면 된다 → 구현 메모에 적었다
- 과잉: 처음 계획의 "사유 200자 제한"은 요청에 없는 정책성 항목이다 → 빼고 OQ로 옮겼다
- 의문: 없음

## 구현 메모
- 상태는 `updateStatus(id, "READY", "RESERVED")`로만 바꾼다. 조건 갱신이 동시 취소를 막고, 진 쪽은 기존 "이미 처리됨" 표시로 간다(`reservation-repo.ts:20-34`, `LoanDetail.tsx:60-66`)
- 가장 가까운 형제는 `markReady()`(`src/loans/ready.ts:12-30`)다. 이력 기록 순서를 이것과 맞춘다

## 구현
사본: 없음
- [x] Task 1 — `npm test -- cancel-ready` → 통과 3건 / `grep -c "cancelReady" src/loans/LoanDetail.tsx` → 1
- [x] Task 2 — 확인: `src/loans/LoanDetail.tsx:41-48` `canCancelReady` 조건 블록 안에만 "준비 취소" 버튼, 블록 밖 0곳, 충족
- 사유 입력은 기존 `src/common/ReasonDialog.tsx`를 그대로 쓴다. 대출 연장 거절 화면이 같은 방식이다(코드로 정한 세부)
- 빌드: `npm run build` → 성공(종료 코드 0) · 컴파일 실행 확인(로그에 `tsc -p .` 실행 기록) · 새 경고: 0

### 계획 밖 변경
- `src/loans/loan-history.ts` — 이력 항목 타입에 선택 필드 `reason` 추가 · AC 2에 필요. 이력은 기존 `detail` JSON 열에 저장돼 마이그레이션은 없다(`loan-history.ts:20-26`)

### 변경 파일
- `src/loans/cancel-ready.ts` (새 파일)
- `src/loans/cancel-ready.test.ts` (새 파일)
- `src/loans/LoanDetail.tsx`
- `src/loans/loan-history.ts` (계획 밖 변경)

### 수정 1회차
- 리뷰 1회차 블로커: `cancelReady()`가 상태를 바꾸기 전에 `requireLibrarian()`을 부르게 했다 — `src/loans/cancel-ready.ts`. 회원 계정 거부 테스트 1건을 더했다 — `src/loans/cancel-ready.test.ts`
- Verify: `npm test -- cancel-ready` → 통과 4건
- 빌드: `npm run build` → 성공(종료 코드 0) · 컴파일 실행 확인 · 새 경고: 0

## 리뷰 1회차
- 블로커
  - [Critical] `src/loans/cancel-ready.ts:9` — 사서 권한 확인이 없다. 형제 전이 `markReady()`는 먼저 `requireLibrarian()`을 부른다(`src/loans/ready.ts:13`). 실패 시나리오: 회원 계정으로 API를 직접 부르면 대출 준비가 취소된다
- 중요: 없음
- 잔여 리스크·테스트 공백
  - Suggestion: `src/loans/cancel-ready.test.ts` — 동시 취소를 재현하는 테스트가 없다
  - 의문: `src/loans/LoanDetail.tsx:47` — 사유 입력창을 닫으면 아무 일도 없다. AC에 없어 의도인지 알 수 없다
  - 정적 판단이다. 빌드·테스트 통과 여부는 보지 않았다
- 문제없던 각도: 라인 · 삭제된 동작 · 저장소 관행 · AC

## 리뷰 2회차
범위: 수정 1회차 변경분(`src/loans/cancel-ready.ts`, `src/loans/cancel-ready.test.ts`)
- 블로커: 없음
- 중요: 없음
- 잔여 리스크·테스트 공백
  - 정적 판단이다. 빌드·테스트 통과 여부는 보지 않았다
- 문제없던 각도: 여섯 각도 모두. `requireLibrarian()`을 상태 갱신 전에 부르는 순서가 `src/loans/ready.ts:13`과 같다

## 인계
대출 준비 상태에서 사서가 사유를 남기고 준비를 취소할 수 있게 했고, 리뷰 2회차에서 블로커가 0이 됐다.

무관 변경(implement 기록): `src/members/MemberList.tsx` — 이번 작업 전부터 있던 미커밋 변경. 커밋 초안에서 뺀다.

### 계획 밖 변경
- `src/loans/loan-history.ts` — 이력 항목 타입에 선택 필드 `reason` 추가 · AC 2에 필요
- 되돌리려면 `/baton:inspect loan-cancel-ready <되돌릴 것>`

### AC 판정
| AC | 판정 | 증거 |
|---|---|---|
| 1 | PASS | `npm test -- cancel-ready` → 통과 4건 |
| 2 | PASS | 같은 실행의 "이력에 사유가 남는다" 케이스 |
| 3 | PASS | 확인: `src/loans/LoanDetail.tsx:41-48` |

### 잔여 리스크
- 동시 취소는 테스트가 없고 읽기로만 확인했다 → 사람 테스트 1번
- 이력 항목 타입이 바뀌었다(계획 밖 변경). 이력 화면은 고치지 않았다 → 사람 테스트 3번
- 사유 입력창을 닫으면 아무 일도 없다(리뷰 1회차 의문). AC에 없어 그대로 뒀다

### 리뷰 기준
`.claude/work/scratch/loan-cancel-ready.review-state.md`

### 사람 테스트
| # | 해 볼 것 | 기대 | 결과 |
|---|---|---|---|
| 1 | 두 창에서 같은 예약을 동시에 "준비 취소"한다 | 한쪽만 성공하고, 다른 쪽은 "이미 처리됨"이 뜬다 | PASS |
| 2 | 준비 취소한 예약에서 다시 "대출 준비"를 누른다 | 대출 준비 상태로 바뀐다 | PASS |
| 3 | 준비 취소를 한 적 없는 예약의 이력 화면을 연다 | 이전 이력이 전과 같이 보인다 | 미검증 |

### 커밋 초안
- 메시지: `loans: 대출 준비 취소 추가`
- add: `git add src/loans/cancel-ready.ts src/loans/cancel-ready.test.ts src/loans/LoanDetail.tsx src/loans/loan-history.ts`

### OQ · 정책성 보류 · 범위 밖 발견
- OQ: 취소 사유의 길이 제한
- 범위 밖: 취소 알림 발송

테스트 후 `/baton:signoff loan-cancel-ready`, 결과물 지적은 `/baton:inspect loan-cancel-ready <기대와 달랐던 것>`, AC 변경은 `/baton:draft loan-cancel-ready …`

## 종결
- 리뷰 기준: signoff 진입 때 대조, 현재 파일과 같다
- 사람 테스트: 1 PASS · 2 PASS · 3 미검증(안 함)
- 커밋: 사람 커밋 예정
