---
layout: post
title: "Adversarial Feedback Loop 설계: 최근 연구가 말하는 것"
date: 2026-07-06 10:00:00 +0900
description: >
  멀티 에이전트 시스템에서 Critic/Verifier가 Generator에게 피드백을
  어떻게 제공해야 하는가 — Self-Refine, Reflexion, VerifiAgent,
  JudgeFlow, SPC, AutoGen AgentEval, Agent-as-a-Judge 7개 시스템을
  통해 확인된 설계 원칙들. 언어 비평의 한계, tool-grounded 검증의 우위,
  block-level 진단, 중간 개입 타이밍, RL 자기 대전까지.
tags: [agents, multi-agent, feedback, critic, harness, research-survey]
series: Agent Team Architecture
chapter: 8
toc:
  sidebar: left
---

[지금까지의 시리즈]({% link _explorations/2026-06-06-cctt-workflows.md %})는
agent 팀을 *어떻게 구성하는가*에 집중했다. 이 포스트는 한 걸음 뒤로
물러서서 묻는다: **Critic(또는 Verifier, Judge)이 Generator에게 피드백을
줄 때 그 피드백은 어떻게 생겨야 하는가?**

이 질문은 생각보다 열려 있다. "비판적으로 검토하라"는 프롬프트를 쓰면
될 것 같지만, 최근 연구들은 피드백의 *형식*, *소스*, *시점*, *범위*
각각이 루프의 효과를 크게 좌우한다는 것을 보여준다.

아래는 2023~2025년에 peer-review를 통과한 7개 시스템에서 추출한
설계 원칙들이다. 각 원칙이 어떤 시스템에서 검증되었는지, 그 한계는
무엇인지를 함께 기록한다.

---

## 1. 피드백 형식 — "어디가 문제"와 "어떻게 고쳐"를 동시에

**Self-Refine** (Madaan et al., NeurIPS 2023,
[arXiv:2303.17651](https://arxiv.org/abs/2303.17651))은 단일 LLM이
Generator와 Critic을 모두 수행하는 가장 단순한 설정을 연구했다.
추가 학습 없이, 피드백을 두 컴포넌트로 강제 구조화했을 때 7개
벤치마크에서 +5~40%의 개선이 나타났다:

> **(i) Localization** — 무엇이, 어디서 문제인지  
> **(ii) Instruction** — 어떻게 수정해야 하는지

이 구조가 없으면 "좋지 않습니다"류의 평가로 끝난다. 구조가 있으면
Generator가 *어디서부터 손을 대야 하는지*를 안다.

Self-Refine의 루프 구조:

```
GENERATE → FEEDBACK(localization + instruction)
         → REFINE
         → FEEDBACK(localization + instruction)
         → REFINE → ...
```

모든 이전 output과 feedback이 컨텍스트에 누적된 채로 다음 iteration에
들어간다. 히스토리 전체가 컨텍스트에 남는다는 점이 단순 retry와 다르다.

**설계 시사점:** Critic의 출력 형식을 프리폼으로 두지 말 것. "어디가
틀렸는가"와 "무엇으로 대체해야 하는가"를 구분해서 요청하는 것만으로도
Generator의 수정 품질이 달라진다.

---

## 2. 중요한 경고 — 순수 언어 비평은 추론 task에서 효과 없음

Self-Refine의 결과를 읽기 전에 먼저 알아야 할 실험이 있다.

**Huang et al. 2024,
*"LLMs Cannot Self-Correct Reasoning Yet"*** 은 순수하게 LLM이 자기
자신을 비평하는 설정(oracle 피드백 없음)에서 수학·논리 추론 task의
성능이 개선되지 않거나 오히려 하락한다는 것을 실험적으로 확인했다.

Self-Refine의 개선이 나타난 영역(코드 스타일, 대화 품질, 감성적
글쓰기)과 개선이 나타나지 않는 영역(수학적 추론, 논리 퍼즐)을 나누는
기준은 **외부 oracle의 유무**다.

```
순수 언어 비평 루프:
  수학·논리 추론  → 효과 없거나 역효과
  코드 스타일     → 효과 있음
  대화·서술 품질  → 효과 있음
```

**설계 시사점:** 어떤 task인지에 따라 피드백 소스를 달리해야 한다.
LLM 의견만으로 닫히는 루프는 추론 정확도를 올리지 못한다. 이 한계를
해결하는 것이 다음 원칙들의 출발점이다.

---

## 3. 피드백 소스 — Tool-Grounded 검증이 LLM 비평보다 강하다

**Reflexion** (Shinn et al., NeurIPS 2023,
[arXiv:2303.11366](https://arxiv.org/abs/2303.11366))은 세 컴포넌트를
분리했다:

- **Actor** — 행동 또는 코드를 생성
- **Evaluator** — 결과를 평가
- **Self-Reflection** — 실패 시 언어 비평을 생성, 누적 저장

코딩 task에서 Evaluator가 LLM 판단이 아니라 **실제 코드 실행**
(pass/fail + 구조화된 test 결과 텍스트)으로 루프를 닫는 것이 핵심이다.
LLM이 "이 코드가 잘못된 것 같다"고 말하는 게 아니라, 인터프리터가
실제로 오류를 던진다.

이 방향을 더 밀고 나간 것이 **VerifiAgent** (EMNLP 2025 Findings,
[arXiv:2504.00406](https://arxiv.org/pdf/2504.00406))의 2계층 Verifier다:

| 계층 | 방식 | task별 도구 |
|---|---|---|
| **1층: Meta-verification** | LLM 타당성 체크 | (공통) |
| **2층: Tool-based verification** | 외부 도구 자율 선택 | 수학 → Python interpreter |
| | | 논리 → Z3 symbolic solver |
| | | 상식 → 검색 엔진 |

VerifiAgent의 ablation 결론이 명확하다: **도구 계층 제거 > LLM 계층
제거 성능 하락.** LLM 비평보다 도구 실행 결과가 피드백의 더 큰
contributor다.

피드백의 내용도 구조화되어 있다:
- (i) 두 계층 검증에서 도출한 *오류 이유*
- (ii) 도구 실행 관찰을 반영한 *수정 방향*

**설계 시사점:** Task가 수학/코드/논리라면 Critic의 언어 비평에만
의존하지 말 것. task type에 맞는 실행 도구를 자동으로 선택하는
verifier 층을 두는 것이 검증된 패턴이다. 그 도구를 LLM 비평 위에
쌓는 것이 아니라, 도구가 primary이고 LLM이 secondary다.

---

## 4. 피드백 범위 — 전체 리뷰가 아니라 Block-Level 책임 진단

**JudgeFlow** ([arXiv:2601.07477](https://arxiv.org/pdf/2601.07477))는
워크플로우 최적화 문제에서 피드백의 *범위*를 재정의했다.

기존 방식: 전체 출력이 좋다/나쁘다 (end-to-end 평가).

JudgeFlow의 방식:

```
실패한 워크플로우 실행 trace
  → Judge: 각 블록에 rank-based 책임 점수 부여
  → Optimizer: 가장 책임 높은 블록 하나만 수정
  → 재실행 → Judge → ...
```

Optimizer가 한 번에 수정하는 범위를 **단일 블록으로 제한**하는 것이
설계의 핵심이다. MATH +3.1점(5.6%), MBPP +1.5점(1.8%).

이 설계의 논리: 여러 블록을 동시에 수정하면 어떤 변경이 성능을
바꿨는지 인과 관계를 알 수 없다. 가장 책임 있는 블록 하나를 고치고
다시 측정하면 신호가 깨끗하다.

**Reflexion이 유사한 원칙을 메모리 설계에서도 보여준다.** 네 가지
메모리 전략을 ablation했다:

| 전략 | 내용 |
|---|---|
| `NONE` | 반성 없음 |
| `LAST_ATTEMPT` | 직전 시도의 trace만 |
| `REFLEXION` | 누적된 언어 비평만 |
| `LAST_ATTEMPT_AND_REFLEXION` | trace + 누적 비평 (최고 성능) |

단순히 "기억을 많이" 넣는 것이 아니라, 어떤 종류의 기억을 넣는지가
성능을 결정한다.

**설계 시사점:** 피드백이 "전체적으로 개선이 필요합니다"로 끝나면
Generator는 어디를 건드려야 할지 모른다. 책임 범위를 좁히고 수정
대상을 하나로 지목하는 것이 빠른 수렴을 만든다.

---

## 5. 평가 기준 자체를 검증하라 — MetaEvaluation

**Microsoft AutoGen AgentEval** (EMNLP 2024,
[arXiv:2405.02178](https://arxiv.org/abs/2405.02178))은 평가 루프에
세 번째 역할을 추가했다:

| 에이전트 | 역할 |
|---|---|
| **CriticAgent** | 이 task에 관련된 평가 기준 생성 |
| **QuantifierAgent** | 기준별 output 점수화 |
| **VerifierAgent** | 기준 집합의 *변별력* 테스트 |

VerifierAgent의 역할이 새롭다: 기준을 적용해서 나온 점수가
실제로 성능 차이를 변별하는지를 테스트한다. 변별력 없는 기준은
모든 출력이 비슷한 점수를 받아서 순위를 매길 수 없게 된다.

이 문제는 rubric 기반 평가 시스템에서 실제로 흔히 발생한다:
기준은 잘 작성되어 있지만 모든 출력이 "4/5"를 받는 상황.
VerifierAgent는 그런 기준을 잡아내서 제거하거나 조정한다.

수학 문제 12,500개 + ALFWorld 시뮬레이션으로 검증됨.

**설계 시사점:** 평가 기준(rubric)을 한번 만들고 끝내지 말 것.
기준 자체가 실제로 좋은 출력과 나쁜 출력을 변별하는지를 별도
에이전트로 검사하는 것이 숨겨진 false positive를 방지한다.

---

## 6. 피드백 타이밍 — 완료 후가 아니라 실행 중

기존 패턴: task 완료 → 최종 출력 평가 → 피드백.

**Agent-as-a-Judge 패러다임** (arXiv:2508.02994, 2025)이 제시하는
방향은 다르다: Judge 에이전트에 tool use + multi-step reasoning +
메모리를 부여해서 task 실행 *도중* 중간 단계에 개입한다.

기존 LLM-as-judge의 비판:
- 최종 출력만 보기 때문에 *어떤 추론 과정*이 그 출력을 낳았는지를 평가하지 못함
- tool 사용 결정, 중간 스텝 품질, 검색 전략 같은 것들이 블랙박스로 남음

Agent-as-a-Judge는 이 블랙박스를 열어서 중간 단계에 피드백을 넣는다.
최종 결과 판단이 아니라 *과정 감독*이다.

이 패러다임은 아직 형성 중이지만 방향은 명확하다: 피드백의 시점이
뒤로 밀릴수록 고칠 수 있는 범위가 줄어든다.

**설계 시사점:** Critic이 task 완료 후에만 개입한다면 이미 만들어진
결과에 대한 사후 판단이다. 실행 중 중간 지점에 개입하면 비용은 높아지지만
수정 범위가 넓어진다. task 특성에 따라 개입 지점을 선택해야 한다.

---

## 7. RL 자기 대전 — 피드백 루프를 학습 신호로

앞의 1~6은 모두 inference-time 패턴이다. 같은 문제를 training-time에서
접근하는 방법이 **SPC (Self-Play Critic)** (NeurIPS 2025,
[arXiv:2504.19162](https://arxiv.org/abs/2504.19162))이다.

같은 base model의 두 복사본을 게임으로 대전시킨다:

- **Sneaky Generator:** 탐지하기 *어렵도록* 의도적으로 잘못된 추론
  스텝을 생성하는 쪽
- **Critic:** 어느 스텝이 틀렸는지 탐지하는 쪽

게임 outcome (win/loss)이 보상. 인간의 step-level 주석 불필요.

반복 학습으로 ProcessBench에서 70.8% → **77.7%** 단계적 향상. 결과
critic은 DeepSeek-R1 distilled보다 강해졌다.

이 설계가 흥미로운 이유: Generator와 Critic이 서로의 약점을 찾아내도록
*설계된* 긴장 관계에 있다. Critic이 강해지면 Generator는 더 교묘한
오류를 만들어야 하고, 그게 다시 Critic을 훈련시킨다. 자기 강화 루프다.

**설계 시사점:** Inference-time 루프만으로는 Critic 능력 자체의 한계를
넘을 수 없다. SPC는 Critic이 *더 좋은 Critic*이 되도록 훈련하는 다른
레이어의 문제다. 이 두 레이어(inference vs. training)를 구분해서 생각해야 한다.

---

## 정리 — 설계 선택지의 지형도

7개 시스템을 4개 축으로 정리하면:

### 축 1: 피드백 소스

```
LLM 의견 ←————————————————————→ 실행 결과
Self-Refine   Reflexion(QA)   VerifiAgent   Reflexion(코드)
(언어 비평)   (혼합)          (도구 우선)   (실행 100%)
```

**추론/코드/수학 task일수록 오른쪽으로.** 순수 언어 비평(왼쪽)은
Huang et al. 2024에 의해 추론 task에서 효과 없음이 실험 확인됨.

### 축 2: 피드백 범위

```
전체 출력 평가 ←——————————→ 단일 블록 책임 진단
AutoGen       Self-Refine    JudgeFlow
(기준별 점수)  (전체 localization) (block-level)
```

**수정이 복잡할수록 범위를 좁혀야** 인과 관계가 보인다.

### 축 3: 피드백 타이밍

```
실행 중 개입 ←——————————→ 완료 후 평가
Agent-as-a-Judge          Self-Refine, Reflexion
(중간 단계)               (최종 출력)
```

**비용 대 수정 가능 범위 트레이드오프.** 개입이 빠를수록 변경 가능한
게 많지만 inference 비용이 높아진다.

### 축 4: 평가 기준의 메타 검증

```
기준 고정 ←——————————→ 기준 자체를 검증
CCTT Ch.1~7   AutoGen AgentEval
(루브릭 진화)  (VerifierAgent: 변별력 테스트)
```

**좋은 Critic은 좋은 기준을 가져야 한다.** 기준이 변별력 없으면
루프가 닫혀도 신호가 없다.

---

## 한 줄 결론

피드백 루프를 설계할 때 가장 먼저 물어야 할 질문은 **"Critic이 무엇으로
판단하는가"**다. LLM 의견인지, 실행 결과인지, 기준의 변별력은 검증되었는지.
이 소스가 결정되고 나서야 형식(localization + instruction), 범위(block vs.
whole), 타이밍(중간 vs. 완료 후)이 의미를 갖는다.

---

## 참고 문헌

- Madaan, A. *et al.* (2023). *Self-Refine: Iterative Refinement
  with Self-Feedback.* NeurIPS 2023.
  [arXiv:2303.17651](https://arxiv.org/abs/2303.17651).
- Shinn, N. *et al.* (2023). *Reflexion: Language Agents with
  Verbal Reinforcement Learning.* NeurIPS 2023.
  [arXiv:2303.11366](https://arxiv.org/abs/2303.11366).
- Huang, J. *et al.* (2024). *Large Language Models Cannot
  Self-Correct Reasoning Yet.*
  [arXiv:2310.01848](https://arxiv.org/abs/2310.01848).
- VerifiAgent. *EMNLP 2025 Findings.*
  [arXiv:2504.00406](https://arxiv.org/pdf/2504.00406).
- JudgeFlow. [arXiv:2601.07477](https://arxiv.org/pdf/2601.07477).
- Chen, L. *et al.* (2024). *AgentEval: A Multi-Agent Framework
  for Evaluating LLM-based AI Agent Systems.* EMNLP 2024.
  [arXiv:2405.02178](https://arxiv.org/abs/2405.02178).
- Self-Play Critic (SPC). NeurIPS 2025.
  [arXiv:2504.19162](https://arxiv.org/abs/2504.19162).
- *When AIs Judge AIs: Agent-as-a-Judge Survey.*
  [arXiv:2508.02994](https://arxiv.org/html/2508.02994v1).
- 이 시리즈의 이전 글들:
  [Ch. 1 — 에이전트 하네스 개념과 레퍼런스]({% link _explorations/2026-05-06-agent-harness-team.md %}) ·
  [Ch. 7 — CCTT Workflows]({% link _explorations/2026-06-06-cctt-workflows.md %}).
