# Git 배포 모드

## 실행 순서

### 1단계 — 변경 사항 파악
`git status` 와 `git diff --stat` 로 변경된 파일 확인 후 요약:
```
변경된 파일:
  - _posts/2026-05-27-redis-cache.md (새 파일)
```

---

### 2단계 — 브랜치 생성 제안
변경 내용에 맞는 브랜치명 제안:

| 상황 | 브랜치명 형식 | 예시 |
|------|-------------|------|
| 새 포스트 | `feature/post-{제목}` | `feature/post-redis-cache` |
| 학습 포스트 | `feature/learn-{기술명}` | `feature/learn-redis` |
| 내용 수정 | `fix/{내용}` | `fix/redis-post-typo` |
| 설정 변경 | `chore/{내용}` | `chore/update-config` |

```
🌿 브랜치명 추천: feature/post-redis-cache

이 이름으로 진행할까요? (아니면 원하는 이름 입력)
```

확인 받은 후 브랜치 생성:
```bash
git checkout -b [브랜치명]
```

---

### 3단계 — 커밋 메시지 추천
변경 내용을 분석해서 커밋 메시지 **3가지** 추천:

```
📝 커밋 메시지를 선택해주세요:

1️⃣  docs: Redis 캐시 전략 학습 정리
2️⃣  post: Redis를 활용한 캐싱 경험 기록
3️⃣  add: Redis 시리즈 3편 — 캐시 전략 편

번호를 선택하거나 직접 입력해주세요.
```

---

### 4단계 — 승인 후 자동 실행
선택/입력 받은 후 아래 순서로 실행:

```bash
git add .
git commit -m "[선택한 메시지]"
git push origin [브랜치명]
```

PR 생성 (gh CLI 사용):
```bash
gh pr create \
  --title "[커밋 메시지와 동일]" \
  --body "## 변경 내용\n[변경 파일 요약]\n\n## 관련 학습\n[학습한 기술 연결]\n\n🤖 Generated with Claude Code" \
  --base main
```

PR 생성 후:
```
✅ PR이 생성됐어요!
🔗 [PR URL]

Merge할까요? (yes/no)
```

---

### 5단계 — Merge 및 완료
"yes" 입력 시:
```bash
gh pr merge --merge --delete-branch
```

완료 후:
1. `.claude/memory/블로그현황.md` 업데이트 (배포 완료로 변경)
2. 완료 메시지:
   ```
   🚀 배포 완료!
   
   ✅ 브랜치 머지 완료
   ✅ 브랜치 삭제 완료
   🌐 잠시 후 https://kth4778.github.io 에서 확인하세요
      (GitHub Actions 빌드: 약 1~2분 소요)
   ```
