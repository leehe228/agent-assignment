# PA1용 Codex Master Instruction (서버 새 Thread용)

아래 내용을 서버의 새 Codex thread에 그대로 붙여 넣어 사용한다.

---

## Prompt to Codex

너는 과제 구현 담당 엔지니어다.  
목표는 `assignment1` 폴더에서 Programming Assignment 1을 끝까지 완성하는 것이다.

### 1) 작업 목표
- 설계 검토/보완
- 코드 구현
- 테스트 및 실험
- 제출물 생성 보조

최종 산출물:
1. `assignment1/PA1.ipynb`
2. `assignment1/predictions.json`
3. 보고서 작성에 바로 쓸 수 있는 실험 결과 정리 파일(표/사례/핵심 분석)

### 2) 입력/참고 파일
- 과제 요약: `assignment1/codex/PA1_assignment_for_codex.md`
- 설계 문서: `assignment1/codex/PA1_execution_design.md`
- 실행 플레이북: `assignment1/codex/PA1_server_end_to_end_playbook.md`
- 공식 예제:
  - `assignment1/examples/ch03_main.ipynb`
  - `assignment1/examples/ch03_exercise-solutions.ipynb`
  - `assignment1/examples/README.md`
- 데이터:
  - `assignment1/data/math500_test.json`
  - `assignment1/data/math_new50_exercise.json`

### 3) 절대 제약
- 생성 백엔드는 `Qwen3-0.6B base`를 사용
- 문제당 총 budget은 반드시 `<= 14`
- example code와 다른 알고리즘이어야 함
  - 단순 one-shot 고정 파이프라인 금지
  - state/action 정책 분기 필수
- `predictions.json`에 최소 필드 포함:
  - `unique_id`, `prediction`, `cost`, `trace`

### 4) 구현 요구사항

#### 4.1 Baseline 재현
- `ch03_main.ipynb`의 baseline 흐름을 동일 evaluator 기준으로 재현:
  - prompt -> single generation -> extract -> grade
- 결과 저장:
  - `assignment1/outputs/baseline_predictions.json`
  - `assignment1/outputs/baseline_metrics.json`

#### 4.2 Proposed policy 구현
- 권장 액션:
  - `solve_once` (+5)
  - `symbolic_check` (+1)
  - `repair_once` (+5)
  - `self_consistency_k3` (+7)
  - `aggregate_vote` (+1)
- 상태 기반 분기 구현:
  - high confidence면 즉시 finalize
  - parsing/format 불량이면 repair
  - low confidence면 self-consistency
- `budget_guard`를 넣어 어떤 경우도 14 초과 금지

#### 4.3 코드 구조
- 최소 모듈:
  - `assignment1/src/parser_checker.py`
  - `assignment1/src/policy.py`
  - `assignment1/src/evaluator.py`
  - 필요 시 `io_utils.py`, `prompting.py`

### 5) 테스트/실험 요구사항

#### 5.1 테스트
- 단위 테스트:
  - answer extraction
  - normalization
  - symbolic equality
  - budget 계산
- 통합 테스트:
  - `math_new50_exercise` 전체 실행
  - `math500_test` 전체 실행

#### 5.2 실험 지표
- baseline vs proposed 정확도
- average cost
- cost 분포
- action 사용률

#### 5.3 필수 산출 파일
- `assignment1/outputs/proposed_predictions.json`
- `assignment1/outputs/proposed_metrics.json`
- 최종 제출용 `assignment1/predictions.json` (필수 포맷 준수)

### 6) 보고서 작성 보조 산출
- 보고서에 바로 넣을 수 있는 내용 생성:
  1. 시스템 설계 요약
  2. budget allocation 설명
  3. 결과 표 (baseline vs proposed)
  4. baseline 실패 / proposed 성공 사례 3~5개 (trace 포함)
  5. 한계와 개선점

가능하면 `assignment1/outputs/report_assets.md`로 저장.

### 7) 진행 방식
- 바로 구현 시작하고, 필요하면 합리적 가정을 먼저 두고 진행
- 막히는 지점만 질문
- 각 단계 종료 시 아래를 반드시 보고:
  1. 무엇을 변경했는지
  2. 어떤 명령으로 검증했는지
  3. 남은 작업이 무엇인지

### 8) 완료 기준 (Definition of Done)
- `math500_test` 전체에 대해 예측 생성 완료
- 모든 샘플 budget `<= 14`
- `predictions.json` 필수 필드 충족
- baseline 대비 proposed 비교 결과 제출 가능 형태로 정리
- notebook 출력 셀 유지

### 9) 최종 응답 형식
최종에는 아래 5개 섹션으로 정리:
1. 변경 파일 목록
2. 구현 요약
3. 실험 결과 요약
4. 제약 준수 체크 결과 (budget/포맷)
5. 제출 직전 체크리스트

---

## 사용 팁
- 위 Prompt 전체를 한 번에 붙여 넣은 뒤 시작하면 된다.
- 첫 응답에서 Codex가 작업 계획과 즉시 실행 항목을 제시해야 정상이다.

