# BigTae's Dev Log

> Backend Engineering & Problem Solving

개발자로 성장하며 마주한 고민, 선택, 해결 과정을 기록하는 기술 블로그입니다.

블로그 바로가기: [https://kth4778.github.io](https://kth4778.github.io)

## About

기술을 배우는 데서 멈추지 않고, 직접 적용하고 기록하며 성장하는 개발자 김태현의 기록 공간입니다.

주로 아래 내용을 다룹니다.

- 백엔드 개발
- 프로젝트 경험
- 트러블슈팅
- 회고

## Tech Stack

- GitHub Pages
- Jekyll
- [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) theme
- GitHub Actions

## Features

- 한국어 UI 및 `Asia/Seoul` 시간대 설정
- GitHub, LinkedIn, 이메일 연락처 연결
- [giscus](https://giscus.app/ko) 기반 댓글
- [GoatCounter](https://www.goatcounter.com/) 기반 게시글 조회수
- PWA 및 코드 하이라이팅

## Writing

게시글은 `_posts` 디렉터리에 Markdown 파일로 작성합니다.

```text
_posts/YYYY-MM-DD-post-title.md
```

예시:

```markdown
---
title: "GitHub 기술 블로그를 시작한 이유"
date: 2026-05-25 23:00:00 +0900
categories: [회고]
tags: [github-blog, jekyll, chirpy]
---

본문을 작성합니다.
```

## Local Development

Ruby와 Bundler가 설치된 환경에서 의존성을 설치합니다.

```bash
bundle install
```

로컬 서버를 실행합니다.

```bash
bundle exec jekyll serve --livereload
```

또는 템플릿에 포함된 실행 스크립트를 사용할 수 있습니다.

```bash
bash tools/run.sh
```

프로덕션 빌드와 링크 검증:

```bash
bash tools/test.sh
```

## Deployment

`main` 브랜치에 변경 사항이 반영되면 GitHub Actions가 사이트를 빌드하고 GitHub Pages에 배포합니다.

배포 과정:

1. Jekyll production 빌드
2. `html-proofer` 링크 검증
3. GitHub Pages artifact 업로드 및 배포

## Contact

- GitHub: [kth4778](https://github.com/kth4778)
- LinkedIn: [TaeHyeon Kim](https://www.linkedin.com/in/%ED%83%9C%ED%98%84-%EA%B9%80-571897392/)
- Email: [cnblue123429@gmail.com](mailto:cnblue123429@gmail.com)

## License

이 블로그는 [Chirpy Starter](https://github.com/cotes2020/chirpy-starter)를 기반으로 구성되었으며, 테마 관련 라이선스는 [MIT License](LICENSE)를 따릅니다.
