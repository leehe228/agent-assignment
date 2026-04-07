# PA1 서버 실행 End-to-End Playbook

## 0. 목적
이 문서는 B200 서버에서 Codex 새 thread를 통해 과제의 전 과정을 일관되게 수행하기 위한 실행 표준이다.  
대상 범위는 다음 4가지다.

1. 설계 확정
2. 구현
3. 테스트/실험
4. 보고서 작성 보조

최종적으로 제출 가능한 산출물 `PA1.ipynb`, `predictions.json`, `[student number].pdf`를 만든다.

---

## 1. 고정 입력/제약

### 1.1 고정 입력 파일
- `assignment1/data/math500_test.json`
- `assignment1/data/math_new50_exercise.json`
- 참고 예제 코드
  - `assignment1/examples/ch03_main.ipynb`
  - `assignment1/examples/ch03_exercise-solutions.ipynb`
  - `assignment1/examples/README.md`
- 기존 설계 문서
  - `assignment1/codex/PA1_assignment_for_codex.md`
  - `assignment1/codex/PA1_execution_design.md`

### 1.2 과제 제약
- 생성 백엔드: `Qwen3-0.6B base`
- 예산: 문제당 `cost <= 14`
- 예제 코드와 다른 알고리즘이어야 함 (정책 기반 분기 필요)
- 평가 지표: accuracy, average cost
- `predictions.json`에 최소 `unique_id/prediction/cost/trace`

---

## 2. 권장 작업 디렉터리/브랜치

### 2.1 디렉터리
- repo root: `agent-assignment/`
- 과제 루트: `agent-assignment/assignment1/`

### 2.2 브랜치
- 새 브랜치 생성 권장: `codex/pa1-agentic-policy`

---

## 3. 단계별 실행 절차 (Gate 기반)

## Phase 1: Kickoff & 환경 검증

### 목표
- 서버에서 실행 가능한 최소 환경 확보
- 데이터/참고 파일 존재 확인

### 실행 항목
1. Python/torch/sympy/reasoning-from-scratch 버전 확인
2. GPU 인식 확인 (`cuda` 사용 가능 여부)
3. 데이터셋 로딩 스모크 테스트
4. 출력 디렉터리 생성: `assignment1/outputs/`

### 통과 조건 (Gate 1)
- 패키지 import 성공
- `math500_test` 500개, `math_new50_exercise` 50개 로딩 확인
- 1개 샘플로 prompt->generate->extract->grade 루프 정상 동작

---

## Phase 2: Baseline 재현 (공식 예제 정합)

### 목표
- `ch03_main.ipynb`의 one-shot baseline을 서버에서 재현
- 이후 proposed policy 비교 기준 생성

### 구현 포인트
- baseline 파이프라인 고정:
  1. `render_prompt`
  2. single generation
  3. `extract_final_candidate`
  4. `grade_answer`
- `math_new50_exercise`로 먼저 실행 후, `math500_test` 실행

### 저장 산출물
- `assignment1/outputs/baseline_predictions.json`
- `assignment1/outputs/baseline_metrics.json`

### 통과 조건 (Gate 2)
- baseline accuracy 계산 완료
- 예측 파일에 `unique_id`, `prediction`, `correct` 등 분석용 필드 존재

---

## Phase 3: Agentic Policy 구현

### 목표
- state/action 기반 정책 구현
- budget-safe 보장

### 권장 모듈 구조
- `assignment1/src/io_utils.py`
- `assignment1/src/prompting.py`
- `assignment1/src/parser_checker.py`
- `assignment1/src/policy.py`
- `assignment1/src/evaluator.py`

### 최소 기능 요구
1. 상태 객체: `raw_outputs`, `parsed_answers`, `check_score`, `cost_so_far`, `trace`
2. 행동 집합:
   - `solve_once` (+5)
   - `symbolic_check` (+1)
   - `repair_once` (+5)
   - `self_consistency_k3` (+7)
   - `aggregate_vote` (+1)
3. 정책 분기:
   - 고신뢰 즉시 종료
   - 파싱 실패/모호 시 repair
   - 저신뢰 시 self-consistency
4. `budget_guard`:
   - 액션 실행 전 초과 여부 검사
   - 어떤 경로도 `>14` 금지

### 저장 산출물
- `assignment1/outputs/proposed_predictions.json`
- `assignment1/outputs/proposed_metrics.json`

### 통과 조건 (Gate 3)
- 50개 exercise 전수 실행 완료
- 전 샘플 budget 제약 통과
- trace 기록 정상

---

## Phase 4: 테스트/검증

### 목표
- 정책 로직 신뢰성 확보
- 보고서에 넣을 근거 데이터 생성

### 테스트 세트
1. 단위 테스트
   - boxed 추출
   - normalize
   - tuple/symbolic grade
   - budget 누적/차단
2. 통합 테스트
   - 10개 subset quick run
   - 50개 exercise full run
3. 본 평가
   - 500개 full run

### 반드시 뽑을 지표
- accuracy (baseline vs proposed)
- average cost
- cost 분포 (min/mean/max)
- action 사용률 (`repair`, `sc`)

### 통과 조건 (Gate 4)
- `math500_test` 전수 결과 생성
- 실패 케이스 분석 가능한 로그/trace 확보

---

## Phase 5: 보고서 작성용 산출물 생성

### 목표
- `[student number].pdf`에 바로 넣을 표/사례/해석 준비

### 필수 포함 내용
1. 시스템 구조도/설계 원리
2. budget allocation 전략
3. 실험 결과 표
4. baseline 실패 vs proposed 성공 사례 (3~5개)
5. 한계와 향후 개선

### 권장 부록 데이터
- subject/level별 성능 분해
- action trace 예시
- 평균 토큰 길이/실행 시간

### 통과 조건 (Gate 5)
- 보고서 초안 markdown 또는 notebook 섹션 완성
- 수치/표/사례가 재현 가능 경로로 저장됨

---

## 4. 최종 패키징 체크리스트

- [ ] `PA1.ipynb` 실행 결과 셀 유지
- [ ] `predictions.json` 최종본 존재
- [ ] 모든 샘플 `cost <= 14`
- [ ] baseline 대비 개선 수치 명시
- [ ] 사례 분석 최소 3개
- [ ] 서버 실행 커맨드/환경정보 기록

---

## 5. 리스크 및 대응

### 리스크 1: 서버에서 모델/라이브러리 불일치
- 대응: kickoff에서 버전 프린트 및 실행 스모크 테스트 선행

### 리스크 2: 수식 파싱 실패 다발
- 대응: normalize 규칙 강화 + fallback 로직 + trace에 실패 사유 기록

### 리스크 3: budget 초과
- 대응: 액션 전 `budget_guard` 강제, 초과 시 즉시 finalize

### 리스크 4: 결과는 좋지만 설명 부족
- 대응: 실험 중간부터 ablation/사례 수집 자동화

---

## 6. 권장 실행 순서 요약 (한 줄 플로우)

환경검증 -> baseline 재현 -> 정책 구현 -> 50개 디버깅 -> 500개 본실행 -> 분석/표/사례 생성 -> notebook/JSON/pdf 패키징

