# 자율 AI 에이전트 설계 및 응용, Spring 2026  
## Programming Assignment 2

**Due:** Thursday, April 30, 23:59 p.m.

## 기본 정보

- **Programming Language:** Python notebook
- **Available Environment:**
  - reasoning from scratch >= 0.1.2
  - sympy >= 1.14.0
  - pandas
- **Additional packages:** numpy, huggingface, pandas, OpenAI
- **LLM Backend:** GPT-4 (recommended), GPT-3.5-turbo, Llama-3-Instruct-8B, Qwen3-0.6B, Qwen3-1.5B, Qwen3-8B

---

# 1. Assignment Overview

## 1.1 Background

이번 과제에서는 Tree of Thoughts (ToT) (Yao et al., https://dl.acm.org/doi/abs/10.5555/3666122.3666639) 를 활용하여 **Game of 24**를 해결하는 LLM-based reasoning system을 구현한다.

Game of 24는 **네 개의 정수를 각각 정확히 한 번씩 사용하여, 사칙연산과 괄호를 통해 최종값 24를 만드는 문제**이다.  
예를 들어, 입력 숫자가 `4, 9, 10, 13`일 때 다음 식은 유효한 정답이다.

```text
(10 − 4) × (13 − 9) = 6 × 4 = 24
```

즉, 본 과제의 목표는 단순히 LLM이 `"24"`라는 숫자를 출력하도록 하는 것이 아니라,  
주어진 숫자 네 개를 정확히 한 번씩 사용한 **arithmetic expression**을 생성하는 것이다.

기본적인 one-shot 또는 Chain-of-Thought (CoT) 접근은 하나의 reasoning path만을 따라간다.  
즉, 주어진 입력 `x`에 대해 모델은 하나의 intermediate reasoning sequence를 생성하고, 그 뒤 최종 답을 출력한다.

```text
x → z1, z2, . . . , zT → y
```

여기서 `zt`는 intermediate reasoning step, `y`는 최종 출력이다.  
이 방식은 간단하지만, 초기에 잘못된 intermediate step이 선택되면 이후 전체 reasoning path가 실패하기 쉽다.

Tree of Thoughts는 이 문제를 완화하기 위해, intermediate reasoning step을 단일 연쇄가 아니라 탐색 가능한 상태 공간(state space)로 해석한다.  
즉, 모델은 특정 시점의 부분 상태 `st`에서 여러 개의 후보 thought를 제안하고, 각 후보의 유망도를 평가한 뒤, 가장 좋은 일부만 유지한다.

```text
st --propose--> {τ(1), τ(2), . . . , τ(k)}
```

각 후보 thought `τ(i)`는 새로운 상태 `s(i)_(t+1)`를 만든다.

```text
st --τ(i)--> s(i)_(t+1)
```

이후 각 상태에 대해 value function 또는 heuristic function을 계산한다.

```text
v(s) ∈ R or v(s) ∈ {sure, maybe, impossible}
```

탐색 과정에서는 이 값을 바탕으로 상위 `b`개의 상태만 유지하는 beam search 또는 breadth-first search (BFS)를 수행할 수 있다.  
이를 형식적으로 쓰면, depth `t`에서의 beam `Bt`는 다음과 같이 갱신된다.

```text
Bt+1 = TopB( ⋃ s∈Bt Expand(s) )
```

여기서 `Expand(s)`는 상태 `s`에서 생성된 후보 다음 상태들의 집합이고, `TopB(·)`는 value score 기준으로 상위 `b`개를 선택하는 연산이다.

이번 과제에서 중요한 점은 thought를 단순한 hidden state가 아니라, 사람이 읽을 수 있는 명시적 intermediate equation으로 다룬다는 것이다.  
Game of 24에서는 이러한 intermediate equation이 현재 숫자 집합을 줄여 나가는 하나의 reasoning step이 된다.

이번 과제는 다음 세 가지 개념을 모두 포함한다.

- **Verifier:** 최종 식이 실제로 유효한지 검사
- **CoT baseline:** 단일 reasoning path baseline 구현
- **ToT solver:** 여러 후보 intermediate thought를 탐색하는 solver 구현

즉, 단순히 한 번 모델을 호출해 답을 받는 것이 아니라,  
**intermediate state representation, candidate expansion, partial-state evaluation, search, and final verification의 전체 파이프라인을 설계하는 것**이 과제의 핵심이다.

## 1.2 Example: ToT flow on [4, 9, 10, 13]

아래는 입력 숫자 `[4, 9, 10, 13]`에 대해 ToT가 어떤 식으로 동작할 수 있는지를 보여주는 예시이다.  
중요한 점은 이것이 단일 정답만을 생성하는 것이 아니라, 여러 intermediate thought를 제안하고 그중 일부를 선택하며 탐색을 진행하는 흐름이라는 점이다.

초기 상태를 다음과 같이 두자.

```text
s0 = (4, 9, 10, 13)
```

이 상태에서 모델은 예를 들어 다음과 같은 후보 thought를 제안할 수 있다.

```text
13 − 9 = 4   (left: 4, 4, 10)
10 − 4 = 6   (left: 4, 6, 9)
13 + 10 = 23 (left: 4, 9, 23)
```

이제 각 후보 상태에 대해 value를 계산하거나 heuristic score를 부여한다.  
예를 들어 첫 번째 후보 `(4, 4, 10)`는 24를 만들기 쉬워 보이므로 `sure` 또는 높은 score를 받을 수 있고,  
세 번째 후보 `(4, 9, 23)`는 덜 유망하므로 `maybe` 또는 낮은 score를 받을 수 있다.

탐색 알고리즘은 이러한 평가를 바탕으로 상위 일부 상태만 유지한다.  
예를 들어 첫 번째 thought를 선택했다면 다음 상태는

```text
s1 = (4, 4, 10)
```

이 된다. 이 상태에서 다시 다음 후보 thought를 제안할 수 있다.

```text
10 − 4 = 6  (left: 4, 6)
10 + 4 = 14 (left: 4, 14)
```

이 중 `(4, 6)` 상태가 더 유망하다고 판단되면 다음 상태는

```text
s2 = (4, 6)
```

이 되고, 마지막으로

```text
4 × 6 = 24 (left: 24)
```

를 통해 목표 상태에 도달한다.

즉, 이 예시는 단순히 `(10 − 4) × (13 − 9)`라는 최종 식 하나를 보여주는 것이 아니라,  
**현재 상태 → 후보 thought 제안 → 상태 평가 → 선택된 다음 상태**의 흐름을 반복하면서 최종 해에 도달하는 ToT의 전형적인 작동 방식을 설명하는 예시이다.

## 1.3 Goal

본 과제의 목표는 다음과 같다.

- LLM을 사용한 Chain-of-Thought baseline을 구현하여 baseline으로 사용하고,
- 동일한 문제에 대해 Tree of Thoughts solver를 구현하여,
- 두 접근의 차이를 실험적으로 비교하는 것이다.

특히, ToT에서는 **intermediate equation을 이용한 state expansion과 candidate pruning이 어떻게 동작하는지**를 명확히 드러내는 것이 중요하다.

## 1.4 Metric

실험과 비교의 기본 metric은 다음과 같다.

- **Primary metric:** success rate on the provided Game of 24 dataset

정답 판정은 기본적으로 최종 arithmetic expression에 대한 verifier 결과를 기준으로 한다.

---

# 2. Requirements

## 2.1 Problem 1. Verifier

Game of 24에서 최종 출력은 arithmetic expression이어야 하며, 다음 조건을 모두 만족해야 한다.

- expression이 syntactically valid할 것
- 주어진 네 숫자를 정확히 한 번씩 사용할 것
- expression의 값이 정확히 24가 될 것

위 세 조건을 판정하는 verifier를 구현해야 한다.  
구현 방식은 자유롭지만, **expression parsing 및 evaluation이 안전하게 이루어져야 한다.**

예를 들어 verifier의 입출력은 다음과 같이 설계할 수 있다.

### Input

```python
expr = "(10 - 4) * (13 - 9)"
numbers = [4, 9, 10, 13]
```

### Output

```python
{
    "valid_syntax": True,
    "used_numbers_ok": True,
    "value_ok": True,
    "value": 24.0,
    "success": True
}
```

반대로, 다음과 같은 잘못된 식은 verifier에서 reject되어야 한다.

### Input

```python
expr = "(10 - 4) * (13 - 8)"
numbers = [4, 8, 10, 13]
```

### Output

```python
{
    "valid_syntax": True,
    "used_numbers_ok": False,
    "value_ok": False,
    "value": 30.0,
    "success": False
}
```

## 2.2 Problem 2. CoT Baseline

LLM에게 한 번만 reasoning을 수행하게 하는 Chain-of-Thought baseline을 구현해야 한다.

기본 흐름은 다음과 같다.

- 입력 숫자에 대해 prompt를 구성
- LLM을 1회 호출
- 최종 arithmetic expression을 추출
- verifier를 사용해 정답 여부를 판정

## 2.3 Problem 3. ToT Solver

Tree of Thoughts solver를 구현해야 한다.

이 solver는 최소한 다음 요소를 포함해야 한다.

- state representation
- candidate thought proposal
- partial-state evaluation or ranking
- search over multiple candidates
- final expression reconstruction and verification

구현 방식은 자유롭다.  
예를 들어 BFS, beam search, heuristic search, 또는 이와 유사한 탐색 전략을 사용할 수 있다.  
다만 단순히 여러 답을 independent하게 샘플링하는 것만으로는 충분하지 않으며,  
**intermediate state를 실제로 유지하고 탐색하는 구조가 있어야 한다.**

## 2.4 Other Constraints

본 과제는 다음 조건을 따른다.

- OpenAI API key는 환경변수 `OPENAI_API_KEY`를 통해 설정한다고 가정.
- Python notebook 제출 시, 주요 출력 결과는 지우지 말 것.
- 제출물은 실행 가능한 상태여야 하며, 외부 dependency를 추가한 경우 명시할 것.

---

# 3. Submission

## 3.1 Submission Format

제출물은 아래를 포함해야 한다.

- `PA2.ipynb`
- `[student number].pdf`

## 3.2 Report Contents

PDF 보고서에는 최소한 다음 내용을 포함해야 한다.

- verifier 설계 방식
- CoT baseline 설계 및 결과
- ToT state representation 및 search strategy
- CoT와 ToT의 비교 결과
- failure case analysis

---

# 4. Final Remark

수업 때 사용했던 Qwen3-0.6B 등의 소형 모델은 본 문제를 풀기에 다소 부적합합니다.  
특히, 10B 이하급의 모델을 사용하는 경우 정교한 prompt engineering을 동반하지 않으면 출력 형식이 consistent하지 않아 parsing과 normalizing이 많이 힘들어질 것입니다.

때문에 본 과제는 **GPT-4, 혹은 그에 준하는 성능을 지닌 model**을 권장합니다.  
단, **자체 reasoning 기능이 있는 모델 (e.g. GPT-o1)은 사용을 금합니다.**

원본 paper를 학습에 참고하시기 바랍니다.  
- ToT paper: https://dl.acm.org/doi/abs/10.5555/3666122.3666639
