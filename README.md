# 숙제 요약: gfc-corporate-deal

생명보험사 법인영업 컨설턴트(GFC)의 법인 딜 하나가 업무 A→E를 지나는 과정을 Superwork World(`superwork.world/v1`)로 모델링했다. 모든 법인·담당자·금액은 예시 값이다.

- mission: 법인 한 곳의 딜을 초회 미팅 준비부터 제안 설계안의 시스템 등록까지, 업무 사이 인계를 빠뜨리지 않고 진행한다.
- 검증: `npm run validate` → Entity 5 · State 10 · Transition 11 · Constraint 12 · Warnings 0 · READY FOR STUDIO IMPORT ✓

## 왜 딜 전체를 World로 만들었나

원래 하네스는 업무 A\~E를 각각 독립된 프로젝트(업무별 명세·장부)로 만들고, 허브가 장부를 읽어 딜 단위로 묶어 보여 준다. 업무 안의 순서는 커널이 막지만, 업무 사이의 순서는 건을 여는 사람과 적재 프로그램에 맡겨져 있고 딜의 단계는 기록되지 않는 계산값이다. 이 World는 그 빈 곳을 다룬다. 딜을 주인공 상태기계로 두어 업무 사이 순서를 State 흐름으로 강제하고, 업무를 넘나드는 규칙(확정된 보고서 판을 근거로 설계했는가 등)을 Constraint로 선언했다. 대가로 업무 안의 세부 통제(자기 채점 금지, 주장 라벨, 검사한 판 묶기)는 이 World에서 다루지 않는다.

## ESTC

**Entity:** 주인공 `Deal`(상태기계)과 상태 없는 산출물 4개. 산출물은 판 번호를 담아 판 대조 규칙을 식으로 쓰기 위해 둔다(결정 2).

| Entity | 업무 | 규칙이 읽는 값 |
|---|---|---|
| `Deal` | 딜 | corp, gfc, dept_head, meeting_held, opinions_expected/received |
| `AnalysisReport` | A 기업분석보고서 | corp |
| `ExpertRequest` | C 1차 진단·배정안·배정 요청 | version, approved_version, experts |
| `CEOReport` | D CEO 종합보고서 | version, confirmed_version, conflicts, resolution |
| `ProposalDesign` | E 제안 설계안 | report_version, premium_total(연 환산 보험료 합계, 원) |

**State와 Transition:**

```
opened ─draft_analysis(agent)→ analysis_drafted ─confirm_prep(gfc)→ prep_confirmed          [A]
       ─receive_followup(gfc)→ followup_received                                             [B]
       ─approve_diagnosis(gfc)→ diagnosis_approved ─send_expert_request(system, 외부·비가역)→ experts_requested   [C]
       ─draft_report(agent)→ report_drafted ─confirm_report(gfc)→ report_confirmed            [D]
                              report_drafted ─return_report(gfc)→ experts_requested (반려)
       ─confirm_proposal(gfc, 기준액 이하) 또는 approve_proposal_over_limit(dept_head, 기준액 초과)→ proposal_confirmed
       ─register_proposal(system, 외부·가역)→ registered (terminal)                            [E]
```

**Constraint 12개:**

| ID | 붙는 전이 | 규칙 |
|---|---|---|
| C1 | A·B·C·D 확정, 반려, E 확정 | 그 딜의 담당 GFC만 확정·승인·반려한다 |
| C2 | receive_followup | 초회 미팅 기록 없이 후속 단계를 열 수 없다 |
| C3 | send_expert_request | 승인한 판과 지금 배정안의 판이 같아야 발송한다 |
| C4 | send_expert_request | 배정 전문가가 비어 있으면 발송할 수 없다 |
| C6 | draft_report | 받아야 할 전문가 의견이 다 와야 보고서를 쓴다 |
| C7 | confirm_report | 의견 충돌이 있으면 조정 근거가 있어야 확정한다 |
| C8 | E 확정 두 경로 | 설계안은 확정된 CEO 보고서 판을 근거로 해야 한다 |
| C9 | confirm_proposal | 기준액(예시 1억 원) 초과면 부서장 결재로 안내한다 |
| C10 | confirm_prep | 분석 보고서의 법인이 딜의 법인과 같아야 한다 |
| C11 | approve_proposal_over_limit | 부서장 경로는 기준액 초과 건에만 열린다 |
| C12 | approve_proposal_over_limit | 그 딜의 부서장만 결재한다 |
| C13 | E 확정 두 경로 | 확정 뒤 CEO 보고서가 바뀌었으면 다시 확정받아야 한다 |

## 시나리오와 예측

판정은 실행 엔진 없이 `world.yaml`·`seed.yaml`로 손으로 판정(desk-check)한 예측이다. because에는 guard 전부의 참·거짓과 실패 목록을 적어, guard 평가 방식과 관계없이 판정이 하나로 정해지게 했다.

| 파일 | 레코드 | 시도 | 예측 |
|---|---|---|---|
| `scenarios/happy-path.yaml` | D-1001 | opened → registered 9단계 | 모두 ALLOW |
| `scenarios/adversarial.yaml` | D-2001 | 보고서 확정 전에 설계안 확정 | DENY (상태) |
| `scenarios/adversarial-stale-report-version.yaml` | D-2002 | 확정본이 아닌 판을 근거로 한 설계안 확정 | DENY (C8) |
| `scenarios/adversarial-head-bypass.yaml` | D-2003 | 기준액 이하 설계안을 부서장 경로로 확정 | DENY (C11) |
| `scenarios/adversarial-other-dept-head.yaml` | D-2004 | 다른 부서장 결재 / 그 딜의 부서장 결재 | DENY (C12) / ALLOW |
| `scenarios/adversarial-revised-report.yaml` | D-2005 | 확정 뒤 고쳐진 판을 근거로 한 설계안 확정 | DENY (C8, C13) |
| `scenarios/adversarial-report-changed-after-confirm.yaml` | D-2006 | 확정 뒤 보고서가 바뀐 상태에서 설계안 확정 | DENY (C13) |

## BREAK → FIX

| | BREAK | BEFORE | FIX | AFTER |
|---|---|---|---|---|
| 1 | 기준액 이하 설계안을 부서장 경로로 확정해 담당 GFC 확정(C1)과 기준액 검사(C9)를 건너뜀 | ALLOW | C11 기준액 초과만, C12 그 딜의 부서장만 | DENY (C11) |
| 2 | C8이 확정한 판이 아닌 현재 판과 비교해, 확정 뒤 고쳐진 보고서를 근거로 한 설계안이 통과 | ALLOW | `confirmed_version` 추가, C8 식 변경, C13 추가 | DENY (C8, C13) |

두 FIX 모두 수정 전과 후의 validate가 똑같이 READY였다. 빈틈은 desk-check로만 드러났다. 증거: `evidence/validate-before.txt`, `evidence/validate-after.txt`.

## 알려진 한계

- 같은 법인에 딜이 두 개 열리면 각 딜이 배정 요청을 한 번씩, 모두 두 번 발송한다. v1의 식은 다른 레코드를 조회할 수 없고 딜을 여는 전이도 없어서 규칙으로 막지 못한다. 되돌릴 수 없는 외부 효과라 운영 전에 풀어야 한다.
- v1의 전이는 값을 쓰지 않는다. `confirmed_version`·`approved_version`은 확정 행위의 결과가 아니라 seed에서 들어오는 값이다(값의 주인).
- 기준액 초과 설계안은 담당 GFC 확정 없이 부서장 혼자 확정한다(2단 결재 아님).

## 파일 안내

| 파일 | 내용 |
|---|---|
| `world.yaml` | World 정의. 머리말에 값과 식의 해석 약속, 속성마다 뜻·단위·쓰는 주체 |
| `seed.yaml` | 딜 7개와 산출물 28개, 모두 35개 레코드 (선언 속성 완비) |
| `assignment/world-fit.md` | World Model이 필요한 이유 (7문항과 결론) |
| `assignment/entity-decisions.md` | 후보 개념 25개의 분류와 이유, 업무 범위 그림 |
| `assignment/modeling-decisions.md` | 애매한 결정 4개 (딜 World, 산출물 Entity, 승인 = Transition, 해석에 기대는 규칙 제거) |
| `assignment/break-fix.md` | BREAK 2건의 BEFORE·FIX·AFTER, 한계, 리뷰 결과표 |
| `assignment/reflection.md` | 회고 5문항 |

---

이 저장소는 [swit001/superwork-starter](https://github.com/swit001/superwork-starter)(Bootcamp Homework Edition)에서 시작했다. 스타터의 템플릿·문서·검증 도구는 원저작자의 것이며 [`LICENSE`](LICENSE)와 [`NOTICE`](NOTICE)를 따른다.
