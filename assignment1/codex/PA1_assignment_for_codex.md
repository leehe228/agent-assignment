# 자율 AI 에이전트 설계 및 응용, Spring 2026, Programming Assignment 1

**Due:** Tuesday, April 7, 23:59 p.m.

## Basic Information

- **Programming Language:** Python notebook
- **Available Environment:**
  - reasoning from scratch >= 0.1.2
  - sympy >= 1.14.0
  - torch >= 2.7.1
  - tokenizers >= 0.21.2
- **Additional packages:** numpy, huggingface

---

## 1. Assignment Overview

### 1.1 Problem Formulation

본 과제에서는 수학 문제를 입력으로 받아 정답을 산출하는 **agentic reasoning system**을 설계한다.  
단순히 한 번의 generation으로 답을 내는 것이 아니라, 현재까지의 **intermediate result**, **verifier feedback**, **heuristic score**, **symbolic parsing 결과** 등을 바탕으로 다음 행동을 결정하는 **policy**를 설계하는 것이다.

즉, 본 과제에서 학생이 설계해야 하는 것은 단일 solver가 아니라, 상태에 따라 행동을 선택하는 정책 **π**이다.

\[
\pi : state_t \mapsto action_t
\]

여기서 `action_t`는 예를 들어 다음을 포함할 수 있다.

- 새로운 답안 생성
- 동일 문제에 대한 재시도
- final answer reformatting
- symbolic checker 실행
- rule-based parser 실행
- learned router / controller 실행
- candidate selection / reranking
- 종료 후 최종 답 제출

### 1.2 Goal

본 과제의 목표는 다음과 같다.

- **Qwen3-0.6B base**를 핵심 생성 백엔드로 사용하여
- 제한된 **budget** 안에서
- one-shot baseline보다 더 나은 reasoning behavior를 보이는
- **agentic system**을 설계하는 것

구현 방식은 자유롭다. 예를 들어 **self-consistency**, **verifier-guided repair**, **symbolic checking**, **reranking**, **adaptive prompting** 등을 활용할 수 있다.  
다만 중요한 것은 특정 기법의 사용 여부가 아니라, 그러한 구성 요소들이 실제로 **상태에 따라 행동을 달리 선택하는 policy 안에서 작동하는가**이다.

### 1.3 Metric

실험과 비교의 기본 관점은 다음과 같다.

- **Primary metric:** accuracy
- **Secondary consideration:** average cost

---

## 2. Requirements

### 2.1 Baseline

공식 baseline은 다음과 같다.

- **Model:** Qwen3-0.6B
- **Protocol:** one-shot generation
- **Reference:** Chapter 3 example code and evaluator

### 2.2 Dataset

Reasoning from scratch github  
(https://github.com/rasbt/reasoning-from-scratch.git) 를 참고할 것.

- `math500 test.json`: 기본 accuracy 측정에 사용
- `math new50 exercise.json`: 필요 시 in-context learning, prompt design, debugging 등에 활용 가능

### 2.3 Budget

본 과제에서는 문제 하나당 총 계산 비용에 다음 upper bound를 둔다.

\[
\sum_t C(a_t) \le 14
\]

즉, 문제 하나에 대해 어떤 행동을 조합하더라도 **총 비용은 반드시 14 이하**여야 한다.

각 행동의 비용은 다음과 같이 정의한다.

- **LLM generation call:** \( C_{\text{llm}} = 5 \)  
  (e.g. initial solve, self-critique generation, reformatted final answer request, second attempt with hint)

- **Symbolic verification / parser / rule-based checker:** \( C_{\text{sym}} = 1 \)  
  (e.g. sympy equivalence check, handcrafted parser, regex-based normalization, rule-based router)

- **Lightweight neural inference:** \( C_{\text{nn-infer}} = 2 \)  
  (e.g. learned router MLP, confidence predictor, reranker, verifier head)

- **Aggregation / voting / deterministic reranking:** \( C_{\text{agg}} = 1 \)

- **Batched self-consistency with \(k\) samples:**  
  \[
  C_{\text{sc}} = 5 + \lceil \log_2 k \rceil
  \]

### 2.4 Other Constraints

본 과제의 시스템은 다음 조건을 만족해야 한다.

- 각 example에 대해 budget을 계산할 것.
- `predictions.json`에 최종 답과 함께 trace를 저장할 것.
- Example code와 다른 알고리즘으로 설계할 것.
- Python notebook 제출 시, 최종 출력 결과는 지우지 말 것.

---

## 3. Submission

### 3.1 Submission Format

제출물은 아래를 포함해야 한다.

- `PA1.ipynb`
- `predictions.json`
- `[student number].pdf`

`predictions.json`은 다음과 같은 형식을 권장한다.

```json
[
  {
    "unique_id": "...",
    "prediction": "\\boxed{...}",
    "cost": 11,
    "trace": ["solve", "verify", "repair", "finalize"]
  }
]
```

### 3.2 Report Contents

PDF 보고서에는 최소한 다음 내용을 포함해야 한다.

- 시스템 구조 설계 원리
- budget 사용 방식
- 실험 결과 및 분석
- Baseline에서 실패하고 proposed system에서 성공하는 사례 분석

---

## 4. Final Remark

본 과제의 핵심은 본 케이스에 대해 무작정 정확도를 높이는 것이 아닌, **타당한 reasoning policy를 설계하는 것**입니다.  
즉, 직·병렬적 검증 프로세스 나열, champion prompting 등의 trick을 막기 위해 연산량의 upper bound를 **budget**으로 추상화하였습니다.

따라서 좋은 제출물은 단순히 accuracy가 높은 제출물이 아니라,

- 왜 그런 state / action 구조를 두었는지,
- 왜 그런 budget allocation을 했는지,
- 그리고 그 설계가 실제로 어떤 장단점을 보였는지

를 일관되게 설명할 수 있는 제출물임을 학습에 참고하시기 바랍니다.

---

## Codex 입력용 메모

아래는 Codex에게 그대로 전달하기 쉬운 형태의 핵심 요구사항 요약이다.

### 해야 할 일
- Python notebook 기반으로 수학 문제를 푸는 **agentic reasoning policy**를 설계
- 단일 one-shot solver가 아니라 상태에 따라 action을 선택하는 정책 설계
- Qwen3-0.6B base를 핵심 생성 백엔드로 사용
- one-shot baseline보다 더 나은 reasoning behavior를 목표로 할 것
- accuracy를 높이되 average cost도 함께 고려

### 사용할 수 있는 구성 요소 예시
- 새로운 답안 생성
- 재시도
- final answer reformatting
- symbolic checker
- rule-based parser
- learned router / controller
- candidate selection / reranking
- verifier-guided repair
- self-consistency
- adaptive prompting

### 필수 제약
- 문제당 총 budget은 반드시 **14 이하**
- 각 example마다 실제 사용 budget 계산 필요
- `predictions.json`에 final answer와 trace 저장 필요
- example code와 다른 알고리즘이어야 함
- notebook 제출 시 최종 출력 결과 삭제 금지

### 비용 정의
- LLM generation: 5
- symbolic verification / parser / rule-based checker: 1
- lightweight neural inference: 2
- aggregation / voting / deterministic reranking: 1
- batched self-consistency with k samples: \(5 + \lceil \log_2 k \rceil\)

### 데이터셋
- `math500 test.json`: 기본 accuracy 측정
- `math new50 exercise.json`: ICL, prompt design, debugging 등에 활용 가능

### 공식 baseline
- Model: Qwen3-0.6B
- Protocol: one-shot generation
- Reference: Chapter 3 example code and evaluator

### 제출물
- `PA1.ipynb`
- `predictions.json`
- `[student number].pdf`

### 보고서에 포함할 것
- 시스템 구조 설계 원리
- budget 사용 방식
- 실험 결과 및 분석
- baseline 실패 / proposed system 성공 사례 분석
