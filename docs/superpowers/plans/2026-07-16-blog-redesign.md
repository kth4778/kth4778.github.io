# 블로그 디자인 전면 교체 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Chirpy 테마를 제거하고 모던 테크블로그 레이아웃(히어로 + 피처드 3열 + 좌썸네일 리스트 + 태그 사이드바)을 자체 구축한다. 기존 포스트 26편은 파일 수정 없이 이식한다.

**Architecture:** Jekyll 4 위에 자체 테마를 직접 작성한다. `_sass/_tokens.scss`의 CSS 변수를 단일 진실 공급원으로 삼고 모든 컴포넌트가 이를 참조한다. 포스트 front matter는 손대지 않고 새 레이아웃이 기존 구조(`title`/`date`/`categories`/`tags`/`image.path`/`image.alt`)를 읽는다.

**Tech Stack:** Jekyll 4.4, Sass, Liquid, Pretendard, JetBrains Mono, jekyll-archives, jekyll-seo-tag, jekyll-sitemap, jekyll-paginate

**Spec:** `docs/superpowers/specs/2026-07-16-blog-redesign-design.md`

## Global Constraints

- **`_posts/` 26편은 한 글자도 수정하지 않는다.** front matter 포함.
- **`permalink: /posts/:title/` 불변.** 기존 URL 26개와 SEO가 여기 걸려 있다.
- **태그 URL `/tags/:name/` 불변.**
- **네이버 인증 메타태그 `content="3acf5e196b4f28c3006c9421b91b997f79ef39aa"` 유실 금지.**
- **색상 하드코딩 금지.** 모든 색은 `_sass/_tokens.scss`의 CSS 변수를 참조한다.
- **썸네일 비율 3:2 고정** (`object-fit: cover`).
- 브랜치: `develop`. `main`은 건드리지 않는다.
- 카카오페이의 로고·일러스트·브랜드 옐로·CSS/HTML 소스는 사용하지 않는다. 레이아웃 구조만 참고한다.
- 각 Task 종료 시 `bundle exec jekyll build`가 통과해야 한다 (Task 1 제외).
- 커밋 메시지는 한국어, `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>` 로 끝낸다.

## 검증 방식에 관하여

Jekyll 테마에는 단위 테스트를 붙일 대상이 없다. 대신 각 Task는 **빌드 통과 + 산출물 assertion**으로 검증한다. `_site/`에 특정 문자열·파일이 존재하는지 grep으로 확인하는 방식이며, 이것이 이 프로젝트에서 "실패하는 테스트를 먼저 본다"의 실질적 등가물이다. 각 Task는 검증을 먼저 실행해 실패를 확인한 뒤 구현한다.

---

### Task 1: Chirpy 제거 및 gem 의존성 명시화

Chirpy는 `jekyll-archives`, `jekyll-paginate`, `jekyll-seo-tag`, `jekyll-sitemap`을 자신의 gemspec에서 선언해 딸려오게 하고 있다. Chirpy를 빼면 이 넷도 사라지므로 직접 선언해야 한다. 또한 테마 없이는 `_config.yml`에 `plugins:` 목록이 필요하다.

**Files:**
- Modify: `Gemfile`
- Modify: `_config.yml`
- Delete: `assets/lib/`, `_data/contact.yml`, `_data/share.yml`, `tools/`, `_tabs/archives.md`, `_tabs/categories.md`, `_tabs/tags.md`, `_site/`, `.jekyll-cache/`
- Preserve first: `_includes/metadata-hook.html` (네이버 메타태그 확보 후 Task 2에서 삭제)

**Interfaces:**
- Produces: `plugins:` 목록이 선언된 `_config.yml`, Chirpy 없는 `Gemfile`

- [ ] **Step 1: 네이버 메타태그를 먼저 확보**

```bash
grep naver-site-verification _includes/metadata-hook.html
```

Expected: `<meta name="naver-site-verification" content="3acf5e196b4f28c3006c9421b91b997f79ef39aa" />`

이 값을 Task 2에서 쓴다. `metadata-hook.html`은 Task 2에서 삭제한다.

- [ ] **Step 2: Gemfile 교체**

`Gemfile` 전체를 아래로 바꾼다.

```ruby
# frozen_string_literal: true

source "https://rubygems.org"

gem "jekyll", "~> 4.3"

# Chirpy가 gemspec에서 선언해주던 플러그인들 — 직접 선언한다
gem "jekyll-archives", "~> 2.2"
gem "jekyll-seo-tag", "~> 2.8"
gem "jekyll-sitemap", "~> 1.4"

gem "html-proofer", "~> 5.0", group: :test

platforms :windows, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

gem "wdm", "~> 0.2.0", :platforms => [:windows]
```

- [ ] **Step 3: `_config.yml`에서 theme 제거하고 plugins 선언**

`theme: jekyll-theme-chirpy` 줄을 삭제하고, 같은 자리에 아래를 넣는다.

```yaml
# 테마 없이 직접 빌드한다. Chirpy gemspec이 대신 선언해주던 플러그인 목록.
plugins:
  - jekyll-archives
  - jekyll-seo-tag
  - jekyll-sitemap
```

`jekyll-paginate`는 뺀다. 26편 전부를 한 페이지에 렌더하므로 페이지네이션이 필요없다 (YAGNI). 글이 100편을 넘어가면 그때 다시 넣는다.

- [ ] **Step 4: `_config.yml`에서 Chirpy 전용 설정 제거**

아래 키를 삭제한다. `comments`, `analytics`, `pageviews`, `toc`는 **삭제하지 않는다** (Task 6·9에서 읽는다).

- `avatar:`
- `cdn:`
- `theme_mode:`
- `assets:` 블록 전체 (self_host)
- `pwa:` 블록 전체
- `paginate: 10` (jekyll-paginate를 뺐으므로 죽은 설정)

`social_preview_image: /assets/img/avatar.png`는 **남긴다.** `jekyll-seo-tag`가 `og:image`에 쓰며 Chirpy와 무관하다.

`jekyll-archives` 블록은 `category`를 빼고 아래로 수정한다.

```yaml
jekyll-archives:
  enabled: [tags]
  layouts:
    tag: tag
  permalinks:
    tag: /tags/:name/
```

- [ ] **Step 5: Chirpy 잔재 파일 삭제**

```bash
rm -rf assets/lib _data/contact.yml _data/share.yml tools _site .jekyll-cache
rm -f _tabs/archives.md _tabs/categories.md _tabs/tags.md
```

`_tabs/about.md`는 남긴다.

- [ ] **Step 6: 번들 재설치 후 빌드 — 실패를 확인**

```bash
bundle install
bundle exec jekyll build
```

Expected: FAIL. 레이아웃이 없으므로 `home`/`post`/`page` 레이아웃을 찾지 못한다는 경고 또는 빈 출력. 이 실패는 정상이며 Task 2에서 해소된다.

- [ ] **Step 7: 커밋**

```bash
git add -A
git commit -F - <<'EOF'
chore(theme): Chirpy 테마 제거 및 gem 의존성 명시화

Chirpy가 gemspec에서 선언하던 jekyll-archives·paginate·seo-tag·sitemap을
Gemfile에 직접 선언하고 _config.yml에 plugins 목록을 추가했다.

이 시점에서 빌드는 의도적으로 깨진 상태다. Task 2에서 복구된다.

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>
EOF
```

---

### Task 2: 디자인 토큰 + default 레이아웃 + 네이버 메타 이전

빌드를 되살리는 단계다. 이 Task 완료 시점에 빈 페이지라도 빌드가 통과해야 한다.

**Files:**
- Create: `_sass/_tokens.scss`, `_sass/_base.scss`, `_sass/_fonts.scss`, `assets/css/main.scss`, `_layouts/default.html`, `_layouts/page.html`
- Create: `assets/fonts/` (woff2 파일)
- Delete: `_includes/metadata-hook.html`

**Interfaces:**
- Produces: CSS 변수 `--color-navy`, `--color-accent`, `--color-text`, `--color-text-sub`, `--color-border`, `--color-bg`, `--color-bg-soft`, `--font-sans`, `--font-mono`. 이후 모든 Task가 이 이름을 참조한다.
- Produces: `_layouts/default.html` — 이후 모든 레이아웃이 `layout: default`로 상속한다.

- [ ] **Step 1: ui-ux-pro-max로 팔레트·폰트 스펙 확정**

`ui-ux-pro-max` 스킬을 호출해 딥 네이비 계열 팔레트와 Pretendard + JetBrains Mono 페어링의 세부 스펙(대비비, 다크 세트 후보)을 받는다. 스펙 6장의 토큰 값이 기준이며, 접근성 대비(WCAG AA)만 보정한다.

- [ ] **Step 2: `_sass/_tokens.scss` 작성**

```scss
:root {
  --color-navy: #0B1F3A;
  --color-accent: #2E6FBF;
  --color-text: #1A2635;
  --color-text-sub: #5A6B82;
  --color-border: #E8EDF3;
  --color-bg: #FFFFFF;
  --color-bg-soft: #F1F5F9;

  --font-sans: "Pretendard", -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  --font-mono: "JetBrains Mono", "D2Coding", Consolas, monospace;

  --maxw-page: 1200px;
  --gap-card: 24px;
  --radius-card: 8px;
}
```

다크 세트는 Task 8에서 추가한다.

- [ ] **Step 2-B: 폰트 파일 확보 및 `_sass/_fonts.scss` 작성**

토큰이 `"Pretendard"`를 참조하므로 폰트를 실제로 로드해야 한다. 이 단계를 빠뜨리면 시스템 폰트로 조용히 폴백된다 — Chirpy가 Lato·Source Sans Pro로 겪던 문제와 같다.

Pretendard(SIL OFL)와 JetBrains Mono(SIL OFL) 모두 자체 호스팅이 허용된다. 서브셋 woff2를 받아 `assets/fonts/`에 넣는다.

```bash
mkdir -p assets/fonts
```

받을 파일 (Pretendard는 dynamic-subset 아닌 통짜 woff2, 400/700/800 세 굵기만):

| 파일 | 출처 |
|------|------|
| `Pretendard-Regular.woff2` | https://github.com/orioncactus/pretendard/releases |
| `Pretendard-Bold.woff2` | 같음 |
| `Pretendard-ExtraBold.woff2` | 같음 |
| `JetBrainsMono-Regular.woff2` | https://github.com/JetBrains/JetBrainsMono/releases |

`_sass/_fonts.scss`:

```scss
@font-face {
  font-family: "Pretendard";
  font-weight: 400;
  font-display: swap;
  src: url("/assets/fonts/Pretendard-Regular.woff2") format("woff2");
}

@font-face {
  font-family: "Pretendard";
  font-weight: 700;
  font-display: swap;
  src: url("/assets/fonts/Pretendard-Bold.woff2") format("woff2");
}

@font-face {
  font-family: "Pretendard";
  font-weight: 800;
  font-display: swap;
  src: url("/assets/fonts/Pretendard-ExtraBold.woff2") format("woff2");
}

@font-face {
  font-family: "JetBrains Mono";
  font-weight: 400;
  font-display: swap;
  src: url("/assets/fonts/JetBrainsMono-Regular.woff2") format("woff2");
}
```

`font-display: swap`을 쓰는 이유는 폰트 로드 중에도 텍스트가 보이게 하기 위함이다.

- [ ] **Step 2-C: 폰트 로드 검증**

```bash
ls assets/fonts/*.woff2 | wc -l
```

Expected: `4`

빌드 후 브라우저 개발자도구 Network 탭에서 woff2 4개가 200으로 로드되는지, `document.fonts.check('16px Pretendard')`가 `true`인지 확인한다. `false`면 경로나 `@font-face`가 잘못된 것이다.

- [ ] **Step 3: `_sass/_base.scss` 작성**

```scss
*, *::before, *::after { box-sizing: border-box; }

body {
  margin: 0;
  background: var(--color-bg);
  color: var(--color-text);
  font-family: var(--font-sans);
  font-size: 17px;
  line-height: 1.8;
  -webkit-font-smoothing: antialiased;
}

a { color: var(--color-accent); text-decoration: none; }
a:hover { text-decoration: underline; }

img { max-width: 100%; height: auto; }

code, pre { font-family: var(--font-mono); }

.wrap {
  max-width: var(--maxw-page);
  margin: 0 auto;
  padding: 0 24px;
}
```

- [ ] **Step 4: `assets/css/main.scss` 작성**

앞의 빈 front matter(`---` 두 줄)가 있어야 Jekyll이 Sass로 처리한다.

```scss
---
---

@import "fonts";
@import "tokens";
@import "base";
```

- [ ] **Step 5: `_layouts/default.html` 작성 — 네이버 메타 포함**

```html
<!DOCTYPE html>
<html lang="{{ site.lang | default: 'ko' }}">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">

  <!-- 네이버 서치어드바이저 등록. 삭제 금지. -->
  <meta name="naver-site-verification" content="3acf5e196b4f28c3006c9421b91b997f79ef39aa" />

  {% seo %}
  <link rel="stylesheet" href="{{ '/assets/css/main.css' | relative_url }}">
  <link rel="icon" href="{{ '/assets/img/favicons/favicon.ico' | relative_url }}">
</head>
<body>
  <main>
    {{ content }}
  </main>
</body>
</html>
```

헤더·푸터는 Task 3에서 끼운다.

- [ ] **Step 6: `_layouts/page.html` 작성**

`_tabs/about.md`가 `layout: page`를 쓴다 (`_config.yml`의 tabs defaults).

```html
---
layout: default
---

<article class="wrap">
  <h1>{{ page.title }}</h1>
  {{ content }}
</article>
```

- [ ] **Step 7: `metadata-hook.html` 삭제**

네이버 메타는 Step 5에서 이전했다. 나머지 두 내용(hero 숨김 CSS, 서비스워커 스크립트)은 스펙 3장에 따라 폐기한다.

```bash
rm _includes/metadata-hook.html
```

- [ ] **Step 8: 빌드 및 네이버 메타 검증**

```bash
bundle exec jekyll build
grep -r "naver-site-verification" _site/ | head -3
```

Expected: 빌드 통과. grep이 `3acf5e196b4f28c3006c9421b91b997f79ef39aa`를 포함한 줄을 출력.

`home`/`post` 레이아웃이 아직 없어 관련 경고는 남아 있을 수 있다. Task 3·5에서 해소된다.

- [ ] **Step 9: 커밋**

```bash
git add -A
git commit -F - <<'EOF'
feat(theme): 디자인 토큰과 default 레이아웃 추가

CSS 변수 기반 토큰을 단일 진실 공급원으로 정의했다.
metadata-hook.html의 네이버 인증 메타태그를 default.html로 이전했다.

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>
EOF
```

---

### Task 3: 헤더 · 히어로 · 푸터

**Files:**
- Create: `_includes/header.html`, `_includes/hero.html`, `_includes/footer.html`, `_sass/_header.scss`, `_sass/_hero.scss`, `_sass/_footer.scss`
- Modify: `_layouts/default.html`, `assets/css/main.scss`

**Interfaces:**
- Consumes: Task 2의 토큰 변수
- Produces: `_includes/hero.html` — `home` 레이아웃이 Task 4에서 호출한다

- [ ] **Step 1: `_includes/header.html`**

```html
<header class="site-header">
  <div class="wrap site-header__inner">
    <a class="site-header__logo" href="{{ '/' | relative_url }}">{{ site.title }}</a>
    <nav class="site-header__nav">
      <a href="{{ '/' | relative_url }}">Tech Log</a>
      <a href="{{ '/about/' | relative_url }}">About</a>
      <button class="site-header__search" type="button" aria-label="검색" data-search-toggle>
        <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <circle cx="11" cy="11" r="7"/><path d="M20 20l-3.5-3.5"/>
        </svg>
      </button>
    </nav>
  </div>
</header>
```

검색 버튼은 Task 10에서 동작을 붙인다. 지금은 자리만 잡는다.

- [ ] **Step 2: `_includes/hero.html`**

장식 글리프는 범용 코드 기호를 인라인 SVG/텍스트로 직접 그린다. 외부 아이콘 자산을 쓰지 않는다.

```html
<section class="hero">
  <div class="hero__grid" aria-hidden="true"></div>
  <div class="wrap hero__inner">
    <h1 class="hero__title">{{ site.title }}</h1>
    <p class="hero__tagline">{{ site.description }}</p>
  </div>
  <div class="hero__glyphs" aria-hidden="true">
    <span>&lt;/&gt;</span><span>{ }</span><span>[ ]</span><span>;</span>
  </div>
</section>
```

- [ ] **Step 3: `_sass/_hero.scss`**

```scss
.hero {
  position: relative;
  background: var(--color-navy);
  color: #fff;
  padding: 80px 0 96px;
  overflow: hidden;
}

.hero__grid {
  position: absolute;
  inset: 0;
  background-image:
    linear-gradient(rgba(255,255,255,.06) 1px, transparent 1px),
    linear-gradient(90deg, rgba(255,255,255,.06) 1px, transparent 1px);
  background-size: 100px 100px;
}

.hero__inner { position: relative; }

.hero__title {
  margin: 0;
  font-size: 48px;
  font-weight: 800;
  letter-spacing: -0.5px;
}

.hero__tagline {
  margin: 12px 0 0;
  font-size: 16px;
  color: rgba(255,255,255,.7);
}

.hero__glyphs {
  position: absolute;
  top: 50%;
  right: 8%;
  transform: translateY(-50%);
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 40px;
  font-family: var(--font-mono);
  font-size: 22px;
  color: rgba(255,255,255,.35);
}

@media (max-width: 900px) {
  .hero { padding: 56px 0 64px; }
  .hero__title { font-size: 32px; }
  .hero__glyphs { display: none; }
}
```

- [ ] **Step 4: `_includes/footer.html`**

```html
<footer class="site-footer">
  <div class="wrap">
    <p>&copy; {{ 'now' | date: "%Y" }} {{ site.social.name }}</p>
    <p class="site-footer__links">
      {% for link in site.social.links %}<a href="{{ link }}">{{ link | split: '//' | last | split: '/' | first }}</a>{% endfor %}
    </p>
  </div>
</footer>
```

- [ ] **Step 5: `_sass/_header.scss` 작성**

```scss
.site-header {
  border-bottom: 1px solid var(--color-border);
  background: var(--color-bg);
}

.site-header__inner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 64px;
}

.site-header__logo {
  font-size: 16px;
  font-weight: 800;
  letter-spacing: -0.3px;
  color: var(--color-text);
}
.site-header__logo:hover { text-decoration: none; }

.site-header__nav {
  display: flex;
  align-items: center;
  gap: 24px;
}

.site-header__nav > a {
  font-size: 14px;
  font-weight: 600;
  color: var(--color-text-sub);
}
.site-header__nav > a:hover { color: var(--color-accent); text-decoration: none; }

.site-header__search,
.site-header__theme {
  display: flex;
  align-items: center;
  padding: 6px;
  border: 0;
  background: none;
  color: var(--color-text-sub);
  cursor: pointer;
}
.site-header__search:hover,
.site-header__theme:hover { color: var(--color-accent); }

@media (max-width: 600px) {
  .site-header__nav { gap: 14px; }
  .site-header__logo { font-size: 14px; }
}
```

- [ ] **Step 5-B: `_sass/_footer.scss` 작성**

```scss
.site-footer {
  margin-top: 80px;
  padding: 32px 0 48px;
  border-top: 1px solid var(--color-border);
  font-size: 13px;
  color: var(--color-text-sub);

  p { margin: 0; }
}

.site-footer__links {
  display: flex;
  gap: 16px;
  margin-top: 8px;
}
```

- [ ] **Step 5-C: 공용 `.section-label` 정의**

Task 4·7의 "최근 올라온 글" / "전체 게시글" 헤딩이 이 클래스를 쓴다. `_sass/_base.scss` 하단에 추가한다.

```scss
.section-label {
  margin: 0 0 20px;
  font-size: 14px;
  font-weight: 600;
  color: var(--color-text-sub);
}
```

- [ ] **Step 5-D: `main.scss`에 등록**

```scss
---
---

@import "fonts";
@import "tokens";
@import "base";
@import "header";
@import "hero";
@import "footer";
```

- [ ] **Step 6: `default.html`에 헤더·푸터 끼우기**

`<body>` 내부를 아래로 바꾼다.

```html
<body>
  {% include header.html %}
  <main>
    {{ content }}
  </main>
  {% include footer.html %}
</body>
```

- [ ] **Step 7: 빌드 및 검증**

```bash
bundle exec jekyll build
grep -c "site-header__logo" _site/about/index.html
```

Expected: 빌드 통과, grep 결과 `1`

- [ ] **Step 8: 커밋**

```bash
git add -A
git commit -F - <<'EOF'
feat(theme): 헤더·히어로·푸터 컴포넌트 추가

히어로 장식 글리프는 범용 코드 기호를 직접 그렸다.

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>
EOF
```

---

### Task 4: 홈 — 피처드 3열 + 리스트 + 태그 사이드바

26편이 처음으로 화면에 나오는 단계다.

**Files:**
- Create: `_layouts/home.html`, `_includes/card-featured.html`, `_includes/list-item.html`, `_includes/tag-sidebar.html`, `_sass/_card.scss`, `_sass/_list.scss`, `_sass/_sidebar.scss`
- Modify: `assets/css/main.scss`

**Interfaces:**
- Consumes: Task 2 토큰, Task 3 `hero.html`
- Produces: `.thumb` 클래스 (3:2 비율 규칙) — Task 7의 태그 페이지가 재사용한다

- [ ] **Step 1: 검증부터 실행해 실패 확인**

```bash
bundle exec jekyll build && grep -c "card-featured" _site/index.html
```

Expected: FAIL 또는 `0`. `home` 레이아웃이 아직 없다.

- [ ] **Step 2: `_includes/card-featured.html`**

`include.post`로 포스트를 받는다.

```html
<article class="card">
  <a href="{{ include.post.url | relative_url }}">
    <div class="thumb">
      {% if include.post.image.path %}
        <img src="{{ include.post.image.path | relative_url }}" alt="{{ include.post.image.alt | default: include.post.title }}" loading="lazy">
      {% else %}
        <div class="thumb__fallback"><span>{{ include.post.title | truncate: 40 }}</span></div>
      {% endif %}
    </div>
    <h3 class="card__title">{{ include.post.title }}</h3>
  </a>
  <p class="card__meta">
    <time datetime="{{ include.post.date | date_to_xmlschema }}">{{ include.post.date | date: "%Y. %-m. %-d." }}</time>
    <span class="card__cat">{{ include.post.categories | join: " › " }}</span>
  </p>
</article>
```

- [ ] **Step 3: `_includes/list-item.html`**

요약은 본문 앞부분을 자른다. front matter에 `description`이 없으므로 이 방식이 유일하다.

```html
<article class="list-item">
  <a class="list-item__thumb thumb" href="{{ include.post.url | relative_url }}">
    {% if include.post.image.path %}
      <img src="{{ include.post.image.path | relative_url }}" alt="{{ include.post.image.alt | default: include.post.title }}" loading="lazy">
    {% else %}
      <div class="thumb__fallback"><span>{{ include.post.title | truncate: 30 }}</span></div>
    {% endif %}
  </a>
  <div class="list-item__body">
    <p class="list-item__cat">{{ include.post.categories | join: " › " }}</p>
    <h2 class="list-item__title"><a href="{{ include.post.url | relative_url }}">{{ include.post.title }}</a></h2>
    <p class="list-item__excerpt">{{ include.post.content | strip_html | normalize_whitespace | truncate: 90 }}</p>
    <time class="list-item__date" datetime="{{ include.post.date | date_to_xmlschema }}">{{ include.post.date | date: "%Y. %-m. %-d." }}</time>
  </div>
</article>
```

- [ ] **Step 4: `_includes/tag-sidebar.html`**

초기 12개만 노출하고 나머지는 토글로 편다.

```html
<aside class="tag-sidebar">
  <h2 class="tag-sidebar__title">Tag</h2>
  <ul class="tag-sidebar__list" data-tag-list>
    {% assign tags = site.tags | sort %}
    {% for tag in tags %}
      <li{% if forloop.index > 12 %} class="is-hidden"{% endif %}>
        <a class="tag-pill" href="{{ '/tags/' | append: tag[0] | slugify | append: '/' | relative_url }}">{{ tag[0] }}</a>
      </li>
    {% endfor %}
  </ul>
  {% if tags.size > 12 %}
    <button class="tag-sidebar__more" type="button" data-tag-more>태그 더보기 ↓</button>
  {% endif %}
</aside>
```

- [ ] **Step 5: `_layouts/home.html`**

```html
---
layout: default
---

{% include hero.html %}

<div class="wrap home">
  <section class="home__featured">
    <h2 class="section-label">최근 올라온 글</h2>
    <div class="card-grid">
      {% for post in site.posts limit: 3 %}
        {% include card-featured.html post=post %}
      {% endfor %}
    </div>
  </section>

  <div class="home__main">
    <section class="home__list">
      <h2 class="section-label">전체 게시글</h2>
      {% for post in site.posts %}
        {% include list-item.html post=post %}
      {% endfor %}
    </section>
    {% include tag-sidebar.html %}
  </div>
</div>
```

- [ ] **Step 6: `_sass/_card.scss` — 3:2 썸네일 규칙**

`.thumb`는 이후 Task에서도 재사용하므로 여기서 확정한다.

```scss
.thumb {
  display: block;
  aspect-ratio: 3 / 2;
  border-radius: var(--radius-card);
  overflow: hidden;
  background: var(--color-bg-soft);

  img { width: 100%; height: 100%; object-fit: cover; display: block; }
}

.thumb__fallback {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
  padding: 16px;
  text-align: center;
  color: var(--color-text-sub);
  font-size: 14px;
  font-weight: 700;
}

.card-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: var(--gap-card);
}

.card__title { margin: 12px 0 0; font-size: 18px; font-weight: 700; line-height: 1.45; color: var(--color-text); }
.card__meta { margin: 8px 0 0; font-size: 13px; color: var(--color-text-sub); display: flex; gap: 8px; }
.card__cat { color: var(--color-accent); font-weight: 700; }

@media (max-width: 900px) {
  .card-grid { grid-template-columns: 1fr; }
}
```

- [ ] **Step 7: `_sass/_list.scss`**

```scss
.home { padding-top: 56px; }
.home__featured { margin-bottom: 80px; }

.home__main {
  display: grid;
  grid-template-columns: minmax(0, 1fr) 240px;
  gap: 48px;
  align-items: start;
}

.list-item {
  display: grid;
  grid-template-columns: 280px minmax(0, 1fr);
  gap: 24px;
  padding: 28px 0;
  border-bottom: 1px solid var(--color-border);
}
.list-item:first-of-type { padding-top: 0; }

.list-item__cat {
  margin: 0;
  font-size: 13px;
  font-weight: 700;
  color: var(--color-accent);
}

.list-item__title {
  margin: 6px 0 0;
  font-size: 20px;
  font-weight: 700;
  line-height: 1.45;

  a { color: var(--color-text); }
  a:hover { color: var(--color-accent); text-decoration: none; }
}

.list-item__excerpt {
  margin: 10px 0 0;
  font-size: 14px;
  line-height: 1.6;
  color: var(--color-text-sub);
}

.list-item__date {
  display: block;
  margin-top: 12px;
  font-size: 13px;
  color: var(--color-text-sub);
}

@media (max-width: 900px) {
  .home__main { grid-template-columns: 1fr; gap: 48px; }
  .list-item { grid-template-columns: 1fr; gap: 14px; }
  .list-item__thumb { max-width: 100%; }
}
```

`.home__main`이 1열로 접히면 태그 사이드바는 자연히 리스트 아래로 내려간다.

- [ ] **Step 7-B: `_sass/_sidebar.scss`**

```scss
.tag-sidebar {
  position: sticky;
  top: 24px;
}

.tag-sidebar__title {
  margin: 0 0 16px;
  font-size: 14px;
  font-weight: 600;
  color: var(--color-text-sub);
}

.tag-sidebar__list {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  list-style: none;
  margin: 0;
  padding: 0;
}

.tag-pill {
  display: inline-block;
  padding: 6px 12px;
  border-radius: 999px;
  background: var(--color-bg-soft);
  font-size: 13px;
  color: var(--color-text-sub);
}
.tag-pill:hover {
  background: var(--color-accent);
  color: #fff;
  text-decoration: none;
}

.tag-sidebar__more {
  margin-top: 16px;
  padding: 0;
  border: 0;
  background: none;
  font-family: var(--font-sans);
  font-size: 13px;
  color: var(--color-text-sub);
  cursor: pointer;
}
.tag-sidebar__more:hover { color: var(--color-accent); }

.is-hidden { display: none; }

@media (max-width: 900px) {
  .tag-sidebar { position: static; }
}
```

- [ ] **Step 7-C: `main.scss`에 등록**

```scss
@import "card";
@import "list";
@import "sidebar";
```

- [ ] **Step 8: 태그 더보기 토글 스크립트**

`_includes/tag-sidebar.html` 하단에 추가한다.

```html
<script>
  document.querySelector('[data-tag-more]')?.addEventListener('click', function () {
    document.querySelectorAll('[data-tag-list] .is-hidden').forEach(function (el) {
      el.classList.remove('is-hidden');
    });
    this.remove();
  });
</script>
```

- [ ] **Step 9: 빌드 및 26편 렌더 검증**

```bash
bundle exec jekyll build
grep -c "list-item__title" _site/index.html
```

Expected: 빌드 통과, grep 결과 `26`

- [ ] **Step 10: 폴백 카드 검증**

```bash
grep -c "thumb__fallback" _site/index.html
```

Expected: `4` — 이미지 없는 2편(`github-blog-setup`, `soma17-dev-log`)이 각각 피처드 카드와 리스트에 중복 노출될 수 있으므로 2~4 사이면 정상. 정확한 수는 최신 3편 구성에 따라 달라진다. 0이면 폴백 로직이 동작하지 않는 것이므로 실패.

- [ ] **Step 11: 커밋**

```bash
git add -A
git commit -F - <<'EOF'
feat(theme): 홈 레이아웃 추가 — 피처드 3열·리스트·태그 사이드바

썸네일은 3:2 고정에 object-fit cover. 이미지 없는 포스트는 폴백 카드로 처리한다.

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>
EOF
```

---

### Task 5: post 레이아웃 + 본문 타이포 + 코드블록

**Files:**
- Create: `_layouts/post.html`, `_sass/_post.scss`, `_sass/_syntax.scss`
- Modify: `assets/css/main.scss`

**Interfaces:**
- Consumes: Task 2 토큰
- Produces: `_layouts/post.html` — Task 6(TOC)·Task 9(giscus)가 여기에 붙는다

- [ ] **Step 1: 검증부터 실행해 실패 확인**

```bash
bundle exec jekyll build && test -f _site/posts/새벽에-스팟-인스턴스가-꺼지면-어떡하나-걱정했다-—-EC2-사이징부터-구매옵션까지/index.html && echo FOUND || echo MISSING
```

Expected: `MISSING` 또는 빌드 경고. `post` 레이아웃이 없다.

- [ ] **Step 2: `_layouts/post.html`**

```html
---
layout: default
---

<article class="post wrap">
  <header class="post__header">
    <p class="post__cat">{{ page.categories | join: " › " }}</p>
    <h1 class="post__title">{{ page.title }}</h1>
    <p class="post__meta">
      <time datetime="{{ page.date | date_to_xmlschema }}">{{ page.date | date: "%Y. %-m. %-d." }}</time>
    </p>
  </header>

  <div class="post__body">
    {{ content }}
  </div>

  <footer class="post__footer">
    <ul class="post__tags">
      {% for tag in page.tags %}
        <li><a class="tag-pill" href="{{ '/tags/' | append: tag | slugify | append: '/' | relative_url }}">{{ tag }}</a></li>
      {% endfor %}
    </ul>
  </footer>
</article>
```

TOC는 Task 6, 댓글은 Task 9에서 추가한다.

- [ ] **Step 3: `_sass/_post.scss`**

```scss
.post { max-width: 760px; padding-top: 48px; padding-bottom: 80px; }

.post__cat { margin: 0; font-size: 13px; font-weight: 700; color: var(--color-accent); }
.post__title { margin: 8px 0 0; font-size: 36px; font-weight: 800; line-height: 1.35; color: var(--color-navy); }
.post__meta { margin: 12px 0 0; font-size: 13px; color: var(--color-text-sub); }

.post__body {
  margin-top: 40px;

  h1, h2 { margin: 56px 0 16px; font-size: 26px; font-weight: 800; line-height: 1.4; color: var(--color-navy); }
  h3 { margin: 40px 0 12px; font-size: 20px; font-weight: 700; }
  p { margin: 0 0 20px; }
  img { border-radius: var(--radius-card); }

  blockquote {
    margin: 24px 0;
    padding: 4px 20px;
    border-left: 3px solid var(--color-accent);
    color: var(--color-text-sub);
  }

  table { width: 100%; border-collapse: collapse; margin: 24px 0; font-size: 15px; }
  th, td { padding: 10px 12px; border-bottom: 1px solid var(--color-border); text-align: left; }
  th { background: var(--color-bg-soft); font-weight: 700; }
}

.post__tags { display: flex; flex-wrap: wrap; gap: 8px; list-style: none; margin: 48px 0 0; padding: 32px 0 0; border-top: 1px solid var(--color-border); }

@media (max-width: 900px) {
  .post__title { font-size: 26px; }
  .post__body h1, .post__body h2 { font-size: 21px; }
}
```

테이블 가로 스크롤이 필요하면 `.post__body table`을 `overflow-x:auto` 래퍼로 감싼다. 태현님 포스트에 표가 많으므로 모바일에서 반드시 확인할 것.

- [ ] **Step 4: `_sass/_syntax.scss` — 코드블록**

`_config.yml`의 kramdown 설정이 `css_class: highlight`, `line_numbers: true`다. Rouge가 생성하는 `.highlight` 마크업에 맞춘다.

```scss
.post__body {
  code {
    padding: 2px 6px;
    background: var(--color-bg-soft);
    border-radius: 4px;
    font-size: 14px;
  }

  pre code { padding: 0; background: none; }

  .highlight {
    margin: 24px 0;
    border-radius: var(--radius-card);
    overflow-x: auto;
    background: var(--color-navy);

    pre { margin: 0; padding: 20px; }
    code { color: #E6EDF3; font-size: 14px; line-height: 1.7; }
  }

  .highlight .lineno { color: rgba(255,255,255,.3); padding-right: 16px; user-select: none; }
}
```

- [ ] **Step 4-B: Rouge 토큰 색상 지정**

`_sass/_syntax.scss` 하단에 추가한다. 네이비 배경 기준이며 라이트·다크 모드 공통이다 (코드블록은 양쪽 모두 다크 배경을 쓴다).

```scss
.highlight {
  .c, .c1, .cm, .cs { color: #7D8FA3; font-style: italic; }  // 주석
  .k, .kd, .kn, .kp, .kr, .kt { color: #FF7B72; }            // 키워드
  .s, .s1, .s2, .sb, .sc, .sd, .se, .sh, .si, .sx { color: #A5D6FF; }  // 문자열
  .n, .na, .nb, .nc, .nf, .nn, .nt, .nv { color: #E6EDF3; }  // 이름
  .nf, .nc { color: #D2A8FF; }                               // 함수·클래스명
  .m, .mi, .mf, .mh, .mo { color: #79C0FF; }                 // 숫자
  .o, .ow { color: #FF7B72; }                                // 연산자
  .p { color: #E6EDF3; }                                     // 구두점
  .err { color: #FFA198; }                                   // 에러
  .gd { color: #FFA198; background: rgba(248,81,73,.15); }   // diff 삭제
  .gi { color: #7EE787; background: rgba(63,185,80,.15); }   // diff 추가
}
```

- [ ] **Step 4-C: 코드블록 렌더 검증**

태현님 포스트에는 SQL·Java·YAML·bash 코드블록이 섞여 있다. 각 언어가 실제로 하이라이팅되는지 확인한다.

```bash
bundle exec jekyll build
grep -c "class=\"highlight\"" _site/posts/*/index.html | grep -v ":0" | wc -l
```

Expected: `1` 이상. 코드블록이 있는 포스트 수만큼 나온다.

- [ ] **Step 5: `main.scss`에 등록 후 빌드 검증**

```scss
@import "post";
@import "syntax";
```

```bash
bundle exec jekyll build
ls _site/posts/ | wc -l
```

Expected: 빌드 통과, `26`

- [ ] **Step 6: URL 불변 회귀 검사**

```bash
ls _site/posts/ | head -5
```

Expected: 포스트 제목 기반 한글 디렉터리명. Task 1 이전과 동일해야 한다. 기존 URL이 `/posts/:title/`이므로 형식이 바뀌었다면 즉시 중단하고 `_config.yml`의 permalink 설정을 확인할 것.

- [ ] **Step 7: 커밋**

```bash
git add -A
git commit -F - <<'EOF'
feat(theme): post 레이아웃과 본문 타이포·코드블록 추가

기존 permalink(/posts/:title/)를 유지해 26편 URL이 불변이다.

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>
EOF
```

---

### Task 6: 포스트 TOC

`_config.yml`의 `toc: true`(posts defaults)를 읽어 본문 h2/h3로 목차를 만든다.

**Files:**
- Create: `_includes/toc.html`, `_sass/_toc.scss`
- Modify: `_layouts/post.html`, `assets/css/main.scss`

**Interfaces:**
- Consumes: Task 5의 `_layouts/post.html`, `.post__body` 마크업
- Produces: 없음 (터미널 컴포넌트)

- [ ] **Step 1: `_includes/toc.html`**

kramdown이 헤딩에 자동으로 `id`를 붙이므로 JS로 수집한다. Liquid만으로는 본문 헤딩 파싱이 불안정하다.

```html
<nav class="toc" data-toc aria-label="목차">
  <p class="toc__title">목차</p>
  <ul class="toc__list" data-toc-list></ul>
</nav>

<script>
  (function () {
    var body = document.querySelector('.post__body');
    var list = document.querySelector('[data-toc-list]');
    if (!body || !list) return;

    var heads = body.querySelectorAll('h1[id], h2[id], h3[id]');
    if (heads.length === 0) { document.querySelector('[data-toc]').remove(); return; }

    heads.forEach(function (h) {
      var li = document.createElement('li');
      li.className = 'toc__item toc__item--' + h.tagName.toLowerCase();
      var a = document.createElement('a');
      a.href = '#' + h.id;
      a.textContent = h.textContent;
      li.appendChild(a);
      list.appendChild(li);
    });
  })();
</script>
```

- [ ] **Step 2: `_layouts/post.html`에 조건부 삽입**

`.post__body` 앞이 아니라 **뒤**에 넣는다. 스크립트가 본문을 읽어야 하므로 DOM 순서상 본문이 먼저 있어야 한다. CSS로 위치를 잡는다.

`<div class="post__body">{{ content }}</div>` 다음 줄에 추가:

```html
{% if page.toc %}{% include toc.html %}{% endif %}
```

그리고 `<article class="post wrap">`를 `<article class="post wrap{% if page.toc %} post--toc{% endif %}">`로 바꾼다.

- [ ] **Step 3: `_sass/_toc.scss`**

```scss
.post--toc {
  max-width: var(--maxw-page);
  display: grid;
  grid-template-columns: minmax(0, 760px) 220px;
  grid-template-areas:
    "header ."
    "body   toc"
    "footer .";
  gap: 0 48px;
  justify-content: center;
}

.post--toc .post__header { grid-area: header; }
.post--toc .post__body { grid-area: body; }
.post--toc .post__footer { grid-area: footer; }
.post--toc .toc { grid-area: toc; }

.toc {
  position: sticky;
  top: 24px;
  align-self: start;
  margin-top: 40px;
  font-size: 13px;
}

.toc__title { margin: 0 0 12px; font-weight: 700; color: var(--color-text); }
.toc__list { list-style: none; margin: 0; padding: 0; border-left: 1px solid var(--color-border); }
.toc__item a { display: block; padding: 5px 0 5px 12px; color: var(--color-text-sub); line-height: 1.5; }
.toc__item a:hover { color: var(--color-accent); text-decoration: none; }
.toc__item--h3 a { padding-left: 24px; }

@media (max-width: 1100px) {
  .post--toc { display: block; max-width: 760px; }
  .toc { display: none; }
}
```

- [ ] **Step 4: 빌드 및 검증**

```bash
bundle exec jekyll build
grep -c "data-toc-list" _site/posts/*/index.html | grep -v ":0" | wc -l
```

Expected: `26` — 전 포스트에 TOC 컨테이너가 있다 (`toc: true`가 posts defaults이므로).

- [ ] **Step 5: 커밋**

```bash
git add -A
git commit -F - <<'EOF'
feat(theme): 포스트 TOC 추가

kramdown이 붙인 헤딩 id를 JS로 수집해 목차를 생성한다.
1100px 이하에서는 숨긴다.

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>
EOF
```

---

### Task 7: 태그 페이지

`jekyll-archives`가 `/tags/:name/`로 페이지를 생성한다. `tag` 레이아웃만 만들면 된다.

**Files:**
- Create: `_layouts/tag.html`
- Modify: 없음

**Interfaces:**
- Consumes: Task 4의 `list-item.html`, `tag-sidebar.html`
- Produces: 없음

- [ ] **Step 1: 검증부터 실행해 실패 확인**

```bash
bundle exec jekyll build && ls _site/tags/ 2>/dev/null | head -3
```

Expected: 빈 출력 또는 `tag` 레이아웃 없음 경고.

- [ ] **Step 2: `_layouts/tag.html`**

`jekyll-archives`는 `page.tag`에 태그명, `page.posts`에 해당 포스트를 넣어준다.

```html
---
layout: default
---

<div class="wrap tag-page">
  <header class="tag-page__header">
    <p class="section-label">Tag</p>
    <h1 class="tag-page__title">{{ page.tag }}</h1>
    <p class="tag-page__count">{{ page.posts | size }}편</p>
  </header>

  <div class="home__main">
    <section class="home__list">
      {% for post in page.posts %}
        {% include list-item.html post=post %}
      {% endfor %}
    </section>
    {% include tag-sidebar.html %}
  </div>
</div>
```

- [ ] **Step 3: 빌드 및 태그 URL 검증**

```bash
bundle exec jekyll build
ls _site/tags/ | wc -l
test -d _site/tags/aws && echo "URL OK" || echo "URL BROKEN"
```

Expected: 태그 개수 출력, `URL OK`. `/tags/:name/` 형식이 유지되어야 한다.

- [ ] **Step 4: 사이드바 링크 연결 검증**

```bash
grep -o 'href="/tags/[^"]*"' _site/index.html | head -3
```

Expected: `href="/tags/aws/"` 형태. Task 4의 `tag-sidebar.html`이 만든 링크가 실제 생성된 페이지와 일치해야 한다. 불일치 시 `slugify` 처리를 맞출 것.

- [ ] **Step 5: 커밋**

```bash
git add -A
git commit -F - <<'EOF'
feat(theme): 태그 페이지 레이아웃 추가

jekyll-archives가 생성하는 /tags/:name/ URL을 유지한다.

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>
EOF
```

---

### Task 8: 다크모드

**Files:**
- Modify: `_sass/_tokens.scss`, `_includes/header.html`, `_layouts/default.html`
- Create: `_sass/_darkmode.scss`

**Interfaces:**
- Consumes: Task 2의 토큰 이름
- Produces: `[data-theme="dark"]` 속성 계약

- [ ] **Step 1: `_sass/_tokens.scss`에 다크 세트 추가**

토큰 이름은 그대로 두고 값만 재정의한다. 컴포넌트 SCSS는 한 줄도 안 고친다.

```scss
[data-theme="dark"] {
  --color-navy: #0A1728;
  --color-accent: #5B9BD5;
  --color-text: #E6EDF3;
  --color-text-sub: #8B9BB0;
  --color-border: #1F2D3F;
  --color-bg: #0D1520;
  --color-bg-soft: #16202E;
}
```

- [ ] **Step 2: 토글 버튼을 `_includes/header.html` 네비에 추가**

검색 버튼 옆에 넣는다.

```html
<button class="site-header__theme" type="button" aria-label="테마 전환" data-theme-toggle>
  <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <path d="M21 12.8A9 9 0 1111.2 3a7 7 0 009.8 9.8z"/>
  </svg>
</button>
```

- [ ] **Step 3: FOUC 방지 스크립트를 `default.html`의 `<head>` 끝에 추가**

CSS 로드 전에 실행되어야 화면 깜빡임이 없다.

```html
<script>
  (function () {
    var saved = localStorage.getItem('theme');
    var dark = saved ? saved === 'dark' : window.matchMedia('(prefers-color-scheme: dark)').matches;
    if (dark) document.documentElement.setAttribute('data-theme', 'dark');
  })();
</script>
```

- [ ] **Step 4: 토글 동작 스크립트를 `default.html`의 `</body>` 직전에 추가**

```html
<script>
  document.querySelector('[data-theme-toggle]')?.addEventListener('click', function () {
    var root = document.documentElement;
    var next = root.getAttribute('data-theme') === 'dark' ? 'light' : 'dark';
    root.setAttribute('data-theme', next);
    localStorage.setItem('theme', next);
  });
</script>
```

- [ ] **Step 5: 다크모드 전용 보정 — `_sass/_darkmode.scss`**

히어로는 라이트에서도 네이비 배경에 흰 글씨였으므로 그대로 동작한다. 보정이 필요한 곳은 썸네일이다. 태현님 포스트 이미지는 대부분 흰 배경이라 다크모드에서 눈부시게 튄다.

```scss
[data-theme="dark"] {
  .thumb img { filter: brightness(.88); }
  .post__body img { filter: brightness(.9); }
}
```

`main.scss`에 `@import "darkmode";` 추가.

- [ ] **Step 6: 빌드 및 검증**

```bash
bundle exec jekyll build
grep -c 'data-theme="dark"' _site/assets/css/main.css
grep -c "data-theme-toggle" _site/index.html
```

Expected: 각각 `1` 이상

- [ ] **Step 7: 커밋**

```bash
git add -A
git commit -F - <<'EOF'
feat(theme): 다크모드 추가

토큰 값만 재정의하는 방식이라 컴포넌트 SCSS는 수정하지 않았다.
흰 배경 다이어그램이 다크모드에서 튀지 않도록 밝기를 보정했다.

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>
EOF
```

---

### Task 9: giscus 댓글 + Google Analytics + GoatCounter

`_config.yml`의 기존 값을 그대로 읽는다. 설정은 Task 1에서 남겨두었다.

**Files:**
- Create: `_includes/comments.html`, `_includes/analytics.html`
- Modify: `_layouts/post.html`, `_layouts/default.html`

**Interfaces:**
- Consumes: `site.comments.giscus.*`, `site.analytics.google.id`, `site.pageviews.provider`, `site.analytics.goatcounter.id`

- [ ] **Step 1: `_includes/comments.html`**

`mapping: pathname`이며 permalink가 불변이므로 기존 댓글 스레드가 그대로 붙는다.

```html
{% if site.comments.provider == 'giscus' and page.comments %}
<section class="comments">
  <script src="https://giscus.app/client.js"
    data-repo="{{ site.comments.giscus.repo }}"
    data-repo-id="{{ site.comments.giscus.repo_id }}"
    data-category="{{ site.comments.giscus.category }}"
    data-category-id="{{ site.comments.giscus.category_id }}"
    data-mapping="{{ site.comments.giscus.mapping | default: 'pathname' }}"
    data-strict="{{ site.comments.giscus.strict | default: '1' }}"
    data-reactions-enabled="{{ site.comments.giscus.reactions_enabled | default: '1' }}"
    data-input-position="{{ site.comments.giscus.input_position | default: 'top' }}"
    data-lang="{{ site.comments.giscus.lang | default: site.lang }}"
    data-theme="light"
    data-emit-metadata="0"
    crossorigin="anonymous"
    async>
  </script>
</section>
{% endif %}
```

- [ ] **Step 2: `_layouts/post.html`에 삽입**

`</article>` 다음, 즉 post 컨테이너 밖에 넣되 같은 폭을 유지한다.

```html
<div class="wrap comments-wrap">
  {% include comments.html %}
</div>
```

`.comments-wrap { max-width: 760px; padding-bottom: 80px; }`를 `_post.scss`에 추가.

- [ ] **Step 3: 다크모드에서 giscus 테마 동기화**

giscus는 iframe이라 CSS가 안 먹는다. postMessage로 테마를 바꾼다. `comments.html` 하단에 추가.

```html
<script>
  (function () {
    function setGiscusTheme(theme) {
      var frame = document.querySelector('iframe.giscus-frame');
      if (!frame) return;
      frame.contentWindow.postMessage(
        { giscus: { setConfig: { theme: theme } } },
        'https://giscus.app'
      );
    }

    function currentTheme() {
      return document.documentElement.getAttribute('data-theme') === 'dark' ? 'dark' : 'light';
    }

    window.addEventListener('message', function (e) {
      if (e.origin === 'https://giscus.app') setGiscusTheme(currentTheme());
    });

    document.querySelector('[data-theme-toggle]')?.addEventListener('click', function () {
      setTimeout(function () { setGiscusTheme(currentTheme()); }, 0);
    });
  })();
</script>
```

- [ ] **Step 4: `_includes/analytics.html`**

프로덕션 빌드에서만 출력한다. 로컬 개발이 통계를 오염시키면 안 된다.

```html
{% if jekyll.environment == 'production' %}
  {% if site.analytics.google.id %}
  <script async src="https://www.googletagmanager.com/gtag/js?id={{ site.analytics.google.id }}"></script>
  <script>
    window.dataLayer = window.dataLayer || [];
    function gtag(){dataLayer.push(arguments);}
    gtag('js', new Date());
    gtag('config', '{{ site.analytics.google.id }}');
  </script>
  {% endif %}

  {% if site.pageviews.provider == 'goatcounter' and site.analytics.goatcounter.id %}
  <script data-goatcounter="https://{{ site.analytics.goatcounter.id }}.goatcounter.com/count"
          async src="//gc.zgo.at/count.js"></script>
  {% endif %}
{% endif %}
```

- [ ] **Step 5: `default.html`의 `</body>` 직전에 삽입**

```html
{% include analytics.html %}
```

- [ ] **Step 6: 개발 빌드에 추적 스크립트가 없는지 검증**

```bash
bundle exec jekyll build
grep -c "googletagmanager" _site/index.html
```

Expected: `0` — 개발 빌드이므로 출력되지 않아야 한다.

- [ ] **Step 7: 프로덕션 빌드에 추적 스크립트가 있는지 검증**

```bash
JEKYLL_ENV=production bundle exec jekyll build
grep -c "G-VGD2N6XR7K" _site/index.html
grep -c "goatcounter" _site/index.html
grep -c "giscus.app" _site/posts/*/index.html | grep -v ":0" | wc -l
```

Expected: 각각 `1` 이상, 마지막은 `26`

- [ ] **Step 8: 커밋**

```bash
git add -A
git commit -F - <<'EOF'
feat(theme): giscus 댓글·GA·GoatCounter 재구현

Chirpy가 제공하던 기능을 직접 붙였다. _config.yml의 기존 설정을 그대로 읽는다.
mapping: pathname과 permalink 불변으로 기존 댓글 스레드가 보존된다.
추적 스크립트는 프로덕션 빌드에서만 출력한다.

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>
EOF
```

---

### Task 10: 검색

**Files:**
- Create: `search.json`, `_includes/search.html`, `_sass/_search.scss`
- Modify: `_layouts/default.html`, `assets/css/main.scss`

**Interfaces:**
- Consumes: Task 3 `header.html`의 `[data-search-toggle]` 버튼
- Produces: `/search.json` 엔드포인트

- [ ] **Step 1: `search.json` — 빌드 시 인덱스 생성**

저장소 루트에 만든다.

```liquid
---
layout: null
permalink: /search.json
---
[
  {% for post in site.posts %}
    {
      "title": {{ post.title | jsonify }},
      "url": {{ post.url | relative_url | jsonify }},
      "date": {{ post.date | date: "%Y. %-m. %-d." | jsonify }},
      "categories": {{ post.categories | join: " › " | jsonify }},
      "tags": {{ post.tags | join: " " | jsonify }},
      "body": {{ post.content | strip_html | normalize_whitespace | truncate: 400 | jsonify }}
    }{% unless forloop.last %},{% endunless %}
  {% endfor %}
]
```

- [ ] **Step 2: 인덱스 생성 검증**

```bash
bundle exec jekyll build
test -f _site/search.json && echo FOUND || echo MISSING
ruby -rjson -e 'puts JSON.parse(File.read("_site/search.json")).size'
```

Expected: `FOUND`, `26`

JSON 파싱이 실패하면 포스트 제목의 따옴표·특수문자가 이스케이프되지 않은 것이다. `jsonify` 필터가 빠진 곳을 찾을 것.

- [ ] **Step 3: `_includes/search.html`**

```html
<div class="search" data-search hidden>
  <div class="search__panel">
    <input class="search__input" type="search" placeholder="검색어를 입력하세요" data-search-input aria-label="검색">
    <ul class="search__results" data-search-results></ul>
  </div>
</div>

<script>
  (function () {
    var box = document.querySelector('[data-search]');
    var input = document.querySelector('[data-search-input]');
    var results = document.querySelector('[data-search-results]');
    var toggle = document.querySelector('[data-search-toggle]');
    var index = null;

    if (!box || !toggle) return;

    function open() {
      box.hidden = false;
      input.focus();
      if (!index) {
        fetch('{{ "/search.json" | relative_url }}')
          .then(function (r) { return r.json(); })
          .then(function (data) { index = data; });
      }
    }

    function close() { box.hidden = true; input.value = ''; results.innerHTML = ''; }

    toggle.addEventListener('click', open);
    box.addEventListener('click', function (e) { if (e.target === box) close(); });
    document.addEventListener('keydown', function (e) { if (e.key === 'Escape') close(); });

    input.addEventListener('input', function () {
      var q = this.value.trim().toLowerCase();
      results.innerHTML = '';
      if (!index || q.length < 2) return;

      var hits = index.filter(function (p) {
        return (p.title + ' ' + p.tags + ' ' + p.categories + ' ' + p.body).toLowerCase().indexOf(q) !== -1;
      }).slice(0, 8);

      if (hits.length === 0) {
        results.innerHTML = '<li class="search__empty">결과가 없습니다</li>';
        return;
      }

      hits.forEach(function (p) {
        var li = document.createElement('li');
        li.className = 'search__hit';
        var a = document.createElement('a');
        a.href = p.url;
        a.innerHTML = '<strong></strong><span></span>';
        a.querySelector('strong').textContent = p.title;
        a.querySelector('span').textContent = p.categories + ' · ' + p.date;
        li.appendChild(a);
        results.appendChild(li);
      });
    });
  })();
</script>
```

`textContent`로 넣는 이유는 검색 결과에 사용자 입력이 섞이지 않게 하기 위함이다. `innerHTML`로 제목을 넣지 말 것.

- [ ] **Step 4: `_sass/_search.scss`**

```scss
.search {
  position: fixed;
  inset: 0;
  z-index: 100;
  background: rgba(11, 31, 58, .5);
  display: flex;
  justify-content: center;
  padding-top: 15vh;
}

.search[hidden] { display: none; }

.search__panel {
  width: min(560px, 90vw);
  height: fit-content;
  background: var(--color-bg);
  border-radius: var(--radius-card);
  overflow: hidden;
}

.search__input {
  width: 100%;
  padding: 18px 20px;
  border: 0;
  border-bottom: 1px solid var(--color-border);
  background: transparent;
  color: var(--color-text);
  font-family: var(--font-sans);
  font-size: 16px;
  outline: none;
}

.search__results { list-style: none; margin: 0; padding: 0; max-height: 50vh; overflow-y: auto; }

.search__hit a {
  display: block;
  padding: 14px 20px;
  border-bottom: 1px solid var(--color-border);
  color: var(--color-text);

  strong { display: block; font-size: 15px; font-weight: 700; }
  span { display: block; margin-top: 4px; font-size: 12px; color: var(--color-text-sub); }
  &:hover { background: var(--color-bg-soft); text-decoration: none; }
}

.search__empty { padding: 20px; color: var(--color-text-sub); font-size: 14px; }
```

- [ ] **Step 5: `default.html`에 삽입**

`{% include footer.html %}` 다음에 추가.

```html
{% include search.html %}
```

`main.scss`에 `@import "search";` 추가.

- [ ] **Step 6: 빌드 및 검증**

```bash
bundle exec jekyll build
grep -c "data-search-input" _site/index.html
```

Expected: `1`

- [ ] **Step 7: 커밋**

```bash
git add -A
git commit -F - <<'EOF'
feat(theme): 클라이언트 사이드 검색 추가

빌드 시 search.json 인덱스를 생성하고 클라이언트에서 필터링한다.
외부 서비스 의존이 없다.

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>
EOF
```

---

### Task 11: 최종 검증 및 회귀 검사

**Files:** 없음 (검증 전용)

- [ ] **Step 1: 프로덕션 빌드 + html-proofer**

GitHub Actions 워크플로우와 동일한 명령이다. 여기서 통과해야 배포도 통과한다.

```bash
JEKYLL_ENV=production bundle exec jekyll build -d "_site"
bundle exec htmlproofer _site \
  --disable-external \
  --ignore-urls "/^http:\/\/127.0.0.1/,/^http:\/\/0.0.0.0/,/^http:\/\/localhost/"
```

Expected: 통과. 깨진 내부 링크·이미지 0건.

- [ ] **Step 2: 스펙 11장 회귀 검사표 전항목 실행**

```bash
# 26편 URL 불변
ls _site/posts/ | wc -l                                    # 26
# 네이버 인증 메타
grep -rc "3acf5e196b4f28c3006c9421b91b997f79ef39aa" _site/index.html   # 1
# GA
grep -c "G-VGD2N6XR7K" _site/index.html                    # 1 이상
# GoatCounter
grep -c "goatcounter" _site/index.html                     # 1 이상
# giscus (전 포스트)
grep -lc "giscus.app" _site/posts/*/index.html | wc -l      # 26
# 태그 URL
ls _site/tags/ | head -3                                   # /tags/:name/ 형식
# 사이트맵·RSS
test -f _site/sitemap.xml && echo OK
```

각 명령의 기대값이 주석에 있다. 하나라도 어긋나면 해당 Task로 돌아간다.

- [ ] **Step 3: 로컬 서버 띄우고 브라우저 검증**

```bash
bundle exec jekyll serve --livereload
```

`.claude/launch.json`에 등록해 preview_start로 띄우는 편이 낫다. 확인 항목:

| 항목 | 확인 내용 |
|------|----------|
| 홈 | 히어로 / 피처드 3열 / 리스트 26편 / 태그 사이드바 |
| 썸네일 | 3:2 비율, 크롭 상태 확인. 폴백 카드 2편 |
| 포스트 | 본문 타이포, 코드블록, 표(모바일 가로 스크롤), TOC 스크롤 추종 |
| 태그 | 사이드바 pill 클릭 → 태그 페이지 이동 |
| 다크모드 | 토글 동작, 새로고침 후 유지, 썸네일 밝기 보정 |
| 검색 | 돋보기 클릭 → 패널, 2글자 이상 입력 시 결과, Esc 닫기 |
| 반응형 | 375px / 768px / 1280px |

- [ ] **Step 4: 스크린샷으로 결과 공유**

데스크톱 홈, 모바일 홈, 포스트 본문, 다크모드 홈 4장을 찍어 사용자에게 보여준다.

- [ ] **Step 5: 메모리 파일 갱신**

`.claude/memory/블로그현황.md`의 "이미지 스타일 히스토리" 섹션 위에 디자인 교체 이력을 기록한다. 앞으로 `/new-post`가 참조한다.

- [ ] **Step 6: 최종 커밋**

```bash
git add -A
git commit -F - <<'EOF'
chore(theme): 디자인 교체 최종 검증 및 메모리 갱신

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>
EOF
```

---

## 후속 작업 (이 계획의 범위 밖)

스펙 13장 참조. 각각 별도 세션에서 진행한다.

1. **썸네일 일러스트 26장 신규 제작** — 1200×800, 플랫 벡터 스타일. 파일 교체만으로 반영된다
2. **Oracle Cloud 프리티어 VM 이전** — 도메인, 배포 파이프라인, HTTPS
3. **`CLAUDE.md` 카테고리 규칙 근거 수정** — 사이드바가 태그 전용이 되면서 "사이드바 개수 최소화" 근거가 성립하지 않는다. 중첩 구조 자체는 유지
4. **`.github/workflows/pages-deploy.yml`의 ruby-version 확인** — 워크플로우는 3.4, 로컬은 3.3.9. 현재 문제없으나 Gemfile.lock 재생성 후 CI 실패 시 여기를 볼 것
