# PA2 구현 및 실험 설계 문서

## 0. 문서 목적

이 문서는 `assignment2/PA2_assignment_description.md` 요구사항을 기준으로, 이후 `PA2.ipynb` 구현 및 실험/보고서 작성에 바로 사용할 수 있도록 설계를 고정하기 위한 문서다.  
범위는 다음 4가지다.

- Problem 1: Verifier 구현
- Problem 2: CoT baseline 구현
- Problem 3: ToT solver 구현
- 실험 설계 및 보고서 연결 구조

### 0.1 `PA2.ipynb` 기준 실행 계약(Contract)

공식 제공 notebook(`assignment2/PA2.ipynb`)을 기준으로, 아래 함수/출력 계약을 반드시 맞춘다.

- Prompt template 함수:
  - `make_cot_prompt(numbers)`
  - `make_propose_prompt(state)`
  - `make_value_prompt(state)`
- Verifier 함수:
  - `verify_24_expression(expr, numbers, target=24)`
- CoT 실행 함수:
  - `run_cot_baseline(dataset=DATASET_24)`
  - 반환 `DataFrame` 최소 컬럼: `id`, `numbers`, `raw_response`, `extracted_expr`, `success`
- ToT 실행 함수:
  - `run_tot_solver(dataset=DATASET_24, beam_width=5)`
  - 반환 `DataFrame` 최소 컬럼: `id`, `numbers`, `tot_expr`, `success`
- 최종 비교 셀:
  - `baseline_df`와 `tot_df`를 merge한 비교 `DataFrame` 생성

또한 notebook 기본 설정을 따른다.

- `OPENAI_MODEL = "gpt-4"` (필요 시 변경 가능)
- `SEED = 7`, `random`/`numpy` seed 고정
- `call_openai(..., temperature=0.0)`를 기본 재현 설정으로 사용

---

## 1. 과제 요구사항 재정의 (구현 관점)

### 1.1 필수 구현 항목

- Verifier는 아래 3조건을 모두 판정해야 함
  - 문법적으로 유효한 expression인지
  - 입력 숫자 4개를 정확히 한 번씩 사용했는지
  - 최종 값이 정확히 24인지
- CoT baseline은 입력당 **LLM 1회 호출**의 단일 reasoning path
- ToT solver는 반드시 아래 구조 포함
  - state representation
  - candidate thought proposal
  - partial-state evaluation/ranking
  - multi-candidate search
  - final expression reconstruction + verification

### 1.2 평가 지표

- Primary metric: dataset success rate (verifier success 기준)
- Secondary 분석 지표(설계 문서에서 추가):
  - syntax validity rate
  - number-usage pass rate
  - average API calls / latency
  - ToT depth별 생존 상태 수

### 1.3 모델/환경 제약

- 기본 실행 환경 가정:
  - `reasoning from scratch >= 0.1.2`
  - `sympy >= 1.14.0`
  - `pandas`
- 추가 패키지 후보: `numpy`, `huggingface`, `OpenAI`
- API key는 환경변수 `OPENAI_API_KEY`로 로드
- 모델은 과제 가이드의 허용 백엔드 범위 내에서 선택
  - GPT-4 (권장), GPT-3.5-turbo, Llama-3-Instruct-8B, Qwen3-0.6B/1.5B/8B
- 자체 reasoning 기능이 강조된 모델(예: GPT-o1 계열)은 사용하지 않음

### 1.4 제출/실행 제약

- 필수 제출물은 `PA2.ipynb`, `[student number].pdf`
- notebook 제출 시 주요 출력 결과는 지우지 않음
- 제출물은 실행 가능한 상태여야 함
- 외부 dependency를 추가했다면 notebook/report에 명시

---

## 2. 전체 시스템 구조

```text
Input numbers
  -> Solver (CoT or ToT)
  -> Expression extraction/normalization
  -> Verifier
  -> Per-instance log + aggregate metrics
```

### 2.1 권장 산출물 구조

```text
assignment2/
  PA2.ipynb
  PA2_assignment_description.md
  PA2_implementation_experiment_design.md
  outputs/
    run_config.json
    cot_instances.jsonl
    tot_instances.jsonl
    summary_metrics.json
```

주의:

- `outputs/*`는 실험 재현성과 보고서 작성 편의를 위한 내부 산출물이다.
- 실제 제출 필수 조건은 1.4의 두 파일(`PA2.ipynb`, `[student number].pdf`)이다.
- notebook 실행 자체는 `baseline_df`, `tot_df`, 최종 비교 `DataFrame`이 정상 생성되어야 완료로 본다.

---

## 3. 구현 설계

### 3.1 백엔드 추상화 (권장)

- 모델 호출 계층을 `generate_expression(prompt, model_name, temperature, max_tokens)` 형태로 추상화
- 동일 인터페이스로 CoT/ToT가 백엔드만 바꿔 실행되게 구성
- 단, notebook 제공 helper `call_openai(...)` 시그니처와 동작은 유지하는 것을 기본으로 함
- 비교 실험 시에는 solver 외 변수를 고정
  - 같은 verifier
  - 같은 dataset
  - 가능한 동일 decoding 설정

### 3.2 공통 유틸/데이터 구조

```python
from dataclasses import dataclass
from fractions import Fraction

@dataclass
class VerifyResult:
    valid_syntax: bool
    used_numbers_ok: bool
    value_ok: bool
    value: float | None
    success: bool
    error: str | None = None
```

```python
@dataclass(frozen=True)
class Item:
    value: Fraction
    expr: str

@dataclass
class NodeState:
    items: tuple[Item, ...]
    steps: tuple[str, ...] = ()
    score: float = 0.0
    depth: int = 0
```

핵심 원칙:

- 값 계산은 `Fraction` 기반으로 수행 (부동소수점 오차 회피)
- expression은 Unicode 연산자(`×`, `÷`, `−`)를 ASCII(`*`, `/`, `-`)로 normalize
- 랜덤성은 `seed` 고정 (`random`, `numpy`, sampling temperature)
- notebook scaffold의 `Item`, `NodeState`를 그대로 쓰거나 확장하되, `run_tot_solver(...)` 계약은 유지

### 3.3 Problem 1: Verifier 설계

### A. 함수 시그니처

```python
def verify_24_expression(expr: str, numbers: list[int], target: int = 24) -> dict:
    ...
```

### B. 처리 단계

1. **Normalization**
   - 연산자 문자 통일, 공백 정리
2. **Safe syntax check**
   - `ast.parse(..., mode="eval")` 사용
   - 허용 노드 whitelist 적용 (`BinOp`, `Add`, `Sub`, `Mult`, `Div`, `UnaryOp`, `Constant` 등)
   - `Name`, `Call`, `Pow`, `FloorDiv`, `Mod` 등 금지
3. **숫자 사용 규칙 검사**
   - AST에서 literal 숫자 추출
   - 입력 `numbers`와 멀티셋이 정확히 일치하는지 검사
4. **안전 평가**
   - AST 재귀 평가로 `Fraction` 계산
   - division by zero 등 예외 처리
5. **정답 판정**
   - `value == Fraction(24, 1)`이면 `value_ok=True`

### C. 반환 포맷

과제 설명서 예시와 동일한 key를 유지:

```python
{
    "valid_syntax": True/False,
    "used_numbers_ok": True/False,
    "value_ok": True/False,
    "value": 24.0 or None,
    "success": True/False,
    "error": None or "..."
}
```

### D. Verifier 단위 테스트 세트(최소)

- 정상 정답 식
- 숫자 중복 사용
- 숫자 누락
- 문법 오류
- division by zero
- 24 근접값(예: 23.9999) 방지 케이스

### 3.4 Problem 2: CoT Baseline 설계

### A. 동작 정의

- 입력 숫자 1개 문제당 LLM 호출 1회
- 출력에서 최종 arithmetic expression 1개 추출
- verifier로 성공/실패 판정
- 함수 시그니처는 notebook과 동일하게 `run_cot_baseline(dataset=DATASET_24)` 사용

### B. 프롬프트 포맷 (권장)

```text
Solve the Game of 24.
Use each number exactly once and only +, -, *, /, parentheses.
Numbers: {a}, {b}, {c}, {d}

You may think step-by-step internally, but return exactly one line:
EXPR: <arithmetic expression>
```

### C. 출력 파싱 규칙

1. `EXPR:` 라인 정규식 우선 추출
2. 실패 시 마지막 줄에서 수식 패턴 fallback
3. normalize 후 verifier 호출

### D. 기록 항목

- 필수 `DataFrame` 컬럼: `id`, `numbers`, `raw_response`, `extracted_expr`, `success`
- 권장 추가 컬럼: `verify_result`, `latency_ms`, `token_usage`
- notebook 최소 요구 컬럼(`raw_response`, `extracted_expr`)과 이름을 맞추기 위해
  - 내부 변수명은 자유
  - 최종 `DataFrame` 컬럼명은 `raw_response`, `extracted_expr`로 통일

### 3.5 Problem 3: ToT Solver 설계

### A. 상태(State) 정의

- `items`: 현재 남은 숫자/부분식 집합 (`Item` 튜플)
- `steps`: 사람이 읽을 수 있는 intermediate equations 로그
- `depth`: 현재 깊이 (초기 0, 목표 최대 3)
- `score`: pruning용 평가 점수

### B. Candidate Thought Proposal

LLM에게 현재 상태를 주고 `k`개 thought 제안:

```text
State numbers: 4, 9, 10, 13
Propose 5 valid next steps.
Each line format:
THOUGHT: <expr_i> <op> <expr_j> = <new_value>
```

검증 규칙:

- state 내 두 항만 사용했는지
- 연산 결과가 실제 계산과 일치하는지
- 파싱 불가 thought는 폐기

Fallback:

- 유효 thought가 부족하면 결정론적 연산 확장(모든 쌍 × 연산자)으로 보강

### C. 상태 전이(Transition)

- 선택된 two-item 연산으로 새 `Item` 생성
- 나머지 item과 합쳐 다음 state 구성
- 새 thought를 `steps`에 append
- canonical signature로 중복 상태 제거

### D. Partial-state Evaluation / Ranking

기본 점수(heuristic):

- `h1`: 목표 24와의 거리(남은 item이 적을수록 가중치 증가)
- `h2`: 연산 안정성(0 나눗셈 유도 패턴 패널티)
- `h3`: 깊이 보정(더 짧은 경로 선호)

옵션 점수(LLM value):

- `sure / maybe / impossible` 라벨 프롬프트
- 매핑 예: `sure=+2`, `maybe=0`, `impossible=-2`

최종 점수:

```text
score = h1 + h2 + h3 + h_llm(optional)
```

### E. 탐색 알고리즘

- 기본: Beam Search
- 권장 기본값:
  - `beam_width b = 5` (notebook scaffold 기본값과 동일)
  - `proposals_per_state k = 5`
  - `max_depth = 3`

의사코드:

```text
beam <- [initial_state]
for depth in 0..2:
  expanded <- []
  for s in beam:
    cands <- propose_and_validate(s, k)
    for c in cands:
      s_next <- transition(s, c)
      s_next.score <- evaluate(s_next)
      expanded.append(s_next)
  beam <- top_b(unique(expanded), b)
  if any terminal success in beam:
    return best_success_expr
return best_candidate_expr_or_fail
```

요건 체크:

- 단순 독립 샘플링(`N`개의 최종식 생성 후 best 선택)만으로는 ToT로 간주하지 않는다.
- 반드시 `state -> thought -> next state` 전이를 유지한 탐색 로그를 저장한다.
- 함수 시그니처는 notebook과 동일하게 `run_tot_solver(dataset=DATASET_24, beam_width=5)` 사용
- 최종 결과 `DataFrame`은 최소 `id`, `numbers`, `tot_expr`, `success` 컬럼 포함

Terminal 성공 조건:

- `len(items) == 1`
- `items[0].value == 24`
- 최종 expr를 verifier로 재검증해 `success=True`

### 3.6 Notebook 구현 섹션 구성안 (`PA2.ipynb`)

1. Prompt template TODO 3개 구현 (`make_cot_prompt`, `make_propose_prompt`, `make_value_prompt`)
2. `verify_24_expression` 구현
3. `run_cot_baseline` 구현 및 `baseline_df` 생성
4. `Item`, `NodeState` 정의 유지/확장 후 `run_tot_solver` 구현
5. `tot_df` 생성
6. Final comparison 셀에서 `baseline_df`/`tot_df` merge
7. 보고서용 결과 표/실패 사례 출력
8. (선택) 재현성용 보조 파일 저장 (`outputs/*.jsonl`)

---

## 4. 실험 설계

### 4.1 데이터 분할/운용

- 기본 실험셋은 notebook의 `DATASET_24`(10개 인스턴스)
- 모든 인스턴스는 solvable이라고 명시되어 있으므로 success rate 비교에 적합
- 소규모 데이터셋 특성상:
  - 최종 지표는 전체 10개에 대해 CoT/ToT 동일 조건으로 측정
  - 하이퍼파라미터 탐색은 과도하게 하지 않고, 변경 사항을 보고서에 명시

### 4.2 메인 비교 실험

- **Exp-Main:** CoT vs ToT
  - 동일 모델 백엔드 사용 (권장 GPT-4급)
  - 동일 verifier/동일 데이터셋에서 success rate 비교
  - 산출: `success_rate_cot`, `success_rate_tot`, `delta`

### 4.3 Ablation 실험

- **Exp-A1 (Search width):** `b in {1,2,3,5}`
- **Exp-A2 (Proposal count):** `k in {3,5,8}`
- **Exp-A3 (Value function):**
  - heuristic only
  - heuristic + LLM value
  - random ranking (control)

보고 항목:

- 성공률 변화
- 평균 호출 수/지연시간 변화
- pruning으로 제거된 유효 경로 비율

### 4.4 추가 비교(선택)

- 모델 비교: GPT-4 vs 경량 모델
- 목적: 과제 remark의 “소형 모델 한계”를 정량적으로 확인

### 4.5 통계/신뢰구간

- success rate에 대해 bootstrap 95% CI 계산
- CoT vs ToT 차이는 paired 관점(동일 문제 집합)으로 해석

---

## 5. 로깅/결과 포맷 설계

우선순위:

- 1순위: notebook 요구 형식의 `DataFrame`(`baseline_df`, `tot_df`, comparison_df`)
- 2순위: 재현성 강화를 위한 JSONL/JSON 보조 로그(선택)

`tot_instances.jsonl` 예시:

```json
{
  "id": "sample_001",
  "numbers": [4, 9, 10, 13],
  "solver": "tot",
  "final_expr": "(10-4)*(13-9)",
  "verify": {
    "valid_syntax": true,
    "used_numbers_ok": true,
    "value_ok": true,
    "value": 24.0,
    "success": true
  },
  "trace": [
    "13-9=4",
    "10-4=6",
    "4*6=24"
  ],
  "depth_reached": 3,
  "api_calls": 4,
  "latency_ms": 1830
}
```

요약 파일 `summary_metrics.json`:

- `n_total`
- `cot_success_rate`
- `tot_success_rate`
- `tot_minus_cot`
- `avg_latency_cot`, `avg_latency_tot`
- `avg_api_calls_cot`, `avg_api_calls_tot`

---

## 6. Failure Case Analysis 프로토콜 (보고서 직결)

실패 케이스를 아래 taxonomy로 분류:

- F1: expression extraction failure
- F2: syntax invalid
- F3: wrong number usage
- F4: arithmetic value != 24
- F5: search/pruning failure (정답 경로가 중간에 소실)

보고서에는 최소 포함:

- CoT 실패 -> ToT 성공 사례 3개
- ToT 실패 사례 3개 + 실패 원인(F1~F5) + 개선안

---

## 7. 리스크 및 대응

- 출력 형식 불안정:
  - 강한 출력 포맷 제약 + regex fallback + normalize
- ToT 탐색 폭주:
  - `b`, `k`, `max_depth` 하드 제한 + 중복 상태 제거
- verifier 오판 위험:
  - `Fraction` 기반 exact eval + 금지 노드 whitelist

---

## 8. 구현 완료 기준 (Definition of Done)

- [x] `PA2.ipynb`의 TODO 셀(프롬프트/Verifier/CoT/ToT/Final comparison) 모두 실행 가능
- [x] verifier가 3조건(문법/숫자사용/값=24)을 분리 리포트
- [x] CoT baseline 1회 호출 프로토콜 구현
- [x] ToT가 명시적 state 탐색(확장/평가/가지치기) 수행
- [x] 동일 verifier로 CoT/ToT 비교 결과 산출
- [x] CoT `DataFrame` 최소 컬럼(`id`, `numbers`, `raw_response`, `extracted_expr`, `success`) 충족
- [x] ToT `DataFrame` 최소 컬럼(`id`, `numbers`, `tot_expr`, `success`) 충족
- [ ] failure case 분석 표본 준비 (현재 실행에서는 CoT/ToT 모두 10/10 성공으로 추가 실패 표본 수집 필요)
- [ ] 보고서 필수 항목(설계/결과/비교/실패분석)과 결과가 1:1 매핑됨 (PDF 작성 단계에서 마무리)
- [x] notebook 주요 출력 셀 유지, 실행 가능성 점검, 추가 dependency 명시 완료

### 8.1 실행 스냅샷 (2026-04-19)

- 실행 환경: `conda env agent` (Python 3.11)
- 의존성 점검: `numpy`, `pandas`, `sympy`, `openai`, `ipykernel`, `huggingface_hub` import 성공
- 노트북 실행: `jupyter nbconvert --execute assignment2/PA2.ipynb` 성공
- 실험 결과(기본 `DATASET_24`, 10개):
  - CoT success rate: `10/10 = 1.000`
  - ToT success rate: `10/10 = 1.000`

---

## 9. 보고서 섹션 매핑 가이드

- `verifier 설계 방식` -> 본 문서 3.3 + 7
- `CoT baseline 설계 및 결과` -> 3.4 + 4.2
- `ToT state representation 및 search strategy` -> 3.5
- `CoT와 ToT 비교 결과` -> 4.2, 4.3, 5
- `failure case analysis` -> 6

이 매핑을 기준으로 `PA2.ipynb` 결과를 캡처하면 PDF 보고서 골격이 바로 완성된다.
