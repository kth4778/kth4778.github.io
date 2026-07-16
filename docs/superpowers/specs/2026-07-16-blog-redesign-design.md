# 블로그 디자인 전면 교체 설계

작성일: 2026-07-16
브랜치: develop
상태: 설계 확정, 구현 대기

---

## 1. 목표

Chirpy 테마를 걷어내고 모던 테크블로그 레이아웃을 자체 구축한다. 기존 포스트 26편은 파일을 수정하지 않고 그대로 새 레이아웃 위에 얹는다.

레퍼런스는 카카오페이 기술블로그(https://tech.kakaopay.com/)의 화면 구조다. 히어로 배너 + 피처드 3열 + 좌썸네일 리스트 + 우측 태그 사이드바라는 구성을 동일한 인상으로 재현한다.

## 2. 범위 경계

### 재현하는 것
레이아웃 구조, 섹션 순서, 컴포넌트 배치, 간격 체계, 카드 비율. 모던 테크블로그의 일반적인 화면 문법에 해당하는 부분.

### 재현하지 않는 것
- 카카오페이 로고 및 브랜드 마크
- 카카오페이 브랜드 옐로 (본 설계는 딥 네이비 팔레트를 사용)
- 카카오페이가 게시한 일러스트 이미지 자산
- 카카오페이 사이트의 CSS/HTML 소스

썸네일 일러스트는 별도 작업으로 신규 제작한다. 플랫 벡터 + 단색 배경 블록이라는 일반적인 스타일 문법은 사용하되, 특정 캐릭터·마스코트 디자인은 새로 정의한다.

### 이 설계에서 제외 (별도 프로젝트)
- **Oracle Cloud 프리티어 VM 이전** — 정적 사이트 빌드 결과물은 호스팅과 무관하므로 레이아웃 완료 후 별도 진행
- **썸네일 일러스트 26장 신규 제작** — 파일 교체만으로 반영되므로 레이아웃과 분리

두 작업 모두 이 설계가 끝난 뒤 각각 별도 스펙으로 진행한다. 동시 진행 시 문제 원인 격리가 불가능해진다.

## 3. 파일 처리

### 삭제

| 대상 | 이유 |
|------|------|
| `Gemfile`의 `jekyll-theme-chirpy` 의존성 | 디자인의 실체 |
| `assets/lib/` | Chirpy 전용 벤더 라이브러리 (fontawesome, Lato, dayjs, clipboard) |
| `_includes/metadata-hook.html` | Chirpy 훅 |
| `_data/contact.yml`, `_data/share.yml` | Chirpy 사이드바·공유버튼 설정 |
| `_config.yml`의 테마 블록 | theme, avatar, toc, paginate 등 |
| `_site/`, `.jekyll-cache/` | 빌드 산출물 |
| `tools/` | Chirpy 번들 스크립트 |
| `_tabs/archives.md`, `_tabs/categories.md`, `_tabs/tags.md` | front matter만 있고 본문 없음. Chirpy 레이아웃 의존이라 gem 제거 시 빈 껍데기가 됨 |
| `_config.yml`의 `pwa` 블록 | PWA 폐기 (9장 참조) |
| `_includes/metadata-hook.html` | **삭제 전 네이버 인증 메타태그를 `default.html`로 이전할 것** (아래 참조) |

### 이전 필수 항목

`_includes/metadata-hook.html`은 삭제하지만 내용물을 그대로 버리면 안 된다.

| 내용 | 처리 |
|------|------|
| `<meta name="naver-site-verification" content="3acf5e196b4f28c3006c9421b91b997f79ef39aa" />` | **`_layouts/default.html`의 `<head>`로 이전.** 유실 시 네이버 서치어드바이저 등록이 풀린다 |
| hero 이미지 숨김 CSS | 폐기. Chirpy 마크업 전용 해킹이라 새 레이아웃에서 불필요 |
| 서비스워커 갱신 스크립트 | 폐기. PWA와 함께 제거 |

### 보존되는 URL 구조

`_config.yml`의 아래 설정은 **변경하지 않는다.** 기존 26편의 URL과 외부 링크·SEO가 여기 걸려 있다.

- `permalink: /posts/:title/` (posts defaults)
- `jekyll-archives`의 `tag: /tags/:name/`

`jekyll-archives`는 Chirpy와 독립된 gem이므로 **제거하지 않고** 태그 페이지 생성에 그대로 사용한다. `category` 항목은 카테고리 전용 페이지를 만들지 않으므로 `enabled`에서 뺀다.

### 보존 (수정하지 않음)

`_posts/` 26편, `assets/img/posts/`, `assets/img/favicons/`, `.claude/` 전체, `.env`, `CLAUDE.md`, `.github/`, `_tabs/about.md`

### 신규 작성

```
_layouts/     default, home, post, tag, page
_sass/        _tokens.scss, 컴포넌트별 파티셜
_includes/    header, hero, card-featured, list-item, tag-sidebar, footer
assets/css/   main.scss
index.html    재작성
```

## 4. 페이지 구성

| 경로 | 레이아웃 | 내용 |
|------|---------|------|
| `/` | `home` | 히어로 + 최근 올라온 글 3열 + 전체 게시글 리스트 + 태그 사이드바 |
| `/posts/:slug` | `post` | 포스트 본문 |
| `/tags/:tag` | `tag` | 해당 태그 글 리스트, 사이드바 동일 |
| `/about` | `page` | 기존 `about.md` 본문 |

상단 네비게이션은 `Tech Log`(홈) / `About` + 검색 아이콘.

## 5. 컴포넌트

| 파일 | 역할 |
|------|------|
| `header.html` | 로고 좌측, 네비 + 검색 아이콘 우측 |
| `hero.html` | 다크 네이비 풀블리드. 그리드 패턴 배경 + 타이틀 + 태그라인 + 장식 글리프 |
| `card-featured.html` | 3열 카드. 3:2 썸네일 / 제목 / 날짜 · 카테고리 |
| `list-item.html` | 좌 썸네일(고정폭) + 우 제목 · 요약 · 날짜 |
| `tag-sidebar.html` | 태그 pill 클라우드 + "태그 더보기" 토글 |
| `footer.html` | 저작권 · 소셜 링크 |

### 히어로 세부
- 타이틀: `_config.yml`의 `title`
- 태그라인: `_config.yml`의 `description`
- 장식 글리프: `{ }`, `</>`, `[ ]`, `;` 등 범용 코드 기호를 인라인 SVG로 직접 작성. 외부 아이콘 자산 미사용

### 피처드 3편 선정
최신 3편 자동 선택. front matter 플래그 방식은 관리 부담 대비 이득이 없어 채택하지 않음.

## 6. 색 · 타이포 토큰

`_sass/_tokens.scss`에 CSS 변수로 정의하고 모든 컴포넌트가 이를 참조한다. 하드코딩된 색상값 금지.

### 라이트

| 토큰 | 값 | 쓰임 |
|------|-----|------|
| `--color-navy` | `#0B1F3A` | 히어로 배경, 본문 제목 |
| `--color-accent` | `#2E6FBF` | 카테고리 라벨, 링크, 활성 상태 |
| `--color-text` | `#1A2635` | 본문 |
| `--color-text-sub` | `#5A6B82` | 날짜, 요약, 보조 텍스트 |
| `--color-border` | `#E8EDF3` | 구분선, 카드 테두리 |
| `--color-bg` | `#FFFFFF` | 페이지 배경 |
| `--color-bg-soft` | `#F1F5F9` | 태그 pill, 썸네일 폴백 |

다크 세트는 구현 6단계에서 동일 토큰명에 값만 재정의한다. 히어로가 이미 네이비이므로 전환이 자연스럽다.

### 폰트
- 본문·제목: **Pretendard** (SIL OFL, 자체 호스팅)
- 코드: **JetBrains Mono**

기존 Lato / Source Sans Pro는 한글 글리프가 없어 시스템 폰트로 폴백되고 있었다.

### 타입 스케일

| 요소 | 크기 / 굵기 |
|------|------------|
| 히어로 타이틀 | 48px / 800 |
| 섹션 헤딩 | 14px / 600, 서브톤 |
| 카드 제목 | 18px / 700 |
| 리스트 제목 | 20px / 700 |
| 요약 | 14px / 400 |
| 날짜 · 메타 | 13px |
| 본문 | 17px, line-height 1.8 |

## 7. 데이터 매핑

포스트 front matter는 수정하지 않는다. 새 레이아웃이 기존 구조를 그대로 읽는다.

| front matter | 화면 위치 |
|--------------|-----------|
| `title` | 카드 제목 / 리스트 제목 / 포스트 h1 |
| `date` | `2026. 7. 16.` 포맷, 카드·리스트 하단 |
| `categories` | 리스트 항목 상단 라벨 (`백엔드 › 데이터베이스`) |
| `tags` | 우측 사이드바 클라우드 + 포스트 하단 |
| `image.path` | 3:2 썸네일, `object-fit: cover` |
| `image.alt` | `alt` 속성 |
| 본문 앞부분 | 리스트 요약 (`strip_html \| truncate: 90`) |

### 카테고리 처리
현행 중첩 구조(`[백엔드, 데이터베이스]`, `[책 Log, 책제목, 목차명]`)를 유지한다. 카테고리 전용 필터 UI는 만들지 않고, 리스트 항목의 라벨로만 노출한다. 탐색은 우측 태그 사이드바가 담당한다.

### 썸네일
- 고정 비율 **3:2** (신규 제작 시 1200×800)
- 기존 이미지 24편은 `object-fit: cover`로 크롭 적용. 신규 썸네일로 교체되면 크롭 이슈 소멸
- `image` 없는 2편(`2026-05-25-github-blog-setup`, `2026-06-03-soma17-dev-log`)은 `--color-bg-soft` 배경에 제목만 얹은 폴백 카드

## 8. 다크모드

유지한다. 레퍼런스 화면에는 없는 요소지만 기존 Chirpy에 있던 기능이라 회귀를 피한다. 토큰이 CSS 변수이므로 다크 세트 정의 + 네비 토글로 구현한다.

## 8-2. 기존 기능 존속

Chirpy gem이 제공하던 기능이라 테마 제거와 함께 사라진다. 아래는 직접 재구현하여 유지한다.

| 기능 | 현재 설정 | 재구현 방식 |
|------|----------|------------|
| giscus 댓글 | `repo: kth4778/kth4778.github.io`, `repo_id: R_kgDOSna5DQ`, `category_id: DIC_kwDOSna5Dc4C90ca`, `mapping: pathname`, `lang: ko` | `post` 레이아웃에 giscus 스크립트 직접 삽입. `mapping: pathname` + permalink 유지로 기존 댓글 스레드 보존 |
| Google Analytics | `G-VGD2N6XR7K` | `default` 레이아웃에 gtag 스니펫. `JEKYLL_ENV=production`일 때만 출력 |
| GoatCounter 조회수 | `id: kth4778` | 카운트 스크립트 + 포스트 조회수 표시 |
| 포스트 TOC | `toc: true` | 본문 h2/h3를 파싱해 사이드 목차 생성 |

`_config.yml`의 `comments`, `analytics`, `pageviews`, `toc` 블록은 **삭제하지 않는다.** 새 레이아웃이 동일한 키를 읽는다.

### 폐기

**PWA** (`pwa.enabled: true`). 기술블로그에 설치형 앱 수요가 없어 서비스워커·manifest·오프라인 캐시를 재구현하지 않는다. `_config.yml`의 `pwa` 블록과 `metadata-hook.html`의 서비스워커 갱신 스크립트를 함께 제거한다.

## 9. 검색

레퍼런스 화면의 돋보기 아이콘을 살린다. Jekyll에는 검색 기능이 없고 Chirpy 구현은 gem과 함께 사라지므로 직접 만든다.

- 빌드 시 `search.json` 생성 (title, categories, tags, url, 본문 일부)
- 클라이언트 사이드 필터링
- 외부 서비스 의존 없음

## 10. 구현 순서

각 단계 종료 시점에 `bundle exec jekyll build`가 통과해야 다음 단계로 진행한다.

| 단계 | 작업 | 완료 기준 |
|------|------|----------|
| 1 | Chirpy 제거 (3장의 삭제 목록). 네이버 메타태그 먼저 확보 | 빌드 실패는 예상된 상태 |
| 2 | `_tokens.scss` + `default.html` + `main.scss` + 네이버 메타 이전 | 빈 페이지라도 빌드 통과 |
| 3 | 홈 조립 — hero → card-featured → list-item → tag-sidebar | 26편이 목록에 렌더 |
| 4 | `post` 레이아웃 + 본문 타이포 + 코드블록 + TOC | 26편 본문 정상 렌더 |
| 5 | `/tags/:tag` 페이지 (jekyll-archives 활용) | 태그 클릭 시 필터 동작 |
| 6 | 다크모드 토큰 + 토글 | 양쪽 모드 정상 |
| 7 | giscus 댓글 + GA + GoatCounter 조회수 | 댓글 로드, 프로덕션 빌드에서만 추적 스크립트 출력 |
| 8 | `search.json` + 클라이언트 필터링 | 검색 동작 |

2단계에서 `ui-ux-pro-max` 스킬로 팔레트·폰트 페어링·컴포넌트 스펙을 확정한다.

## 11. 검증

- `bundle exec jekyll build` 무경고 통과
- 26편 전부 렌더 확인 (누락·깨짐 없음)
- `html-proofer`로 깨진 링크·이미지 검사 (Gemfile에 이미 포함)
- 모바일 / 데스크톱 반응형
- 다크모드 라이트·다크 양쪽
- 검색 동작
- 브라우저로 직접 띄워 스크린샷 확인

### 회귀 검사 (유실되기 쉬운 항목)

| 항목 | 확인 방법 |
|------|----------|
| 26편 URL 불변 | 빌드 후 `_site/posts/` 디렉터리명이 기존과 동일 |
| 네이버 인증 메타 | `_site` 내 `naver-site-verification` 문자열 검색 |
| giscus 댓글 | 기존 댓글이 달린 포스트에서 스레드 로드 확인 |
| GA / GoatCounter | `JEKYLL_ENV=production` 빌드 결과에만 스크립트 존재 |
| 태그 URL | `/tags/:name/` 형식 유지 |

## 12. 롤백

develop 브랜치에서만 작업한다. main은 건드리지 않으므로 실패 시 브랜치 폐기로 복구된다. 발행은 기존대로 develop → main PR 머지.

## 13. 후속 작업

1. 썸네일 일러스트 26장 신규 제작 (1200×800, 플랫 벡터 스타일)
2. Oracle Cloud 프리티어 VM 이전 (도메인, 배포 파이프라인, HTTPS)
3. `.claude/memory/블로그현황.md`의 이미지 스타일 히스토리 갱신
4. `CLAUDE.md`의 카테고리 규칙에서 "사이드바 개수 최소화" 근거 문구 수정 — 사이드바가 태그 전용으로 바뀌었으므로 기존 근거가 성립하지 않음
