---
layout: post
title: "Recipe for Research Team Management with Claude — 스스로 자라는 문헌 위키"
date: 2026-08-07 11:00:00 +0900
description: >
  연구 그룹이 매주 반복하는 문헌 작업 — 논문·세미나 수집, 읽기, 위키
  관리, 강의자료·발표자료·리포트 생성 — 을 Claude Code Routine이
  대신하도록 만든 오픈 레포 소개. 결정론적 파이프라인과 사람의 판단을
  work queue 하나로 분리하고, data/를 유일한 source of truth로 두고,
  언급 빈도가 임계값을 넘으면 위키 노트가 자동 승격되는 self-extending
  wiki, 그리고 모든 설계 결정을 commit note로 남기는 규율까지.
tags: [agents, claude-code, research-workflow, wiki, automation]
series: Agent Team Architecture
chapter: 10
toc:
  sidebar: left
---

[Chapter 5]({% link _explorations/2026-05-28-cctt-research-team-repo.md %})는
CCTT — 논문 *쓰기*를 위한 에이전트 팀 — 를 소개했다. 이번 챕터는 같은
축 위에 있지만 방향이 다른 레포다:
[`yjkim-stat/Recipe-for-Research-Team-Management-with-Claude`](https://github.com/yjkim-stat/Recipe-for-Research-Team-Management-with-Claude)는
논문을 *쓰는* 대신 그룹이 매주 반복하는 문헌 작업 — 새 논문 확인,
읽기, 정리, 신입 온보딩 자료 준비 — 을 대신한다.

README의 첫 문장이 이 레포의 목적을 가장 정확하게 요약한다:

> *The literature work a research group repeats every week, run by
> Claude instead of by hand.*

---

## 1. 무엇을 하는가

토픽(추적할 주제)을 YAML 파일 하나로 정의하면, 매일:

1. arXiv, 주요 학회 인덱스, 추적 중인 YouTube 채널에서 새 논문·발표를
   수집하고
2. 관련성 기준을 넘는 것들을 스코어링·중복제거하고
3. 아직 읽지 않은 항목마다 "읽기 작업"을 큐에 쌓고
4. (사람 또는 Claude 세션이) 그 작업을 처리하면
5. archive, 자기 확장형 위키, 강의자료·슬라이드·리포트를 전부
   재생성한다.

사람에게 남는 일은 **판단이 필요한 부분뿐**이다 — 그룹이 무엇을
추적할지 결정하는 것, 그리고 시스템이 내린 결론을 정정하는 것.
새로 들어온 팀원이 물어볼 법한 것들 — "요즘 뭐가 나왔지", "이 용어가
뭐지", "어디부터 읽어야 하지" — 은 이미 다 쓰여 있고 최신 상태다.

field-agnostic 하다는 점도 중요하다. 토픽은 YAML 파일 하나고, 그
파일이 수집 대상을 전부 결정하므로 같은 레포가 ML 연구실에도, 통계
연구실에도, 다른 분야의 리딩 그룹에도 그대로 쓰인다.

---

## 2. 핵심 아키텍처 — Work Queue에서 갈라지는 두 세계

이 레포를 이해하는 데 가장 중요한 그림은 이거다:

```
                    deterministic                    judgement
             ┌──────────────────────────┐   ┌──────────────────────┐

arXiv ─┐
venues ─┼─► collect ─► score ─► dedupe ─► store ─► queue ─► read ─► complete
YouTube ┘                                   │                          │
                                            ▼                          ▼
                                          data/                  data/queue/done/
                                            │                          │
                                            └──────► render ◄──────────┘
                                                       │
                                        archive/ · wiki/ · outputs/
```

Work queue의 **왼쪽은 전부 재현 가능**하다 — 같은 입력이면 같은
레코드가 나온다. **오른쪽은 사람(또는 모델)의 읽기가 필요**하다. 이
둘을 파일 기반 큐로 명확히 분리해 놓은 덕분에, 읽기 단계가 느리거나
실패해도 잃는 건 그날의 요약뿐이고 수집한 데이터는 안전하다. 반대로
렌더링 방식을 바꾸는 것도 절대 재수집을 요구하지 않는다.

이 분리를 지탱하는 설계 원칙 하나가 README에 명시되어 있다:

> **No model is called from inside the pipeline.** Collection files a
> task describing what needs reading; something else answers it. By
> default that is the daily Claude Code session, which is why the
> system runs with no API key at all.

파이프라인 코드는 순수 Python이고 결정론적이다. "읽는다"는 행위 자체는
파이프라인 밖, 큐를 통해서만 시스템에 들어온다. 기본 백엔드가
"Claude Code 세션"이라는 것 — 즉 API 키 없이 매일 열리는 Claude
Code Routine이 큐를 비운다는 것 — 이 이 레포 이름에 "with Claude"가
붙은 이유다.

---

## 3. 세 가지 디렉토리, 세 가지 권한

CCTT의 `intra/`·`extra/` 분리와 같은 발상이지만, 이 레포는 그걸 한
단계 더 명시적으로 만들었다 — 디렉토리마다 "내가 여기 써도 되는가"를
표로 못박았다.

| 종류 | 어디 | 규칙 |
|---|---|---|
| **당신 것** | `config/`, `templates/`, `inbox/`, 위키의 수동 영역 | 자유롭게 편집. 아무것도 덮어쓰지 않는다 |
| **진실** | `data/` | 파이프라인만 쓴다. 나머지 전부가 여기서 파생된다 |
| **파생물** | `archive/`, `wiki/`, `outputs/` | 지우고 render를 돌리면 동일하게 복원된다. 손으로 고치지 말 것 |

`CLAUDE.md`는 이 표를 아예 "may I write here" 질문에 대한 답으로
재구성해 놓았다 — 에이전트가 세션을 시작할 때 필요한 건 "어디를
봐야 하지"가 아니라 "여기 써도 되나"라는 판단이기 때문이다
(commit note 0008이 이 재구성 자체를 다룬다).

가장 중요한 규칙은 하나로 요약된다: **`data/`가 유일한 source of
truth다.** `archive/`, `wiki/`, `outputs/`를 전부 지우고
`python3 -m pipelines.render`를 실행하면 완전히 동일하게 복원된다.
템플릿을 바꾸거나 렌더러를 고치는 일이 절대로 재수집을 요구하지
않는 이유가 이거다.

---

## 4. Self-Extending Wiki — 언급이 쌓이면 노트가 태어난다

가장 흥미로운 메커니즘은 위키가 스스로 자란다는 점이다. 모든 요약은
자신이 의존하는 개념·방법론·데이터셋의 이름을 명시한다. 그 이름들은
`data/concepts/`에 증거로 쌓이고, **독립적인 출처에서 충분히 (기본값
2회) 언급되면** 자동으로 정식 노트로 승격되고 정의 작성 작업이 큐에
올라간다. 즉 문헌이 어떤 용어에 수렴하기 시작하는 순간, 아무도
요청하지 않아도 위키가 그 용어의 제대로 된 노트를 얻는다.

노트 자체는 절반은 생성되고 절반은 사람 몫이다:

```markdown
# Instrumental Variable
<!-- auto:begin -->
... rebuilt on every render: definition, sources, backlinks ...
<!-- auto:end -->

## Notes
Anything here is never touched.
```

`<!-- auto:end -->` 이후 영역은 영원히 보존된다 — 세미나에서 나온
반박, 우리 데이터에서만 통하는 트릭 같은, 그룹 고유의 읽기가 들어갈
자리다.

더 인상적인 부분은 **staleness(낡음)를 명시적으로 보고**한다는
점이다. 큐가 비어 있다고 해서 모든 게 최신이라는 뜻은 아니다 —
render는 이런 걸 보고한다:

> `definition for 'X' was written against 3 source(s); there are now 9`

3개 출처를 근거로 쓰인 정의가 지금은 9개 중 3분의 1만 반영하고
있다는 뜻이고, 이건 아예 정의가 없는 것보다 나쁘다 — 겉보기엔 완결된
것처럼 보이기 때문이다. 그런데도 **자동으로 다시 쓰지는 않는다.**
정의를 다시 쓰려면 근거를 다시 읽어야 하고, 카운터가 산술만으로
사람이 쓴 글을 폐기해서는 안 되기 때문이다 — 재작성은 여전히 사람의
판단을 거친다.

---

## 5. 만들어내는 것들

| 경로 | 내용 |
|---|---|
| `archive/papers/<year>/<id>/summary.md` | 논문 한 편당 한 페이지: 문제, 기여, 방법, 결과, 한계 |
| `archive/seminars/<id>/` | 타임스탬프가 달린 챕터별 발표 요약 + 전체 transcript |
| `archive/daily/<date>.md` | 그날 들어온 것들, 토픽별로 그룹핑 |
| `wiki/` | 개념·방법론·데이터셋 노트 + backlink 그래프 |
| `outputs/lecture-notes/<slug>/` | 교육 자료: 지형도, 핵심 아이디어, 읽기 순서, 열린 문제 |
| `outputs/slides/<slug>/index.html` | 화살표 키·오버뷰·인쇄 지원되는 단일 파일 덱 |
| `outputs/reports/<slug>/index.html` | 목차·활동 뷰·전체 인덱스가 담긴 단일 파일 리포트 |

슬라이드와 리포트가 **외부 요청이 전혀 없는 단일 HTML 파일**이라는
점이 실용적으로 크다 — 디스크에서 바로 열리고, 첨부파일로도, 네트워크
없는 방에서 USB로도 그대로 작동한다. 그룹 미팅이나 강의에 바로 쓸 수
있는 형태로 나온다는 뜻이다.

---

## 6. 스케줄링 — 왜 cron이 아니라 Claude Code Routine인가

의도된 트리거는 일반 cron이 아니라 **Claude Code Routine**이다.
이유는 명확하다: 2단계(읽기)는 모델이 루프 안에 있어야 하고, 평범한
cron job은 그걸 할 방법이 없다. 등록은 이 프롬프트 하나로 끝난다:

> *Run the daily routine in CLAUDE.md: collect, drain the
> summarization queue, render, and commit.*

여기서 이 레포의 상태 관리 철학이 다시 드러난다: **컨테이너는
매번 휘발성이다.** Routine이 울릴 때마다 완전히 새로운 clone에서
시작한다. 그래서 살아남아야 하는 상태는 전부 `data/`에 있어야 하고,
런이 끝날 때 반드시 커밋되어야 한다 — `data/index/seen.sqlite`가
바이너리인데도 커밋 대상인 이유가 이거다: "clone을 거치고도 살아남지
않는 중복제거 상태는 중복제거 상태가 아니다."

---

## 7. Commit Note — 히스토리 자체를 문서로 설계하기

이 레포에서 가장 독특한 규율은 코드가 아니라 **git 히스토리를 다루는
방식**이다. `CLAUDE.md`에 박힌 규칙:

> **No change to the system lands without a commit note.** ... This
> repository is deployed into other projects and its history is read
> as documentation.

`.claude/skills/commit-notes/` 스킬이 이걸 강제한다 — 커밋마다
`docs/commit/NNNN-slug.md`를 같은 커밋에 스테이징해서 남긴다. 지금까지
27개의 노트(0000~0026)가 쌓여 있고, 각 노트는 고정된 5개 섹션을
따른다: 무엇이 바뀌었는지, 왜 이렇게 만들었는지, 어떤 대안을
버렸는지, 리뷰어가 뭘 확인해야 하는지, 다운스트림에 뭐가 영향을
받는지.

스킬 파일 자체의 한 문장이 이 규율의 존재 이유를 정확히 짚는다:

> A commit message is one screen; the note in `docs/commit/` is the
> page that survives.

몇 개 노트를 직접 읽어보면 이게 형식적인 규율이 아니라는 걸 알 수
있다. **0019 (제출한 결과를 정정하는 기능)**는 `reopen` 커맨드를
추가하면서 왜 "이미 render가 소비한 작업은 reopen을 거부해야 하는가"를
이렇게 설명한다:

> A wrong split announces itself as two thin notes, while a wrong
> fusion presents as one healthy note with nothing visibly wrong.

즉 엔티티를 잘못 나누면 눈에 띄지만, 잘못 합치면(잘못된 alias 하나가
서로 다른 두 개념을 병합해버리면) 겉보기엔 멀쩡한 노트 하나로
보인다는 것 — 그래서 검증기를 우회하는 손 편집이 특히 위험하다는
논리다.

**0026 (abstract는 커밋하지 않고 ledger만 남긴다)**는 "잃어버려도
되는 것"과 "잃어버리면 안 되는 것"을 구분하는 기준을 이렇게 정리한다:

> Losing the abstracts is recoverable; losing the ledger is not. ...
> A large debt on a fresh clone is the ledger working, not failing.

수집하지 못한 게 있어도 ledger(며칠치가 비었는지 기록한 원장)만
살아있으면 다음 실행이 정확히 무엇을 다시 가져와야 하는지 안다 —
반면 ledger 자체를 잃으면 "아무것도 없었던 날"과 "읽기를 실패한 날"을
구분할 방법이 영영 사라진다.

이런 노트들을 27개 쌓아두는 이유는 결국 하나다 — **이 레포는 다른
프로젝트로 복제되어 배포되는 것을 전제로 설계**되었고, 복제해 간
사람이 "이 결정을 지금도 유효한가, 우리 상황에서 바꿔야 하는가"를
코드만 보고는 재구성할 수 없기 때문이다.

---

## 8. Field-Neutral 하게 만들기 — 이름부터 바꾼 리팩터링

이 레포의 기원도 흥미롭다. **commit note 0004**가 기록하듯, 원래 이름은
*Recipe for World Action Model* 이었다 — 즉 특정 연구 주제(world
action model)를 위해 만들어졌다가, 이후 어떤 분야에도 종속되지 않는
범용 문헌 관리 시스템으로 리팩터링되었다. 노트가 이 리네이밍의
이유를 이렇게 짚는다:

> The old README said "nothing here is specific to one field"
> directly under a title that contradicted it.

기본값, 코드, 픽스처에서 특정 분야를 가리키는 흔적을 다 지워도
(0002, 0003), 제목 자체가 "이건 world action model 얘기"라고 계속
말하고 있었다는 것 — 그래서 이름을 **무엇을 하는지가 아니라 무엇을
위한 것인지**로 바꿨다. "Research Archive Toolkit" 같은 더 일반적인
이름은 오히려 기각되었는데, 메커니즘은 보면 알 수 있지만 의도는 알
수 없기 때문이라는 논리였다.

작은 디테일이지만, 범용 도구를 만들 때 "기능에서 필드 종속성을
제거하는 것"과 "이름과 문서에서 필드 종속성을 제거하는 것"이 별개의
작업이라는 걸 보여주는 좋은 사례다.

---

## 9. CCTT와 비교하면

| | CCTT (Chapter 5) | Recipe for Research Team Management (이 글) |
|---|---|---|
| 목적 | 논문 초안을 **쓴다** | 문헌을 **추적·정리**한다 |
| 핵심 루프 | 팀-리드가 루브릭으로 팀원 결과물을 평가 | 결정론적 파이프라인 ↔ 사람(모델) 판단, work queue로 분리 |
| 상태 관리 | `workspace/{topic}/`, 세션 간 handoff hook | `data/`가 유일한 source of truth, 나머지는 전부 파생 |
| 자기 확장 | 루브릭이 라운드마다 진화 | 위키가 언급 빈도에 따라 자동 승격 |
| 히스토리 규율 | 없음 (세션별 handoff 문서만) | 커밋마다 강제되는 commit note, 27개 누적 |
| 배포 전제 | 단일 사용자, 토픽별 클론 | 다른 프로젝트로의 재배포를 전제로 설계 |

두 레포 모두 "결정론적인 부분과 판단이 필요한 부분을 명시적으로
분리한다"는 같은 원칙을 공유한다 — CCTT는 그 경계를 팀-리드의 루브릭
평가로 긋고, 이 레포는 파일 기반 work queue로 긋는다. 다만 이 레포
쪽이 한 걸음 더 나아간 지점은 **재현성**이다 — `data/`만 있으면
`archive/`, `wiki/`, `outputs/`를 무조건 동일하게 복원할 수 있다는
불변식이, CCTT에는 없는 종류의 안정성을 준다.

---

## 10. 앞으로 지켜볼 것

- **스코어링 규칙의 한계.** `enrich/score.py`는 의도적으로 키워드
  규칙이다 — 읽을 수 있고, 반박할 수 있고, 토픽 YAML에서 고칠 수
  있다는 게 장점이지만, 의미적으로 관련 있으나 키워드가 겹치지 않는
  논문은 여전히 놓친다. 임베딩 기반 스코어링을 옵션으로 추가하는 게
  다음 확장으로 자연스러워 보인다.
- **읽기 백엔드의 다양화.** 기본 백엔드는 큐를 매일 비우는 Claude
  Code 세션이지만, `pipelines/common/llm.py`의 계약을 만족하는 두
  메서드만 구현하면 `anthropic`/`ollama` 백엔드로 API 기반 자동
  요약도 가능하다. 사람이 읽는 것과 모델이 자동으로 읽는 것 사이의
  품질 차이가 실제로 어느 정도인지는 지켜볼 지점이다.
- **여러 그룹이 공유하는 위키.** 지금은 레포 하나가 그룹 하나의
  상태다. 여러 그룹이 개념 노트를 공유하되 토픽 설정만 갈라지는
  형태로 확장되면, 위키의 "언급 2회 이상 승격" 임계값이 그룹 간에도
  의미를 가지게 될 것 같다.
- **Commit note가 실제로 읽히는가.** 27개의 노트를 쌓는 규율 자체는
  훌륭하지만, 이 레포를 복제해 가는 사람이 실제로 그 노트들을 읽고
  판단에 반영하는지는 별개의 문제다. `docs/commit/README.md`의
  인덱스 테이블이 그 마찰을 줄이려는 시도인데, 노트 수가 더 늘어날
  때도 스캔 가능성이 유지되는지 지켜볼 만하다.

---

## References

- Repository:
  [github.com/yjkim-stat/Recipe-for-Research-Team-Management-with-Claude](https://github.com/yjkim-stat/Recipe-for-Research-Team-Management-with-Claude).
- Series chapters:
  [1]({% link _explorations/2026-05-06-agent-harness-team.md %}),
  [2]({% link _explorations/2026-05-18-when-and-how-to-use-agents.md %}),
  [3]({% link _explorations/2026-05-18-research-agent-team-rules.md %}),
  [4]({% link _explorations/2026-05-25-implicit-instruction.md %}),
  [5 — CCTT]({% link _explorations/2026-05-28-cctt-research-team-repo.md %}),
  [8 — Adversarial Feedback Loop 설계]({% link _explorations/2026-07-06-adversarial-feedback-design.md %}),
  [9 — Ultracode Handoff Readiness]({% link _explorations/2026-07-29-ultracode-handoff-readiness.md %}).
