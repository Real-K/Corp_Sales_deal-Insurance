# 애매한 모델링 결정

![](orca-paste-1792492291178-a518c348-bf3c-4016-85b4-86a4f2fdb4b1.png)

## 결정 1

- **개념:** 딜(허브). 법인 하나가 업무 A→E를 지나는 단위
- **고려한 대안:**
  - (가) 업무 하나만 World로 만들고 딜은 World 밖에 둔다(지금 허브처럼 읽기 전용 계산값)
  - (나) 딜 전체 A→E를 하나의 World로 만들고 딜을 주인공 상태기계로 둔다
- **최종 선택:** (나)
- **이유:** 7문항 중 1번(다시 돌아오는 대상 = 딜)이 결정적이었다. 지금 하네스는 업무 사이 순서를 사람과 적재 프로그램이 지키고, 딜의 단계는 기록되지 않는 계산값이다. 딜을 상태기계로 만들면 순서가 State 흐름 자체로 강제된다. 대가는 업무 안의 세부 통제(자기 채점 금지, 주장 라벨, 검사한 판 묶기)를 이 World에서 표현하지 않는 것이다.
- **ESTC에 미친 영향:**
  - Entity: 주인공이 Deal이 되고, 정체성은 `deal_id`로 세운다
  - State: 업무 A~E가 딜의 단계 10개가 된다
  - Transition: 업무 사이 인계(예: `report_confirmed` → E 확정)가 전이가 된다
  - Constraint: C8처럼 업무를 넘나드는 규칙을 쓸 수 있다

## 결정 2

- **개념:** 업무 건 A~E (PrepCase · FollowupCase · DiagCase · ReportCase · ProposalCase)
- **고려한 대안:**
  - (a) 딜의 State로 녹인다
  - (b) 판 번호가 있는 산출물만 상태 없는 Entity로 둔다
  - (c) 업무 건 다섯 개 모두 Entity로 둔다
- **최종 선택:** (b). C의 1차 진단은 `ExpertRequest`로 합치고, B는 Entity로 두지 않는다.
- **이유:** (b)를 고른 이유는 셋이다.
  1. 이 World에서 지켜야 할 규칙의 핵심은 판 번호 대조다. 승인한 판으로만 발송하는 것(C3), 확정된 보고서 판을 근거로 설계하는 것(C8)이 그렇다. 판 번호와 법인 값(C10)을 담을 곳이 산출물에 있어야 규칙을 식으로 쓸 수 있다.
  2. 상태기계는 딜 하나에만 둔다. 업무 건마다 상태를 두면 업무 사이 순서가 다시 여러 상태기계 사이의 약속이 되고, 결정 1의 이유(순서를 State 흐름으로 강제한다)가 무너진다. 그래서 산출물은 상태 없는 Entity로 둔다.
  3. 판 번호가 없는 것은 Entity로 만들지 않는다. B는 승인도 판도 없는 수령 확인이라 Deal 속성(meeting_held)과 전이 하나로 충분하다. C의 1차 진단과 배정 요청은 같은 승인 판으로 함께 묶여 나가므로 ExpertRequest 하나로 합쳤다.
  대가: 산출물마다 따로 있던 상태(초안·검사·반려)는 이 World에서 보이지 않는다. 이 부분은 실제 업무 World(업무별 명세)가 맡는다고 본다.
- **ESTC에 미친 영향:**
  - Entity: 산출물 Entity 4개(AnalysisReport, ExpertRequest, CEOReport, ProposalDesign), 모두 상태 없음
  - State: 상태는 Deal에만 있다
  - Transition: 변화 없음
  - Constraint: C3 · C8 · C10이 산출물 속성(판 번호·법인)을 참조한다

## 결정 3

- **개념:** 승인(확정)
- **고려한 대안:**
  - State (승인됨)
  - Transition (승인하다)
  - 별도 Entity (승인 기록)
- **최종 선택:** Transition
- **이유:** 이유는 둘이다.
  - 승인 주체가 금액에 따라 바뀐다. 기준액 이하는 GFC(confirm_proposal), 초과는 부서장(approve_proposal_over_limit)이 확정한다. 이 갈림은 같은 from·to를 가진 두 전이와 역할로 표현하는 것이 가장 단순하다.
  - 승인을 State로 두면 상태가 늘어나고, 별도 Entity로 두면 딜과 승인 건의 상태를 다시 맞춰야 한다. 둘 다 결정 1·2의 방향과 맞지 않는다.
- **ESTC에 미친 영향:**
  - Entity: 승인 Entity 없음. 승인·확정한 판은 산출물 속성(`approved_version`, `confirmed_version`)
  - State: 승인 결과는 `*_confirmed` / `diagnosis_approved` 상태로 남는다
  - Transition: E 확정이 `confirm_proposal`(gfc)과 `approve_proposal_over_limit`(dept_head) 두 전이로 갈린다
  - Constraint: C9의 `on_fail_hint`가 두 전이를 잇는다

## 결정 4

- **개념:** 해석에 따라 결과가 달라지는 규칙과 값. v1 내장 함수 `no_payment_attempt_with_key`, 없는 값, 뜻이 정의되지 않은 값
- **고려한 대안:**
  - 유지하고 전제를 주석으로 밝힌다(C5를 남기고 "Runtime이 결제 아닌 효과도 시도로 센다고 가정"이라고 적는다)
  - 해석에 기대는 규칙은 지우고, 값은 seed에서 빠짐없이 채우고, 뜻은 world에 정의한다
- **최종 선택:** 두 번째. C5와 `idempotency_key`를 지우고, seed의 모든 레코드가 선언된 속성을 모두 갖게 하고, `world.yaml`에 값과 식의 해석 약속을 적었다.
- **이유:** 판정은 World와 seed만으로 한 가지로 정해져야 한다. 전제를 밝혀 두어도 그 전제가 Runtime과 다르면, 판정과 그 판정에 기댄 모든 설명이 함께 바뀐다. 같은 딜 안의 재발송은 상태 흐름이 확정적으로 막으므로 C5를 지워도 잃는 통제가 없다. 딜 사이 중복은 어차피 v1로 표현할 수 없다(break-fix 5절).
- **ESTC에 미친 영향:**
  - Entity: `ExpertRequest.idempotency_key`와 쓰이지 않는 속성(`confirmed_by`, `round`, 산출물 3종의 `corp`) 삭제. 속성마다 뜻·단위·쓰는 주체를 주석으로 적음
  - State: 변화 없음
  - Transition: `send_expert_request`의 guard가 `[C3, C4]`가 됨
  - Constraint: C5 삭제(13 → 12). 시나리오 because에는 guard 전부의 참·거짓과 실패 목록을 적는다

