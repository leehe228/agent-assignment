# PA1 수행 설계안 (Codex)

## 1) 과제 요구사항 해석 (핵심)
- 목표는 단일 one-shot solver가 아니라, 상태(`state_t`)에 따라 행동(`action_t`)을 선택하는 정책 기반 agent 설계.
- 핵심 생성 백엔드는 `Qwen3-0.6B base`.
- 평가지표는 `accuracy`(primary), `average cost`(secondary).
- 문제당 총 비용은 반드시 `<= 14`.
- `predictions.json`에는 최소 `unique_id`, `prediction`, `cost`, `trace` 포함.
- 제출 알고리즘은 공식 example code와 달라야 함.

## 2) 공식 예제 코드 분석 반영

### 2.1 참고 파일별 역할
- `assignment1/examples/README.md`
  - Chapter 3 메인/솔루션 노트북의 역할을 안내.
  - standalone evaluator(`evaluate_math500.py`, `evaluate_math500_batched.py`) 존재를 명시.
- `assignment1/examples/ch03_main.ipynb`
  - baseline 평가 파이프라인의 기준 구현이 들어 있음.
  - 핵심 함수: `render_prompt` -> `generate_text_stream_concat` -> `extract_final_candidate` -> `grade_answer` -> `evaluate_math500_stream`.
- `assignment1/examples/ch03_exercise-solutions.ipynb`
  - 프롬프트 민감도, 데이터셋 크기 효과, `math_new50_exercise` 활용 아이디어 제공.
  - 특히 base 모델이 프롬프트 템플릿에 민감하다는 관찰(예: 500개 기준 15.3% -> 31.2%)을 제시.

### 2.2 baseline의 정확한 정의 (과제 비교 기준)
- 공식 baseline은 사실상 다음 one-pass 루프:
  1. `render_prompt(problem)`
  2. 단일 generation 호출
  3. `extract_final_candidate`로 답 추출
  4. `grade_answer`로 정오 판정
- 즉, 문제별 행동 선택(재시도, verifier 기반 분기, 후보 선택)이 없는 고정 파이프라인.

### 2.3 과제용 설계 원칙 (example와 차별화)
- 예제의 추출/정규화/채점 유틸은 최대한 재사용 가능.
- 그러나 정책은 반드시 확장:
  - 상태 기반 분기
  - budget-aware action selection
  - 실패 시 repair 또는 self-consistency 호출
  - trace와 cost를 per-example로 명시 저장

## 3) 데이터 확인 결과 및 사용 규칙
- `data/math500_test.json`: 500개, 본 평가.
- `data/math_new50_exercise.json`: 50개, 개발/튜닝.
- 공통 필드: `problem`, `solution`, `answer`, `subject`, `level`, `unique_id`.
- 정답 형식은 정수 비중이 크지만, tuple/식/루트/각도도 빈도가 높음.
- 중요: 추론 입력에는 `problem`만 사용. `solution`, `answer`는 평가/분석에만 사용.

## 4) 제안 정책 (Budget-safe)

### 4.1 상태 정의
- `raw_outputs`: 지금까지의 생성 텍스트들
- `parsed_answers`: 후보별 추출 답
- `format_ok`: `\\boxed{}` 또는 fallback 추출 성공 여부
- `check_score`: symbolic/규칙 기반 점검 결과
- `consistency_score`: 후보 합의도
- `cost_so_far`: 누적 비용

### 4.2 행동 정의 및 비용
- `solve_once` (LLM, +5)
- `symbolic_check` (parser/sympy/rule, +1)
- `repair_once` (LLM, +5)
- `self_consistency_k3` (batched SC, +7)
- `aggregate_vote` (deterministic rerank/vote, +1)
- `finalize` (+0)

### 4.3 정책 흐름
1. `solve_once` (5)
2. `symbolic_check` (6)
3. 분기
   - A: 고신뢰(`format_ok` + checker pass) -> `finalize` (총 6)
   - B: 파싱 실패/모호 -> `repair_once` -> `symbolic_check` -> `aggregate_vote` -> `finalize` (총 13)
   - C: 저신뢰/불안정 -> `self_consistency_k3` -> `symbolic_check` -> `aggregate_vote` -> `finalize` (총 9~10)

상한 검증: 최악 경로도 `<= 13`, 과제 제한 `14` 이내.

## 5) 예제 코드 재사용/확장 포인트

### 5.1 재사용 권장
- `get_last_boxed`, `extract_final_candidate`
- `normalize_text`, `split_into_parts`
- `sympy_parser`, `equality_check`, `grade_answer`

### 5.2 확장 구현
- `policy_step(state)` 추가: 다음 action 선택.
- `should_repair / should_sc` 판단 규칙 추가.
- `budget_guard`: 매 step 실행 전 남은 budget 검사.
- 예제의 `evaluate_math500_stream`를 policy 실행 루프 버전으로 확장:
  - 생성 1회 고정 -> 다단계 action trace 실행.

## 6) 구현 구조 제안

```text
assignment1/
  data/
  examples/
  src/
    io_utils.py
    prompting.py
    parser_checker.py
    policy.py
    evaluator.py
  outputs/
    predictions.json
    metrics.json
  PA1.ipynb
```

- `parser_checker.py`: ch03 함수 재사용 + normalization 보강
- `policy.py`: 상태/행동/비용/분기 핵심
- `evaluator.py`: baseline/proposed 동일 조건 비교
- `PA1.ipynb`: 실험 실행, 표, 사례분석, 보고서용 figure

## 7) 실험 설계 (공식 예제와 정합)

### 7.1 baseline 재현
- `ch03_main.ipynb`의 one-shot 프로토콜 그대로 재현.
- 동일 데이터(`math500_test.json`)에서 정확도 산출.

### 7.2 proposed policy 평가
- 동일 데이터/동일 채점(`grade_answer`)으로 비교.
- 추가 측정:
  - 평균 cost
  - cost 분포(최소/평균/최대)
  - action 빈도(`repair`, `sc` 사용률)

### 7.3 ablation
- `-repair`
- `-self_consistency`
- `-symbolic_check`
- 목적: 어떤 구성 요소가 정확도/비용에 기여했는지 분리.

## 8) 로컬(Mac) -> B200 서버 운영 전략

### 8.1 Mac (개발 단계)
- `math_new50_exercise.json` 중심으로 빠른 디버깅.
- 프롬프트/파서/분기 규칙 안정화.
- `predictions.json` 스키마 및 budget 계산 검증.

### 8.2 B200 (최종 실행)
- 코드 동기화 후 `math500_test.json` 전체 실행.
- 기본은 단일 샘플 루프, 필요 시 배치 추론 모드 추가(정확도 변화 없이 throughput 개선).
- 최종 산출:
  - `outputs/predictions.json`
  - `outputs/metrics.json`
  - notebook 결과 셀(삭제 금지)

### 8.3 재현성
- seed 고정
- 모델/토크나이저/라이브러리 버전 기록
- 실행 설정(config dict/json) 저장

## 9) 제출물 포맷

### 9.1 `predictions.json` 권장 스키마
```json
[
  {
    "unique_id": "gen/algebra/001.json",
    "prediction": "\\boxed{24}",
    "cost": 6,
    "trace": [
      "solve_once",
      "symbolic_check(pass)",
      "finalize"
    ]
  }
]
```

### 9.2 보고서에 반드시 포함
- 정책 설계 근거(왜 그 state/action인가)
- budget allocation 근거(왜 그 경로를 택했는가)
- baseline 대비 개선 결과
- baseline 실패 / proposed 성공 사례(trace 포함)

## 10) 제출 전 체크리스트
- [ ] 모든 example의 `cost <= 14`
- [ ] baseline 재현 결과와 proposed 결과를 같은 evaluator로 비교
- [ ] `predictions.json`에 `unique_id/prediction/cost/trace` 포함
- [ ] notebook 최종 출력 유지
- [ ] 보고서에 사례 분석(최소 3개) 포함
