# Break → Fix → Re-run

## 1. BREAK 시나리오

| 파일 | 시도 | 결과 |
|---|---|---|
| `scenarios/adversarial.yaml` | CEO 보고서 확정 전에 설계안 확정 | 처음부터 DENY(상태). 업무 사이 순서는 State 흐름이 막는다 |
| `scenarios/adversarial-stale-report-version.yaml` | 확정된 보고서(v1)가 아닌 판(v2)을 근거로 한 설계안 확정 | 처음부터 DENY(C8) |
| `scenarios/adversarial-head-bypass.yaml` | 기준액 이하 설계안을 부서장 경로로 확정 | 빈틈 발견 → BREAK 1 |
| `scenarios/adversarial-revised-report.yaml` | 확정 뒤 고쳐진 보고서 판을 근거로 한 설계안 확정 | 빈틈 발견 → BREAK 2 |
| `scenarios/adversarial-other-dept-head.yaml` | 기준액 초과 설계안을 다른 부서장이 결재 | BREAK 1의 FIX에서 생긴 C12 확인 |
| `scenarios/adversarial-report-changed-after-confirm.yaml` | 확정본을 근거로 했지만 확정 뒤 보고서가 바뀐 설계안 확정 | BREAK 2의 FIX에서 생긴 C13 단독 확인 |

## 2. BREAK 1 (부서장 경로 우회)

### BEFORE

| 항목 | 기록 |
|---|---|
| 시나리오 (파일 · 이름) | `scenarios/adversarial-head-bypass.yaml` · `under-limit-proposal-via-dept-head` |
| 지키려는 안전 속성 | 기준액 이하 설계안은 담당 GFC만 확정한다. 부서장 결재 경로로 우회할 수 없다 |
| 시작 레코드와 상태 | D-2003 · `report_confirmed` (설계안 연 환산 보험료 합계 80,000,000원, 기준액 예시 1억 원 이하) |
| 행위자 / 역할 | head-01 / dept_head |
| 시도한 Transition | `approve_proposal_over_limit` (report_confirmed → proposal_confirmed) |
| 관련 Constraint | `C8_uses_confirmed_report` 하나뿐. `confirm_proposal`에 걸린 C1·C9는 이 전이에 없다 |
| 예상 판정 | **ALLOW** |
| 이유 (desk-check) | 상태 report_confirmed = from ✓ · 역할 dept_head ✓ · C8: report_version 1 == ceo_report.version 1 ✓ · global 없음 → ALLOW |
| 문제 | "기준액 초과" 여부를 보는 guard가 빠져 있었다. 같은 from·to를 가진 두 전이 중 confirm_proposal에만 기준액 규칙(C9)이 있어서, 부서장 경로로 돌아가면 담당 GFC 확정과 기준액 규칙을 모두 건너뛴다 |

### FIX

- 바꾼 것: 새 Constraint 2개(`C11_over_limit_only`, `C12_deal_dept_head_only`)를 `approve_proposal_over_limit`의 guard에 추가했다. 이를 위해 Deal 속성 `dept_head`와 행위자 `head-02`(다른 부서장)를 추가하고, seed의 딜에 `dept_head: head-01`을 넣었다. 수정 방식은 guard 추가이고 상태 구조는 바꾸지 않았다.
- 변경 전 → 변경 후:

```yaml
# before
entities:
  Deal:
    attributes:
      gfc: string
transitions:
  approve_proposal_over_limit:
    from: report_confirmed
    to: proposal_confirmed
    principal_role: dept_head
    guards: [C8_uses_confirmed_report]

# after
entities:
  Deal:
    attributes:
      gfc: string
      dept_head: string            # 이 딜의 부서장 principal id. 기준액 초과 설계안의 결재자
transitions:
  approve_proposal_over_limit:
    from: report_confirmed
    to: proposal_confirmed
    principal_role: dept_head
    guards: [C8_uses_confirmed_report, C11_over_limit_only, C12_deal_dept_head_only]
constraints:
  C11_over_limit_only:
    expr: deal.proposal.premium_total > 100000000
    message: 기준액(예시 1억 원) 이하 설계안은 담당 GFC가 확정한다.
  C12_deal_dept_head_only:
    expr: principal.id == deal.dept_head
    message: 그 딜의 부서장만 기준액 초과 설계안을 결재할 수 있다.
principals:
  - id: head-02
    name: 다른 부서장 (예시)
    roles: [dept_head]
```

- validate: `evidence/validate-before.txt`(Constraints 10)와 `evidence/validate-after.txt`(Constraints 12) 모두 READY FOR STUDIO IMPORT ✓. 수정 전 World도 READY였다. valid했지만 안전하지 않았다.
- 막지 않은 것: 기준액 초과 설계안은 여전히 담당 GFC 확정 없이 부서장 혼자 확정한다(2단 결재 아님). 다음 단계 후보로 남긴다.

### AFTER

| 항목 | 기록 |
|---|---|
| 같은 시나리오 | `scenarios/adversarial-head-bypass.yaml` · `under-limit-proposal-via-dept-head` (D-2003, head-01, `approve_proposal_over_limit`) |
| 새 예상 판정 | **DENY** |
| 이유 | 상태 report_confirmed = from ✓ · 역할 dept_head ✓ · C8: report_version(1) == ceo_report.version(1) → 참 · **C11: premium_total(80,000,000) > 100,000,000 → 거짓** · C12: head-01 == deal.dept_head(head-01) → 참 · 실패 목록: C11 → DENY. 판정을 바꾸는 규칙은 C11이다. BEFORE에는 상태·역할·C8만 보고 ALLOW였기 때문에 기준액 이하 설계안이 담당 GFC 확정(C1)과 기준액 검사(C9)를 건너뛸 수 있었다. AFTER에는 부서장 경로가 기준액 초과 건에만 열리고, 기준액 이하 건이 확정되는 길은 confirm_proposal(C1·C8·C9) 하나뿐이다. 이 표는 FIX 1 직후 기준이다. BREAK 2의 FIX 뒤에는 C8이 confirmed_version과 비교하고 C13이 더해지지만, 둘 다 참이라 실패 목록은 C11 그대로다 |

### 추가 확인 (FIX에서 새로 생긴 C12)

`scenarios/adversarial-other-dept-head.yaml` · D-2004(report_confirmed, 연 환산 보험료 합계 150,000,000원, dept_head: head-01). D-2003으로는 C12를 시험할 수 없다. 8천만 원이라 head-02도 C11에서 먼저 막히기 때문이다. 그래서 기준액을 넘는 딜을 따로 두었다.

| 단계 | 행위자 | 예상 판정 | 이유 |
|---|---|---|---|
| 1 | head-02 | DENY | 상태·역할·C8·C11 통과, C12: head-02 == deal.dept_head(head-01) → 거짓 |
| 2 | head-01 | ALLOW | 1단계가 DENY라 상태는 report_confirmed 그대로다. 상태·역할·C8·C11·C12 모두 통과. FIX가 정상 결재 경로까지 막지 않았는지 확인한다 |

## 3. BREAK 2 (확정 뒤 고쳐진 CEO 보고서)

### BEFORE

| 항목 | 기록 |
|---|---|
| 시나리오 (파일 · 이름) | `scenarios/adversarial-revised-report.yaml` · `proposal-on-revised-unconfirmed-report` |
| 지키려는 안전 속성 | 설계안은 확정된 CEO 보고서 판에만 근거해야 한다. 확정 뒤 고쳐진 판은 다시 확정받아야 한다 |
| 시작 레코드와 상태 | D-2005 · `report_confirmed` (보고서는 v1로 확정된 뒤 v2로 고쳐짐 · 설계안 report_version 2 · 8천만 원) |
| 행위자 / 역할 | gfc-01 / gfc |
| 시도한 Transition | `confirm_proposal` |
| 관련 Constraint | C1 · C8 · C9 |
| 예상 판정 | **ALLOW** |
| 이유 (desk-check) | 상태 report_confirmed = from ✓ · 역할 gfc ✓ · C1 gfc-01 == deal.gfc ✓ · C8 report_version(2) == ceo_report.version(2) ✓ · C9 8천만 ≤ 1억 ✓ → ALLOW |
| 문제 | C8은 이름과 달리 확정한 판이 아닌 현재 판과 비교한다. CEOReport에 확정한 판을 담을 곳이 없었다. 확정되지 않은 판이 제안의 근거가 되어, world-fit 결론의 위험(CEO 보고서에 잘못된 내용 → 잘못된 컨설팅)이 E까지 흘러간다. 같은 World의 C3는 approved_version과 version을 구분하는데 C8에만 구분이 없었다 |

### FIX

- 바꾼 것: CEOReport에 확정한 판 `confirmed_version`을 추가하고, C8이 현재 판(`version`) 대신 확정한 판(`confirmed_version`)과 비교하게 바꿨다. 새 Constraint `C13_report_unchanged_since_confirm`을 `confirm_proposal`과 `approve_proposal_over_limit`의 guard에 붙였다. seed의 CEOReport마다 `confirmed_version`을 넣었다(CR-2001은 아직 확정 전이라 0).
- 변경 전 → 변경 후:

```yaml
# before
entities:
  CEOReport:
    attributes:
      version: int
transitions:
  confirm_proposal:
    guards: [C1_assigned_gfc_only, C8_uses_confirmed_report, C9_premium_limit]
  approve_proposal_over_limit:
    guards: [C8_uses_confirmed_report, C11_over_limit_only, C12_deal_dept_head_only]
constraints:
  C8_uses_confirmed_report:
    expr: deal.proposal.report_version == deal.ceo_report.version

# after
entities:
  CEOReport:
    attributes:
      version: int
      confirmed_version: int       # 담당 GFC가 확정한 판, 0 = 미확정
transitions:
  confirm_proposal:
    guards: [C1_assigned_gfc_only, C8_uses_confirmed_report, C9_premium_limit, C13_report_unchanged_since_confirm]
  approve_proposal_over_limit:
    guards: [C8_uses_confirmed_report, C11_over_limit_only, C12_deal_dept_head_only, C13_report_unchanged_since_confirm]
constraints:
  C8_uses_confirmed_report:
    expr: deal.proposal.report_version == deal.ceo_report.confirmed_version
  C13_report_unchanged_since_confirm:
    expr: deal.ceo_report.version == deal.ceo_report.confirmed_version
    message: CEO 보고서가 확정 뒤 바뀌었다. 다시 확정받아야 한다.
```

- validate: `evidence/validate-after.txt`에 덧붙임. Constraints 13, READY FOR STUDIO IMPORT ✓.
- 풀리지 않은 것: v1의 전이는 값을 쓰지 않으므로 `confirm_report`가 실행돼도 `confirmed_version`이 기록되지 않는다. 이 값(그리고 C3의 `approved_version`)은 어느 전이도 쓰지 않고 seed나 World 밖에서 들어온다. 값의 주인이 World 안에 없다.
  - 사례: happy-path에서 `confirm_report`(7단계)는 `confirmed_version`을 써 넣지 않는다. 8단계의 C8·C13은 seed에 미리 넣은 `confirmed_version: 1` 덕분에 통과한다. 이 World에서 확정한 판은 확정 행위의 결과가 아니라 처음부터 주어진 값이다.

### AFTER

| 항목 | 기록 |
|---|---|
| 같은 시나리오 | `scenarios/adversarial-revised-report.yaml` · `proposal-on-revised-unconfirmed-report` (D-2005, gfc-01, `confirm_proposal`) |
| 새 예상 판정 | **DENY** |
| 이유 | 상태 report_confirmed = from ✓ · 역할 gfc ✓ · C1 gfc-01 == deal.gfc → 참 · **C8: report_version(2) == confirmed_version(1) → 거짓** · C9: 8천만 ≤ 1억 → 참 · **C13: version(2) == confirmed_version(1) → 거짓** · 실패 목록: C8, C13 → DENY. 판정을 바꾸는 규칙은 새 식의 C8이고, C13도 거짓이라 이중으로 막는다. BEFORE에는 C8이 현재 판(2)과 비교해, 확정하지 않은 2판을 근거로 한 설계안이 통과했다. AFTER에는 확정한 판(1)과 비교해 막힌다. 이 레코드에는 C13 하나만으로 판정이 갈리는 경우가 없다(아래 추가 확인) |

- 기존 시나리오 영향: 판정은 모두 그대로다. 식이 바뀐 C8과 새 C13을 반영해 because 다섯 곳을 고쳤다(happy-path 8단계 · stale-report-version · other-dept-head 1·2단계 · head-bypass). adversarial.yaml은 상태에서 막혀 손대지 않았다.

### 추가 확인 (C13 단독)

`scenarios/adversarial-report-changed-after-confirm.yaml` · D-2006(보고서 version 2 · confirmed_version 1 · 설계안 report_version 1) · gfc-01 `confirm_proposal` → 예상 **DENY**. 상태·역할·C1·C8(1 == 1)·C9를 모두 통과하고 마지막 C13(2 == 1 → 거짓) 하나만 막는다. C13은 C8과 겹치는 규칙이 아니다.

## 4. 관찰

BEFORE와 AFTER 모두 validate는 READY였다. 검증기는 문법과 참조가 맞는지만 보고, 정책에 빈틈이 있는지는 보지 않는다. 두 빈틈 모두 desk-check 시나리오로만 드러났다.

## 5. v1로 막지 못한 것 (같은 법인에 딜 두 개)

- desk-check: 같은 법인 CORP-001에 딜 D-1001과 D-1002가 각각 열려 `diagnosis_approved`에 있다고 하자. 두 딜에서 `sys-01`이 `send_expert_request`를 하면 둘 다 상태 ✓ · 역할 ✓ · C3 ✓ · C4 ✓ → ALLOW다. 같은 법인으로 배정 요청이 두 번 발송된다.
- 막지 못하는 이유:
  1. 제약 식은 이 레코드(`deal.*`와 참조)와 행위자(`principal.*`)만 본다. 다른 레코드를 조회하는 문법이 없다(함수는 `remaining`, `no_payment_attempt_with_key` 둘뿐).
  2. v1의 중복 방지 함수 `no_payment_attempt_with_key`가 결제 아닌 효과에서 무엇을 "시도"로 세는지 정해져 있지 않다. 실행 키를 법인 기준으로 만들어 딜을 가로질러 막는 방법은 규칙이 아니라 데이터 입력 약속이다. 그래서 이 World는 이 함수에 기대는 규칙을 두지 않는다(결정 4).
  3. 딜을 여는 전이가 World에 없다. `opened`는 처음 상태일 뿐이라 "법인당 진행 중인 딜은 하나" 같은 생성 조건을 선언할 곳이 없다.
- 위험 등급: C 발송은 되돌릴 수 없는 외부 효과다. 이 한계는 운영 전에 반드시 풀어야 한다.
- 시나리오로 만들지 않은 이유: 시나리오 하나에는 레코드가 하나뿐이라 두 딜 사이의 일을 표현할 수 없다. 이것도 한계의 일부다.
- 억지로 고치면 가짜 FIX가 되므로 World 규칙은 바꾸지 않았다.

## 6. 판정을 한 가지로 정하기 위한 정리

FIX 두 건 뒤에, 판정이 World와 seed만으로 한 가지로 정해지는지 다시 점검했다(결정 4). 판정이 바뀐 시나리오는 없다.

| 무엇을 | 왜 | 바꾼 것 |
|---|---|---|
| because 표기 | guard를 첫 실패에서 멈추는지 끝까지 평가하는지 정해져 있지 않다. "평가되지 않는다"는 설명은 그 전제에 기대고 있었다 | 모든 시나리오에 전이의 guard 전부의 참·거짓과 실패 목록을 적는다. 평가 방식이 어느 쪽이든 판정과 실패 목록이 같다 |
| seed의 빈 참조 | D-2001~D-2006에 `analysis`·`expert_request` 참조가 없었다. 없는 값에 대한 식의 결과는 정해져 있지 않다 | 모든 레코드가 선언된 속성을 빠짐없이 갖게 했다. 딜마다 산출물 레코드 4개, 모두 35개 레코드 |
| C5 `no_payment_attempt_with_key` | Runtime이 결제 아닌 효과에서 무엇을 "시도"로 세는지 모른다. 판정이 해석에 달린 규칙이다 | C5와 `ExpertRequest.idempotency_key`를 지웠다. 같은 딜 안의 재발송은 상태 흐름이 막고, 딜 사이 중복은 5절에 한계로 남겼다 |
| 값의 뜻 | 단위(원), 판 번호의 시작(1)과 0의 뜻, 쉼표 목록, ref가 가리키는 대상이 world에 없었다 | `world.yaml` 머리말에 값과 식의 해석 약속을 두고, 속성마다 뜻과 쓰는 주체를 주석으로 적었다 |
| 쓰이지 않는 선언 | 규칙이 읽지 않는 속성·역할은 뜻을 추측하게 만든다 | `confirmed_by`·`round`·산출물 3종의 `corp`·`support` 역할·`css-01`을 지웠다. `AnalysisReport.version`은 결정 2의 기준(판 번호가 있는 산출물만 Entity) 때문에 남기고 이유를 주석에 적었다 |
| 시나리오 `input` | 서술과 seed 값이 1:1로 대응하지 않았다 | `input` 문장에 seed의 레코드 id와 값을 그대로 적었다 |

- validate: `evidence/validate-after.txt`에 덧붙임. Constraints 12, READY FOR STUDIO IMPORT ✓.

## 7. 리뷰 결과

| | 예측 | 실제 Runtime 결과 |
|---|---|---|
| BREAK 1 BEFORE | ALLOW | |
| BREAK 1 AFTER | DENY (C11) | |
| BREAK 2 BEFORE | ALLOW | |
| BREAK 2 AFTER | DENY (C8, C13) | |
