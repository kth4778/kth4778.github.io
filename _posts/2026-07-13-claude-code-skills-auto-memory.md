---
title: "SKILL.md는 안 만들어도 됐다 — Claude Code 커맨드와 자동 기억 세팅기"
date: 2026-07-13 21:00:00 +0900
categories: [AI, Claude Code]
tags: [claude-code, skills, slash-command, claude-md, auto-memory, memory-system, workflow, dotclaude]
image:
  path: /assets/img/posts/claude-code-skills-auto-memory/skill-command-file-structure.png
  alt: .claude/commands 폴더와 슬래시 커맨드 파일 구조
---

지난 포스트 마지막에 "SKILL.md, Hooks, Sub-agent는 아직 제대로 세팅 안 했다. 다음 포스트에서 이어서 쓸 예정이다"라고 써놓고 두 달 가까이 방치했다. 이번에 이 블로그 자동화 커맨드(`/teach`, `/new-post`, `/deploy`, `/status`)를 실제로 만들면서 그 SKILL.md를 드디어 손댔는데, 막상 해보니 예상과 다르게 흘러갔다.

---

# SKILL.md 대신 진짜로는 이랬다

저번 글 쓸 때만 해도 "SKILL.md라는 파일 하나를 만들면 되겠지" 정도로 생각했다. 실제로 커맨드를 만들려고 문서를 다시 찾아보니 파일 이름이 `SKILL.md`가 아니라 `.claude/commands/` 폴더 안에 커맨드 이름으로 된 마크다운 파일 여러 개였다.

```
.claude/commands/
  ├─ teach.md      → /teach 입력 시에만 로드
  ├─ new-post.md   → /new-post 입력 시에만 로드
  ├─ deploy.md     → /deploy 입력 시에만 로드
  └─ status.md     → /status 입력 시에만 로드
```

이 구조를 보고 나서야 지난 포스트에서 개념적으로만 이해했던 "SKILL.md는 필요할 때만 로드된다"가 실제로 뭘 의미하는지 감이 왔다. 파일 하나가 아니라, 커맨드 개수만큼 파일이 쪼개져 있고 그 이름이 곧 호출 키워드였다.

`teach.md` 안을 열어보면 이런 식으로 단계별 절차가 적혀 있다.

```markdown
# 선생님 모드

## -1단계 — 인자 없을 때 클릭 메뉴로 주제/단계 선택
...

## 0단계 — 진행 상황 확인
`.claude/memory/학습진행도.md`를 읽어서...
...

## 4단계 — 학습 진행
[아래 7가지 순서로 가르친다]
1. 등장 배경
2. 설계 철학
...
```

이게 진짜 이 블로그 프로젝트에서 매번 `/teach`를 칠 때마다 로드되는 실제 파일 내용이다. 회사로 치면 신입 온보딩 매뉴얼 전체를 CLAUDE.md에 넣는 대신, "코드 리뷰 SOP", "배포 체크리스트"처럼 업무별로 문서를 쪼개서 사내 위키에 올려두고 그때그때 필요한 문서만 펼쳐보는 것과 같은 구조였다. 카카오나 토스 같은 곳의 기술 블로그에서 "런북(runbook)을 업무 단위로 쪼갰다"는 글을 본 적 있는데 딱 그 얘기였다.

처음엔 "그냥 CLAUDE.md에 커맨드 절차까지 다 넣으면 편하지 않나" 싶었는데, 직접 4개를 만들어보고 나서 생각이 바뀌었다. `/teach`용 절차만 300줄 가까이 되는데, 이걸 CLAUDE.md에 넣었으면 `/deploy` 한 번 칠 때도 그 300줄이 매번 컨텍스트에 딸려 왔을 거다.

![.claude/commands 폴더 구조와 teach.md 파일 내용](/assets/img/posts/claude-code-skills-auto-memory/skill-command-file-structure.png)

---

# 커맨드가 안 먹혔던 삽질

커맨드 파일을 처음 만들 때 `deploy.md`를 폴더 밖 프로젝트 루트에 잘못 두고 `/deploy`를 쳤다. 아무 반응 없이 그냥 일반 대화로 넘어갔다. "왜 안 되지?" 싶어서 한참 헤맸다.

**추정** — 혹시 커맨드 이름 자체를 잘못 쳤나 싶어서 다시 확인했다. `/deploy` 철자는 맞았다.

**소거** — 그럼 파일 내용에 문법 오류가 있나 싶어서 다른 정상 작동하는 `teach.md`랑 비교했다. 마크다운 문법 차이는 없었다.

**검증** — 마지막으로 파일 위치를 확인했다. `.claude/commands/deploy.md`가 아니라 그냥 프로젝트 루트에 `deploy.md`로 저장돼 있었다. 폴더를 잘못 짚은 거였다.

```
❌ 이렇게 뒀었음
프로젝트루트/deploy.md

✅ 이래야 함
프로젝트루트/.claude/commands/deploy.md
```

폴더를 옮기고 다시 `/deploy`를 치니까 바로 인식됐다. 별거 아닌 실수였는데 "커맨드가 왜 하필 이 이름이어야 하지?" 보다 "이 파일이 대체 어디 있어야 인식되지?"가 더 자주 막히는 지점이라는 걸 알았다. 시스템 프롬프트에 어떤 커맨드가 로드 가능한 상태인지 목록으로 뜨는데, 그 목록에 커맨드 이름이 안 보이면 위치 문제일 확률이 높다는 것도 이때 배웠다.

![슬래시 커맨드가 인식되지 않는 상황 재현](/assets/img/posts/claude-code-skills-auto-memory/slash-command-not-triggered-error.png)

---

# Lessons.md 만들려다 알게 된 것

커맨드는 그렇게 세팅했고, 이제 지난 포스트에서 약속했던 Lessons.md 차례였다. 내 계획은 이랬다. 세션 중에 실수하거나 사용자가 뭔가 고쳐주면 `Lessons.md`라는 파일 하나에 그때그때 적어두고, 나중에 CLAUDE.md로 승격시키는 방식.

실제로 이 블로그 프로젝트에 `.claude/memory/학습진행도.md`, `블로그현황.md`, `글쓰기스타일.md`를 만들어서 비슷한 걸 하고 있었다. `/teach` 실행할 때마다 학습진행도.md를 갱신하고, `/new-post` 실행할 때마다 블로그현황.md를 갱신하는 식이다. 그런데 이건 내가 CLAUDE.md 규칙으로 직접 설계해서 Skill이 파일을 읽고 쓰게 만든 거였다. Claude Code가 알아서 해주는 게 아니라 내가 절차를 짜서 강제한 구조였다.

그러다 다른 얘기를 하다가 알게 됐는데, Claude Code에는 이거랑 별개로 진짜 자동으로 동작하는 기억 시스템이 이미 있었다. 대화 도중 내가 뭔가를 교정해주거나 프로젝트 사정을 설명하면, 별도로 시키지 않아도 그 내용을 알아서 파일로 저장해두고 다음 세션에서 자동으로 불러온다. 내가 직접 설계했던 학습진행도.md 같은 파일들은 "이 프로젝트 전용 상태 기록"이었고, 이 자동 기억 시스템은 그거랑 완전히 다른 층에서 따로 돌아가고 있던 거다.

굳이 Lessons.md를 직접 만들 필요가 없었다. 이미 있었다.

![Lessons.md 직접 관리 방식과 자동 기억 시스템 구조 비교](/assets/img/posts/claude-code-skills-auto-memory/lessons-md-vs-auto-memory-structure.png)

---

# 메모리도 타입별로 나뉜다

자동 기억 시스템을 좀 더 뜯어보니, 그냥 아무 문장이나 저장하는 게 아니라 저장될 때부터 종류가 나뉘어 있었다. 크게 네 가지였다.

| 타입 | 언제 저장되나 |
|---|---|
| `user` | 사용자의 역할·성향을 알게 됐을 때 |
| `feedback` | 사용자가 방식을 교정해줬을 때 |
| `project` | 진행 중인 작업·의사결정을 알게 됐을 때 |
| `reference` | 외부 자료 위치를 알게 됐을 때 |

이 블로그 프로젝트 기준으로 실제로 쌓인 걸 보면 `feedback` 타입이 압도적으로 많았다. `/teach` 세션 마무리 형식을 고쳐준 것, 진행률 표시 방식을 바꿔달라고 한 것, 모드 전환 요청을 처리하는 방식 같은 것들. 반대로 `user`(나에 대한 정보)나 `reference`(외부 자료 위치) 타입은 거의 없었다.

처음엔 "왜 하나로 안 몰아넣고 굳이 나눴을까" 싶었는데, 곰곰이 생각해보니 판단 속도 문제였다. 지금 하려는 작업이 "글쓰기 방식"에 관한 거면 `feedback` 타입만 훑으면 되고, "이 프로젝트가 지금 어떤 상태인지" 궁금하면 `project` 타입만 보면 된다. 다 섞여 있으면 관련 없는 기억까지 매번 다 뒤져야 한다.

![user·feedback·project·reference 네 가지 메모리 타입 분류](/assets/img/posts/claude-code-skills-auto-memory/memory-four-types-classification.png)

---

# 실제 메모리 파일 하나 까보기

`feedback` 타입 파일 하나를 실제로 열어보면 이런 구조다. (내용은 이해를 돕기 위해 단순화했다.)

```markdown
---
name: feedback-teach-mode-switch-midsection
description: mode1로 진행해 요청 시 즉시 전환, 다음 섹션엔 다시 질문
metadata:
  type: feedback
---

소단원 진행 중 "mode1로 진행해"처럼 모드 전환을 요청하면
그 자리에서 즉시 전환해서 이어간다.

**Why:** 세분화 학습 중 흐름이 답답하다고 직접 요청한 것이라
세션 끝까지 기다리지 않고 즉시 반영해야 함.

**How to apply:** 전환은 그 섹션 안에서만 유지되고,
다음 섹션이 시작되면 모드를 다시 묻는다.
```

이 구조를 보고 제일 인상 깊었던 건 `Why`와 `How to apply`를 강제로 나눠 쓰게 해둔 부분이었다. 그냥 "규칙: 모드 전환 즉시 반영"이라고만 적어뒀으면, 나중에 이 파일을 다시 읽는 세션 입장에서는 "이게 지금도 유효한 규칙인가?"를 판단할 근거가 없다. 근데 이유가 같이 적혀 있으면 "아, 사용자가 흐름 답답해서 요청한 거였구나. 지금도 같은 상황이면 적용하면 되겠다"처럼 스스로 판단할 수 있다.

실무에서 문서 작성할 때 "왜 이렇게 했는지" 안 쓰고 "결론만" 남기면 몇 달 뒤 아무도 이유를 모르게 되는 문제랑 똑같다. 사내 위키에 "이 API는 캐시를 쓰지 않는다"라고만 적혀 있으면 다음 담당자가 캐시를 붙였다가 장애를 낼 수도 있다. "왜 캐시를 안 쓰는지"까지 적혀 있어야 판단이 가능하다.

![feedback 타입 메모리 파일의 frontmatter와 Why/How to apply 구조](/assets/img/posts/claude-code-skills-auto-memory/memory-file-frontmatter-anatomy.png)

---

# MEMORY.md는 도서관 카드 목록이었다

메모리 파일이 쌓이다 보니 궁금해졌다. "이게 10개, 20개 쌓이면 세션 시작할 때마다 다 읽는 건가? 그럼 그것도 무거워지는 거 아닌가."

답은 `MEMORY.md`라는 인덱스 파일에 있었다. 개별 메모리 파일 하나하나의 전체 내용이 아니라, 한 줄 요약만 모아둔 목록 파일을 먼저 읽는다.

```
- [teach 세션 마무리 형식](feedback_teach_session_end.md) — 진행도+포스트 묶음+이해도 확인 질문 포함할 것
- [teach 상호작용 스타일](feedback_teach_interaction_style.md) — 예시 인물명 쓰지 말 것
- [new-post 분량 기준](feedback_newpost_depth.md) — teach 분량만큼 채우기 + 실제 서비스 사례 녹이기
```

이게 정확히 도서관 카드 목록이었다. "3층 A구역에 역사책 있음"이라고 카드만 먼저 훑고, 필요한 책만 그 자리로 가서 꺼내는 것처럼, 세션 시작할 때 이 인덱스만 먼저 읽고 지금 하려는 작업과 관련 있어 보이는 파일만 골라서 열어본다.

```
① 세션 중 배운 것 발견
        ↓
② 개별 파일로 저장 (frontmatter: name, description, type)
        ↓
③ MEMORY.md 인덱스에 한 줄 포인터 추가
        ↓
④ 다음 세션 시작 시 인덱스만 먼저 로드 → 필요한 파일만 골라 읽음
```

이 흐름을 보고 나니 앞서 만든 `.claude/commands/` 구조랑 원리가 똑같다는 걸 알았다. 커맨드는 "이름을 부를 때만" 로드되고, 메모리는 "관련 있어 보일 때만" 로드된다. 둘 다 "항상 켜두지 않고 필요할 때만 켠다"는 하나의 원칙 위에서 다르게 구현된 거였다.

![세션에서 배운 것이 저장되고 MEMORY.md 인덱스를 거쳐 다음 세션에 로드되는 흐름](/assets/img/posts/claude-code-skills-auto-memory/memory-md-index-library-card-analogy.png)

---

# 팀 공유 커맨드 vs 나만 쓰는 커맨드

커맨드 위치도 하나만 있는 게 아니었다. 이 블로그 프로젝트의 `/teach`, `/new-post`, `/deploy`, `/status`는 프로젝트 폴더 안 `.claude/commands/`에 있다. git으로 커밋되기 때문에 이 저장소를 나 말고 다른 사람이 클론해도 똑같이 쓸 수 있다.

반면 사용자 홈 폴더(`~/.claude/commands/`)에 두면 이 컴퓨터 전체에서, 어떤 프로젝트를 열든 항상 쓸 수 있다. 예를 들어 "커밋 메시지는 항상 이 형식으로 써줘" 같은, 프로젝트랑 상관없이 나만 매번 쓰고 싶은 절차가 있으면 여기가 맞는 자리다.

```
프로젝트폴더/.claude/commands/    ← 팀·저장소 전체 공유 (git 추적됨)
~/.claude/commands/               ← 이 컴퓨터의 나만 사용 (git 추적 안 됨)
```

이건 CLAUDE.md 자체도 똑같은 구조였다. 이 블로그 프로젝트를 보면 `C:\Users\cnblu\.claude\CLAUDE.md`(글로벌, 모든 프로젝트 공통 — 한국어로 답하기, 이모지 금지 같은 개인 규칙)와 프로젝트 루트의 CLAUDE.md(이 블로그 전용 — 브랜치 전략, 카테고리 규칙)가 따로 있다. "이 규칙이 이 프로젝트에서만 의미 있는가, 아니면 내가 어디서든 쓰고 싶은가"로 위치를 가르면 된다는 걸 몸으로 이해했다.

![프로젝트 전용 커맨드와 사용자 전용 커맨드 위치 비교](/assets/img/posts/claude-code-skills-auto-memory/project-vs-personal-commands-location.png)

---

# 실제로 뭐가 달라졌나

세팅하고 나서 며칠 지나 실제로 체감한 걸 정리해봤다.

| 항목 | 세팅 전 | 세팅 후 |
|---|---|---|
| `/teach` 세션 마무리 형식 재지적 횟수 | 세션마다 반복 | feedback 메모리 저장 이후 0회 |
| 새 세션 시작할 때 진행 상황 재설명 | 매번 직접 설명 | `/status` 한 번으로 확인 |
| CLAUDE.md 안 커맨드 절차 분량 | (계획만 있고 실체 없었음) | 0줄 (표 한 줄씩만, 절차는 전부 커맨드 파일로 분리) |

제일 크게 체감한 건 세 번째다. CLAUDE.md에는 지금 커맨드 목록이 표로 4줄 있을 뿐이고, `/teach`의 실제 절차는 300줄 가까이 되지만 CLAUDE.md 어디에도 안 보인다. 지난 포스트에서 "CLAUDE.md가 200줄 넘어가니까 중요한 규칙이 묻히더라"고 썼던 문제가, 커맨드를 파일로 분리하고 나니 자연스럽게 해결됐다.

![세팅 전후 재지적 횟수와 CLAUDE.md 절차 분량 변화](/assets/img/posts/claude-code-skills-auto-memory/before-after-repeated-feedback-count.png)

Hooks랑 Sub-agent는 여전히 손을 못 댔다. 이번엔 진짜로 다음 포스트에서 다룰 생각이다.

---

# 참고 자료

<iframe width="100%" height="420" src="https://www.youtube.com/embed/1_bRmkUvjHA" title="Claude Code 왕초보 입문 튜토리얼 23가지 팁 | 클로드코드 설치 사용법 50분 강의 완전정복 2026 가이드" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen style="border-radius:12px;"></iframe>
