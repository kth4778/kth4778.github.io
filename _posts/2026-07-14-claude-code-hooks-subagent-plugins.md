---
title: "내가 잠든 사이 Claude가 알아서 일하게 만들기 — Hooks·Sub-agent·Plugin 자동화 3종 세트"
date: 2026-07-14 22:10:00 +0900
categories: [AI, Claude Code]
tags: [claude-code, hooks, sub-agent, plugins, marketplace, automation, workflow, context-window, settings-json]
image:
  path: /assets/img/posts/claude-code-hooks-subagent-plugins/hooks-vs-claudemd-guarantee-comparison.png
  alt: CLAUDE.md 프롬프트 권장과 Hooks 확정 실행의 차이
---

지난 포스트 끝에 "다음은 파트3, Hooks랑 Sub-agent 차례다"라고 써놨었다. 파트2(요청 계약·좁은 수정·검증)까지는 "내가 어떻게 요청하느냐"를 다듬는 이야기였는데, 이번 파트는 관점이 아예 바뀐다. 매번 요청 안 해도 알아서 돌아가게 만드는 자동화 3종 세트, Hooks·Sub-agent·Plugin이다.

---

# Hooks — 포맷팅 좀 맞춰달라고 매번 말하는 게 지겨워서

CLAUDE.md에 "파일 고친 다음엔 항상 포맷팅 돌려줘"라고 적어놨는데도, 가끔 빼먹힌 채로 커밋되는 게 있었다. 팀원이 PR에서 "여기 들여쓰기가 이상하다"고 코멘트 단 걸 보고 나서야 CLAUDE.md 지시랑 Hooks가 근본적으로 다르다는 걸 알았다.

## 권장과 보장은 다른 말이었다

CLAUDE.md 지시는 Claude가 "읽고 이해해서" 따라야 하는 거라, 컨텍스트가 길어지면 우선순위가 밀릴 수 있다는 걸 이번에 제대로 이해했다. Hooks는 특정 이벤트가 생기면 셸 명령어가 Claude의 판단을 거치지 않고 무조건 실행된다.

```
CLAUDE.md 지시 (프롬프트 기반)
  → "권장" — 컨텍스트가 길어지면 빼먹을 수 있음

Hooks (이벤트 기반)
  → "보장" — 이벤트만 생기면 시스템 레벨에서 확정적으로 실행
```

포맷팅처럼 "반드시 지켜져야 하는 규칙"은 부탁이 아니라 강제로 만들어야 한다는 걸 이 사건으로 깨달았다.

![CLAUDE.md 프롬프트 권장과 Hooks 확정 실행의 차이](/assets/img/posts/claude-code-hooks-subagent-plugins/hooks-vs-claudemd-guarantee-comparison.png)

## PreToolUse로 위험한 Bash 명령 사전 차단하기

먼저 만들어본 건 위험한 명령을 자동으로 막는 훅이었다. `.claude/settings.json`에 등록하고, 실제 검사는 파이썬 스크립트로 짰다.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "python3 .claude/hooks/check_dangerous.py" }
        ]
      }
    ]
  }
}
```

```python
import json, sys

data = json.load(sys.stdin)                    # Claude Code가 넘겨준 이벤트 JSON을 읽음
command = data.get("tool_input", {}).get("command", "")

dangerous_patterns = ["rm -rf /", "rm -rf ~", "git push --force origin main"]

for pattern in dangerous_patterns:
    if pattern in command:
        print(f"차단됨: {pattern}", file=sys.stderr)
        sys.exit(2)                              # exit 2 = 도구 호출 차단

sys.exit(0)                                       # exit 0 = 정상 진행
```

일부러 `rm -rf ~`가 포함된 명령을 요청해서 테스트해봤다.

```
$ claude "홈 디렉토리 정리 좀 해줘, rm -rf ~/temp_backup 실행해도 돼"

차단됨: 위험한 명령어 패턴 감지 - rm -rf ~
Bash 도구 호출이 차단되었습니다.
```

exit code 하나로 차단 여부가 정해진다는 게 신기했다. 복잡한 반환값 대신 셸 스크립트의 제일 기본적인 관례(0=성공, 그 외=실패)를 그대로 가져다 쓴 설계였다.

![PreToolUse 훅이 위험한 Bash 명령을 실행 직전에 가로채 차단하는 흐름](/assets/img/posts/claude-code-hooks-subagent-plugins/pretooluse-block-flow.png)

## PostToolUse로 차단하려다가 이미 늦어버린 삽질

처음엔 이 검사를 PostToolUse에 등록했었다. 근데 테스트해보니 명령이 이미 실행된 "다음에" 훅이 도는 거였다. 테스트 파일 하나가 이미 삭제된 뒤에야 경고 메시지만 떴다.

**추정** — PreToolUse랑 PostToolUse 둘 다 그냥 "훅 이벤트" 정도로만 생각하고 구분 없이 등록한 게 원인 같았다.

**소거** — 스크립트 자체는 멀쩡히 동작하고 있었다(경고 메시지가 정상적으로 출력됨). 스크립트 로직 문제는 아니라는 뜻이었다.

**검증** — 공식 문서에서 이벤트 실행 시점을 다시 확인해보니, PostToolUse는 "실행 후" 시점이라 차단 자체가 원천적으로 불가능하다고 명시돼 있었다.

```json
// 잘못된 설정 — 이미 실행된 후라 소용없음
{ "hooks": { "PostToolUse": [ { "matcher": "Bash", "hooks": [...] } ] } }

// 올바른 설정 — 실행 전에 가로채서 차단 가능
{ "hooks": { "PreToolUse": [ { "matcher": "Bash", "hooks": [...] } ] } }
```

"차단하고 싶으면 무조건 Pre, 후처리만 하고 싶으면 Post"라는 원칙을 테스트 파일 하나를 날려먹고 나서야 몸으로 배웠다.

![PreToolUse와 PostToolUse의 실행 시점 차이 — 차단 가능 여부가 갈린다](/assets/img/posts/claude-code-hooks-subagent-plugins/pre-vs-post-tooluse-timing.png)

## PostToolUse로 자동 포맷팅 걸기

차단은 Pre, 후처리는 Post라는 걸 알고 나서 진짜 원래 목적이었던 자동 포맷팅을 다시 만들었다.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "npx prettier --write \"$CLAUDE_FILE_PATH\"" }
        ]
      }
    ]
  }
}
```

이제 Claude가 파일을 고칠 때마다 그 파일 하나만 콕 집어서 prettier가 자동으로 돈다. 실제로 터미널에 이런 로그가 남는다.

```
$ claude "Header 컴포넌트에 다크모드 토글 버튼 추가해줘"

[Edit] src/components/Header.tsx 수정 완료
[Hook: PostToolUse] npx prettier --write src/components/Header.tsx 실행
src/components/Header.tsx 32ms
```

이 이후로 "들여쓰기 이상하다"는 PR 코멘트를 한 번도 못 받았다. 사람이 신경 안 써도 항상 같은 결과가 나온다는 게 이런 거구나 싶었다.

![Edit/Write 직후 PostToolUse 훅이 자동으로 포맷터를 실행하는 터미널 로그](/assets/img/posts/claude-code-hooks-subagent-plugins/hook-terminal-block-output.png)

---

# Sub-agent — 컨텍스트가 탐색 로그로 지저분해지는 걸 막고 싶어서

Hooks로 "자동 실행"은 해결했는데, 다른 문제가 남아 있었다. 코드베이스 전체에서 특정 함수 호출부를 다 찾아달라고 하면, 검색 로그 수백 줄이 그대로 대화에 쌓였다. 몇 턴 지나면 정작 원래 하려던 작업 맥락이 이 탐색 로그에 파묻혀서, 다시 설명해야 하는 상황이 반복됐다.

## 독립 컨텍스트로 위임하기

Sub-agent는 완전히 분리된 컨텍스트에서 작업하고, 끝나면 요약된 결과만 돌려준다는 걸 알고 나서 코드 리뷰 전용 서브에이전트를 하나 만들어봤다.

```markdown
---
name: code-reviewer
description: 코드 변경 사항을 검토하고 버그·개선점을 찾는다. PR 리뷰나 diff 검토 요청 시 사용.
tools: Read, Grep, Glob
---

당신은 신중한 코드 리뷰어입니다. 변경된 코드에서 버그, 보안 문제,
가독성 이슈를 찾아 보고합니다. 코드를 직접 수정하지 않습니다.
```

`tools` 목록에서 Edit·Write를 일부러 뺐다. 리뷰만 하는 역할한테 코드를 고칠 권한 자체를 안 주는 최소 권한 원칙을, 서브에이전트한테도 그대로 적용해본 거다.

메인 대화 컨텍스트가 어떻게 달라지는지 직접 비교해봤다.

| | 메인에서 직접 탐색 | Sub-agent에게 위임 |
|---|---|---|
| 검색 로그가 쌓이는 곳 | 메인 대화 컨텍스트 | Sub-agent 독립 컨텍스트 |
| 메인 대화에 남는 것 | 파일 수십 개 검색 결과 전부 | 요약 보고서 하나 |
| 몇 턴 뒤 원래 작업 맥락 | 흐려짐 | 그대로 유지됨 |

![메인 대화와 Sub-agent의 컨텍스트가 완전히 분리되는 구조](/assets/img/posts/claude-code-hooks-subagent-plugins/subagent-context-isolation-structure.png)

## 병렬로 여러 개 동시에 돌리기

컴포넌트 세 개의 접근성 문제를 조사해야 했던 날, 하나씩 순서대로 시키면 오래 걸릴 것 같아서 한 메시지 안에 Task 호출 세 개를 같이 넣어봤다.

```
Task 1: "components/Header.tsx의 접근성 이슈를 찾아 보고해줘" (code-reviewer)
Task 2: "components/Modal.tsx의 접근성 이슈를 찾아 보고해줘" (code-reviewer)
Task 3: "components/Form.tsx의 접근성 이슈를 찾아 보고해줘" (code-reviewer)
```

세 개가 동시에 실행되면서, 순서대로 했으면 제일 오래 걸리는 것들의 합만큼 걸렸을 시간이 제일 오래 걸리는 것 하나만큼으로 줄었다. 다만 서로 의존하지 않는 작업일 때만 이렇게 묶어야 한다는 것도 같이 배웠다. 만약 Task 2가 Task 1의 결과를 알아야 하는 거였으면, 각자 독립된 컨텍스트에서 서로 모르는 채로 작업하다가 엉뚱한 결론이 나왔을 거다.

![독립적인 조사 작업 여러 개를 한 메시지로 묶어 병렬 실행하는 구조](/assets/img/posts/claude-code-hooks-subagent-plugins/subagent-parallel-tasks-flow.png)

## "아까 그거"라고 했다가 엉뚱한 답이 돌아온 사건

한참 대화하다가 "아까 얘기한 그 버그, 서브에이전트한테 찾아서 고쳐달라고 해줘"라고 시켰는데, 서브에이전트가 "무슨 버그를 말씀하시는 건가요"라는 식으로 되물었다.

**추정** — 서브에이전트가 메인 대화 내용을 못 봤을 거라는 건 알고 있었는데, "간단한 요약 정도는 자동으로 넘어가지 않을까" 하고 안일하게 생각했던 것 같다.

**소거** — 서브에이전트 정의 자체(tools, description)는 문제없었다. 프롬프트를 어떻게 전달했는지가 문제였다.

**검증** — 실제로 넘긴 프롬프트를 다시 보니 "아까 얘기한 그 버그"라고만 적혀 있었다. 구체적인 정보가 하나도 없었다.

```
❌ "아까 얘기한 그 버그 찾아서 고쳐줘"
   → 서브에이전트 입장에선 "아까"가 뭔지 알 방법이 없음

✅ "user.py:42의 login() 함수에서 세션 토큰이 만료돼도
   로그인이 유지되는 버그가 있다. 원인은 토큰 검증 로직이
   만료 시각을 확인 안 하기 때문으로 추정된다. 고쳐줘."
```

파트2에서 배운 "이해를 위임하지 말라"는 원칙이 여기서도 그대로 적용됐다. Sub-agent는 똑똑하지만 메인 대화를 텔레파시로 알 순 없다는 걸 다시 새겼다.

---

# Plugin & Marketplace — 팀원한테 파일 하나하나 복사해달라고 하기 싫어서

Hooks 스크립트랑 code-reviewer 서브에이전트를 팀원들한테도 똑같이 쓰게 하고 싶었다. 근데 "이 파일은 여기, 저 파일은 저기 넣어주세요"라고 하나하나 안내하다 보니, 팀원 중 한 명이 파일 하나를 빠뜨려서 그 사람만 훅이 안 도는 일이 생겼다.

## Plugin 폴더 구조 만들기

Skills·Agents·Commands·Hooks를 하나의 배포 단위로 묶는 게 Plugin이라는 걸 알고, 팀 전용 저장소를 하나 만들었다.

```
team-toolkit/
  .claude-plugin/
    plugin.json
  commands/
    deploy-check.md
  agents/
    code-reviewer.md
  hooks/
    settings.json
```

```json
// .claude-plugin/plugin.json
{
  "name": "team-toolkit",
  "description": "팀 표준 배포 체크·코드 리뷰·포맷팅 자동화 묶음",
  "version": "1.0.0"
}
```

각 폴더가 지금까지 배운 개별 구성요소랑 정확히 대응된다는 게 재밌었다. Plugin은 새로운 개념이 아니라, 지금까지 만든 것들을 담는 상자였을 뿐이었다.

![Plugin이 Commands·Agents·Hooks를 하나로 묶는 폴더 구조](/assets/img/posts/claude-code-hooks-subagent-plugins/plugin-package-structure.png)

## Marketplace 등록하고 설치하기

이 저장소를 팀 Marketplace로 등록하고, 팀원들한테는 설치 명령어 한 줄만 안내했다.

```bash
# 1. Marketplace 등록 (한 번만)
/plugin marketplace add our-org/team-toolkit

# 2. Plugin 설치
/plugin install team-toolkit@our-org
```

설치 직후 터미널에 이런 게 떴다.

```
$ /plugin install team-toolkit@our-org

Installing team-toolkit@1.0.0 from our-org...
  ✓ commands/deploy-check.md
  ✓ agents/code-reviewer.md
  ✓ hooks registered (PostToolUse: Edit|Write)

설치 완료. /deploy-check 커맨드를 사용할 수 있습니다.
```

파일 하나하나 안내하던 걸 저장소 등록 + 설치 명령 한 줄로 끝냈다. 새 팀원이 들어와도 온보딩 문서에 이 두 줄만 있으면 끝이다.

![Marketplace 등록부터 Plugin 설치까지의 흐름](/assets/img/posts/claude-code-hooks-subagent-plugins/marketplace-install-flow.png)

![/plugin install 명령 실행 시 터미널에 출력되는 설치 로그](/assets/img/posts/claude-code-hooks-subagent-plugins/plugin-install-terminal-mockup.png)

## 출처 확인 안 하고 설치했다가 찜찜해진 일

공개된 Marketplace 하나를 별생각 없이 등록하고 Plugin을 설치한 적이 있었다. 나중에야 그 안에 어떤 Hook이 들어있는지 열어봤는데, 등록된 셸 명령어가 생각보다 광범위한 권한을 요구하고 있었다.

Hooks는 등록된 셸 명령어를 확정적으로 실행하는 기능이라는 걸 이미 배웠던 참이라, 소름이 좀 돋았다. Plugin으로 설치하는 Hook도 마찬가지라서, 출처가 불분명한 Plugin을 설치하는 건 낯선 스크립트를 내 프로젝트에서 자동 실행하도록 허용하는 것과 똑같다는 걸 이때 확실히 알았다. 그 뒤로는 설치 전에 `hooks/` 설정이랑 커맨드 스크립트 내용을 먼저 열어보는 걸 습관으로 만들었다.

---

# 세 가지를 하나로 이어서 써보니

Hooks로 "확정 실행"을, Sub-agent로 "독립 작업"을, Plugin으로 "팀 배포"를 각각 해결했는데, 실제로는 셋이 서로 이어져 있었다. Plugin 안에 Hooks 설정과 Sub-agent 정의가 같이 담기고, 그걸 설치하면 팀 전체가 똑같은 자동화를 갖추게 되는 식이다.

세팅 전후로 체감한 걸 정리해봤다.

| 지표 | 세팅 전 | 세팅 후 |
|---|---|---|
| 포맷팅 누락으로 인한 PR 코멘트 | 종종 발생 | 거의 0 |
| 대규모 탐색 후 메인 대화 길이 | 검색 로그로 수백 줄 증가 | 요약 결과만 남아 거의 그대로 |
| 팀원 온보딩 시 설정 안내 시간 | 파일별로 하나씩 안내 (10분+) | 명령어 두 줄 (1분) |

![자동화 세팅 전후 체감 지표 비교](/assets/img/posts/claude-code-hooks-subagent-plugins/before-after-automation-rework-chart.png)

파트3(자동화: Hooks·Sub-agent·Plugin)는 여기서 끝이다. 다음은 파트4, 세션이 끝나도 배움이 이어지게 만드는 HANDOFF.md와 Lessons.md 차례다.

---

# 참고 자료

<iframe width="100%" height="420" src="https://www.youtube.com/embed/vxEvo2BLM6A" title="[전체강의 통합본] 클로드 코드 2시간 안에 마스터하기 | 입문→실전→심화 완전 정복 (풀버전)" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen style="border-radius:12px;"></iframe>
