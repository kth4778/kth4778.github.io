# 뱅크샐러드 스타일 블로그 디자인 개편 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Chirpy gem 기반 디자인을 걷어내고, 뱅크샐러드 기술블로그(blog.banksalad.com) 스타일을 참고한 커스텀 Jekyll 테마로 사이트 전체(홈·포스트·카테고리·태그·아카이브·About)를 재구축한다.

**Architecture:** Chirpy gem은 마지막 태스크까지 Gemfile에 유지한 채, 리포 안에 새 `_layouts/*.html` · `_includes/new-*.html` · `assets/css/main.css`를 추가해 Jekyll의 테마 오버라이드 메커니즘으로 페이지 단위로 점진적으로 교체한다. 새 레이아웃은 Chirpy의 `default.html`을 공유하지 않고 독자적인 `_layouts/base.html`을 shell로 쓰기 때문에, 아직 오버라이드하지 않은 Chirpy 페이지는 마지막 태스크 전까지 100% 정상 동작한다. 마지막 태스크에서만 Gemfile·`_config.yml`에서 Chirpy 의존성을 제거한다.

**Tech Stack:** Jekyll 4.4 (gem, 유지), jekyll-archives / jekyll-paginate / jekyll-seo-tag / jekyll-sitemap (Chirpy가 물고 있던 걸 직접 의존성으로 승격), 순수 CSS(Sass 미사용, `assets/css/main.css`), Pretendard 웹폰트(CDN).

## Global Constraints

- **브랜치:** 별도 브랜치·워크트리 생성 금지. `develop` 브랜치에서 직접 작업한다 (CLAUDE.md: "브랜치는 항상 2개만 유지, 글마다 브랜치 생성하지 않는다"). `superpowers:using-git-worktrees`는 사용하지 않는다.
- **보존 파일 (절대 수정·삭제 금지):** `_posts/**`, `assets/img/**`, `.env`, `_data/**`, `.claude/**`, `CLAUDE.md`, `_plugins/posts-lastmod-hook.rb` (Chirpy와 무관한 git 로그 기반 커스텀 로직이라 디자인 개편과 별개로 보존).
- **네임스페이스 충돌 방지:** 새로 만드는 `_includes/*` 파일은 전부 `new-` 접두사를 붙인다 (`new-head.html`, `new-nav.html`, `new-footer.html`, `new-post-card.html`, `new-pager.html`). Chirpy gem이 이미 `head.html`, `footer.html`, `post-list.html` 같은 이름을 쓰고 있어서, 접두사 없이 만들면 아직 마이그레이션 안 된 Chirpy 페이지까지 의도치 않게 오버라이드된다. `_layouts/*`는 반대로 Chirpy와 **동일한 파일명**을 의도적으로 써야 해당 페이지 타입만 오버라이드된다 (`home.html`, `post.html`, `page.html`, `categories.html`, `tags.html`, `archives.html`, `category.html`, `tag.html`).
- **새 레이아웃은 `default`가 아니라 `base`를 상속한다:** 새 `_layouts/*.html`은 front matter에 `layout: base`를 쓴다 (Chirpy의 `default.html`을 절대 오버라이드하지 않는다 — 그건 아직 안 건드린 모든 Chirpy 페이지의 공통 부모라 건드리는 순간 전체 사이트가 깨진다).
- **폰트:** Pretendard, CDN import `@import url('https://cdn.jsdelivr.net/gh/orioncactus/pretendard/dist/web/static/pretendard.css');`
- **테스트 방식:** 이 리포는 정적 사이트라 별도 유닛테스트 프레임워크가 없다. 모든 태스크의 "테스트"는 다음 패턴으로 통일한다:
  1. `bundle exec jekyll build` 로 사이트 빌드
  2. `grep`으로 `_site/**` 안의 기대하는 마크업이 나왔는지 확인
  3. 실패하면 코드 수정 후 재빌드·재확인
- **Chirpy gem 제거 시점:** 태스크 1~9(모든 레이아웃 완성)가 끝나고 로컬 빌드 검증까지 통과한 뒤, 태스크 10에서만 제거한다. 순서를 바꾸지 않는다.
- **커밋:** 각 태스크 끝에 1회. 태스크 10 전까지는 항상 `bundle exec jekyll build`가 에러 없이 성공하는 상태를 유지한다.

---

## Task 1: 디자인 토큰 & 베이스 CSS

**Files:**
- Create: `assets/css/main.css`

**Interfaces:**
- Produces: CSS 커스텀 프로퍼티(`--color-bg`, `--color-text`, `--color-text-secondary`, `--color-text-tertiary`, `--color-border`, `--color-accent`, `--font-sans`, `--space-1`~`--space-6`, `--radius-sm`, `--container-width`)와 컴포넌트 클래스(`.site-nav`, `.nav-inner`, `.nav-brand`, `.nav-tabs`, `.nav-tab`, `.is-active`, `.site-main`, `.site-footer`, `.footer-inner`, `.post-list`, `.post-card`, `.post-card-date`, `.post-card-day`, `.post-card-month`, `.post-card-body`, `.post-card-tags`, `.post-card-tag`, `.post-card-title`, `.post-card-excerpt`, `.post-card-more`, `.pager`, `.post-detail`, `.post-header`, `.post-title`, `.post-meta`, `.post-tags`, `.post-tag`, `.post-content`, `.page-content`, `.page-title`, `.archive-index`, `.archive-title`, `.archive-group-list`, `.archive-group`, `.tag-cloud`, `.tag-pill`, `.archive-year`, `.archive-post-list`). 이후 태스크(2~9)의 모든 HTML은 이 클래스명을 정확히 그대로 사용한다.

- [ ] **Step 1: `assets/css/main.css` 작성**

```css
@import url('https://cdn.jsdelivr.net/gh/orioncactus/pretendard/dist/web/static/pretendard.css');

:root {
  --color-bg: #ffffff;
  --color-text: #1a1a1a;
  --color-text-secondary: #757575;
  --color-text-tertiary: #a3a3a3;
  --color-border: #ececec;
  --color-accent: #292929;
  --font-sans: 'Pretendard', -apple-system, BlinkMacSystemFont, 'Apple SD Gothic Neo',
    'Malgun Gothic', sans-serif;
  --space-1: 4px;
  --space-2: 8px;
  --space-3: 16px;
  --space-4: 24px;
  --space-5: 40px;
  --space-6: 64px;
  --radius-sm: 4px;
  --container-width: 720px;
}

* { box-sizing: border-box; }

body {
  margin: 0;
  font-family: var(--font-sans);
  color: var(--color-text);
  background: var(--color-bg);
  line-height: 1.7;
}

a { color: inherit; text-decoration: none; }

.site-main {
  max-width: var(--container-width);
  margin: 0 auto;
  padding: var(--space-5) var(--space-3);
}

/* nav */
.site-nav {
  border-bottom: 1px solid var(--color-border);
}
.nav-inner {
  max-width: var(--container-width);
  margin: 0 auto;
  padding: var(--space-3);
  display: flex;
  align-items: center;
  gap: var(--space-4);
}
.nav-brand {
  font-weight: 600;
  font-size: 16px;
}
.nav-tabs {
  display: flex;
  gap: var(--space-3);
}
.nav-tab {
  font-size: 14px;
  color: var(--color-text-secondary);
  padding: var(--space-1) 0;
}
.nav-tab.is-active {
  color: var(--color-accent);
  font-weight: 600;
  border-bottom: 2px solid var(--color-accent);
}

/* footer */
.site-footer {
  border-top: 1px solid var(--color-border);
  margin-top: var(--space-6);
}
.footer-inner {
  max-width: var(--container-width);
  margin: 0 auto;
  padding: var(--space-4) var(--space-3);
  font-size: 12px;
  color: var(--color-text-tertiary);
}
.footer-links a { margin-right: var(--space-2); color: var(--color-text-secondary); }

/* home hero */
.home-hero { margin-bottom: var(--space-5); }
.home-hero h1 { font-size: 32px; font-weight: 500; margin: 0 0 var(--space-1); }
.home-tagline { color: var(--color-text-secondary); margin: 0; }

/* post card list */
.post-list { list-style: none; margin: 0; padding: 0; }
.post-card {
  display: flex;
  gap: var(--space-4);
  padding: var(--space-4) 0;
  border-bottom: 1px solid var(--color-border);
}
.post-card-date {
  flex: 0 0 48px;
  text-align: center;
}
.post-card-day { display: block; font-size: 20px; font-weight: 600; }
.post-card-month { display: block; font-size: 11px; color: var(--color-text-tertiary); letter-spacing: 0.05em; }
.post-card-tags { font-size: 12px; color: var(--color-text-tertiary); margin-bottom: var(--space-1); }
.post-card-tag { margin-right: var(--space-2); }
.post-card-title { font-size: 18px; font-weight: 600; margin: 0 0 var(--space-1); }
.post-card-excerpt { font-size: 14px; color: var(--color-text-secondary); margin: 0 0 var(--space-1); }
.post-card-more { font-size: 13px; color: var(--color-accent); }

/* pager */
.pager {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: var(--space-3);
  margin-top: var(--space-5);
  font-size: 13px;
}
.pager-count { color: var(--color-text-secondary); }

/* post detail */
.post-header { margin-bottom: var(--space-4); }
.post-title { font-size: 28px; font-weight: 600; margin: 0 0 var(--space-2); }
.post-meta { font-size: 13px; color: var(--color-text-secondary); display: flex; gap: var(--space-2); }
.post-tags { margin-top: var(--space-2); }
.post-tag { font-size: 12px; color: var(--color-text-tertiary); margin-right: var(--space-2); }
.post-content { font-size: 16px; }
.post-content img { max-width: 100%; border-radius: var(--radius-sm); }
.post-content pre { background: #f5f5f5; padding: var(--space-3); overflow-x: auto; border-radius: var(--radius-sm); }

/* page (about) */
.page-title { font-size: 26px; font-weight: 600; }

/* archive index pages */
.archive-title { font-size: 26px; font-weight: 600; margin-bottom: var(--space-4); }
.archive-group-list { list-style: none; margin: 0; padding: 0; }
.archive-group {
  display: flex;
  justify-content: space-between;
  padding: var(--space-2) 0;
  border-bottom: 1px solid var(--color-border);
  font-size: 15px;
}
.archive-group-count { color: var(--color-text-tertiary); }

.tag-cloud { list-style: none; margin: 0; padding: 0; display: flex; flex-wrap: wrap; gap: var(--space-2); }
.tag-pill {
  font-size: 13px;
  border: 1px solid var(--color-border);
  border-radius: 999px;
  padding: var(--space-1) var(--space-3);
  color: var(--color-text-secondary);
}
.tag-pill span { color: var(--color-text-tertiary); margin-left: var(--space-1); }

.archive-year { margin-bottom: var(--space-4); }
.archive-year h2 { font-size: 18px; font-weight: 600; }
.archive-post-list { list-style: none; margin: 0; padding: 0; }
.archive-post-list li {
  display: flex;
  gap: var(--space-3);
  font-size: 14px;
  padding: var(--space-1) 0;
}
.archive-post-list time { color: var(--color-text-tertiary); width: 48px; }

@media (max-width: 600px) {
  .nav-tabs { gap: var(--space-2); font-size: 13px; }
  .post-card { flex-direction: column; gap: var(--space-1); }
}
```

- [ ] **Step 2: 빌드 확인**

Run: `bundle exec jekyll build`
Expected: 에러 없이 성공 (기존 Chirpy 페이지는 아직 전혀 안 건드렸으므로 통상 빌드와 동일)

- [ ] **Step 3: CSS 파일이 `_site`에 복사됐는지 확인**

Run: `grep -q "color-accent" _site/assets/css/main.css && echo PASS`
Expected: `PASS`

- [ ] **Step 4: 커밋**

```bash
git add assets/css/main.css
git commit -m "style: 뱅크샐러드 스타일 디자인 토큰 및 베이스 CSS 추가"
```

---

## Task 2: 공통 인클루드 (head / nav / footer)

**Files:**
- Create: `_includes/new-head.html`
- Create: `_includes/new-nav.html`
- Create: `_includes/new-footer.html`

**Interfaces:**
- Consumes: Task 1의 CSS 클래스명(`.site-nav`, `.nav-inner`, `.nav-brand`, `.nav-tabs`, `.nav-tab`, `.is-active`, `.site-footer`, `.footer-inner`, `.footer-links`)
- Produces: `_layouts/base.html`(Task 3)이 `{% include new-head.html %}`, `{% include new-nav.html %}`, `{% include new-footer.html %}`로 그대로 호출

- [ ] **Step 1: `_includes/new-head.html` 작성**

```html
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{% if page.title %}{{ page.title }} · {{ site.title }}{% else %}{{ site.title }}{% endif %}</title>
  <meta name="description" content="{{ page.excerpt | default: site.description | strip_html | strip_newlines | truncate: 160 }}">
  <meta name="naver-site-verification" content="3acf5e196b4f28c3006c9421b91b997f79ef39aa">
  <link rel="icon" href="{{ '/assets/img/favicons/favicon.ico' | relative_url }}">
  <link rel="stylesheet" href="{{ '/assets/css/main.css' | relative_url }}">
  {% seo %}
</head>
```

(`{% seo %}`는 jekyll-seo-tag가 제공하는 태그. Task 10까지 Chirpy gem이 이 gem을 물고 오므로 지금은 그대로 동작한다.)

- [ ] **Step 2: `_includes/new-nav.html` 작성**

```html
<header class="site-nav">
  <div class="nav-inner">
    <a class="nav-brand" href="{{ '/' | relative_url }}">{{ site.title }}</a>
    <nav class="nav-tabs">
      <a href="{{ '/' | relative_url }}" class="nav-tab{% if page.layout == 'home' %} is-active{% endif %}">Home</a>
      <a href="{{ '/categories/' | relative_url }}" class="nav-tab{% if page.layout == 'categories' or page.layout == 'category' %} is-active{% endif %}">Categories</a>
      <a href="{{ '/tags/' | relative_url }}" class="nav-tab{% if page.layout == 'tags' or page.layout == 'tag' %} is-active{% endif %}">Tags</a>
      <a href="{{ '/archives/' | relative_url }}" class="nav-tab{% if page.layout == 'archives' %} is-active{% endif %}">Archives</a>
      <a href="{{ '/about/' | relative_url }}" class="nav-tab{% if page.layout == 'page' %} is-active{% endif %}">About</a>
    </nav>
  </div>
</header>
```

- [ ] **Step 3: `_includes/new-footer.html` 작성**

```html
<footer class="site-footer">
  <div class="footer-inner">
    <p>&copy; {{ site.time | date: '%Y' }} {{ site.social.name }}</p>
    <p class="footer-links">
      {% for link in site.social.links %}<a href="{{ link }}" target="_blank" rel="noopener">{{ link | remove_first: 'https://' | split: '/' | first }}</a>{% endfor %}
      <a href="mailto:{{ site.social.email }}">{{ site.social.email }}</a>
    </p>
  </div>
</footer>
```

- [ ] **Step 4: 빌드 확인 (아직 어떤 레이아웃도 이 include를 안 부르므로 빌드 성공만 확인)**

Run: `bundle exec jekyll build`
Expected: 에러 없이 성공

- [ ] **Step 5: 커밋**

```bash
git add _includes/new-head.html _includes/new-nav.html _includes/new-footer.html
git commit -m "feat: 새 디자인용 head/nav/footer 인클루드 추가"
```

---

## Task 3: 베이스 레이아웃 + About 페이지

**Files:**
- Create: `_layouts/base.html`
- Create: `_layouts/page.html`
- Test: 빌드 결과 `_site/about/index.html`

**Interfaces:**
- Consumes: Task 2의 `new-head.html`/`new-nav.html`/`new-footer.html`
- Produces: `layout: base` — 이후 모든 새 레이아웃(home/post/categories/tags/archives/category/tag)이 자기 front matter에서 `layout: base`로 상속

- [ ] **Step 1: `_layouts/base.html` 작성**

```html
---
---
<!doctype html>
<html lang="{{ site.lang }}">
{% include new-head.html %}
<body>
{% include new-nav.html %}
<main class="site-main">
  {{ content }}
</main>
{% include new-footer.html %}
</body>
</html>
```

- [ ] **Step 2: `_layouts/page.html` 작성 (Chirpy의 동일 이름 레이아웃을 오버라이드 — about.md가 즉시 새 디자인으로 렌더링됨)**

```html
---
layout: base
---
<article class="page-content">
  <h1 class="page-title">{{ page.title | default: "About" }}</h1>
  <div class="page-body">
    {{ content }}
  </div>
</article>
```

- [ ] **Step 3: 빌드**

Run: `bundle exec jekyll build`
Expected: 에러 없이 성공

- [ ] **Step 4: About 페이지가 새 디자인으로 렌더링됐는지 확인**

Run: `grep -q 'class="page-title"' _site/about/index.html && grep -q 'class="site-nav"' _site/about/index.html && echo PASS`
Expected: `PASS`

- [ ] **Step 5: 브라우저로 육안 확인 (선택)**

`bundle exec jekyll serve` 후 `http://localhost:4000/about/` 접속해서 새 nav·타이포가 적용됐는지 확인. 아직 categories/tags/archives 링크는 Chirpy 옛 디자인으로 뜨는 게 정상(다음 태스크들에서 순차 교체).

- [ ] **Step 6: 커밋**

```bash
git add _layouts/base.html _layouts/page.html
git commit -m "feat: 베이스 레이아웃 및 About 페이지 새 디자인 적용"
```

---

## Task 4: 포스트 카드 & 페이지네이터 인클루드

**Files:**
- Create: `_includes/new-post-card.html`
- Create: `_includes/new-pager.html`

**Interfaces:**
- Consumes: `include.post`(Jekyll post 객체), `paginator`(jekyll-paginate가 주입하는 전역 변수 — `paginator.posts`, `paginator.page`, `paginator.total_pages`, `paginator.previous_page`, `paginator.next_page`, `paginator.previous_page_path`, `paginator.next_page_path`)
- Produces: `{% include new-post-card.html post=post %}`, `{% include new-pager.html %}` — Task 5(home.html)와 Task 7(category.html)에서 그대로 호출

- [ ] **Step 1: `_includes/new-post-card.html` 작성**

```html
<li class="post-card">
  <div class="post-card-date">
    <span class="post-card-day">{{ include.post.date | date: '%d' }}</span>
    <span class="post-card-month">{{ include.post.date | date: '%b' | upcase }}</span>
  </div>
  <div class="post-card-body">
    <div class="post-card-tags">
      {% for tag in include.post.tags %}<span class="post-card-tag">#{{ tag }}</span>{% endfor %}
    </div>
    <h2 class="post-card-title"><a href="{{ include.post.url | relative_url }}">{{ include.post.title }}</a></h2>
    <p class="post-card-excerpt">{{ include.post.excerpt | strip_html | truncate: 90 }}</p>
    <a class="post-card-more" href="{{ include.post.url | relative_url }}">Read More →</a>
  </div>
</li>
```

- [ ] **Step 2: `_includes/new-pager.html` 작성**

```html
{% if paginator.total_pages > 1 %}
<nav class="pager">
  {% if paginator.previous_page %}<a href="{{ paginator.previous_page_path | relative_url }}" class="pager-prev">← Prev</a>{% endif %}
  <span class="pager-count">{{ paginator.page }}/{{ paginator.total_pages }}</span>
  {% if paginator.next_page %}<a href="{{ paginator.next_page_path | relative_url }}" class="pager-next">Next →</a>{% endif %}
</nav>
{% endif %}
```

- [ ] **Step 3: 빌드 확인 (아직 아무도 안 부르므로 빌드 성공만)**

Run: `bundle exec jekyll build`
Expected: 에러 없이 성공

- [ ] **Step 4: 커밋**

```bash
git add _includes/new-post-card.html _includes/new-pager.html
git commit -m "feat: 포스트 카드·페이지네이터 인클루드 추가"
```

---

## Task 5: 홈 레이아웃

**Files:**
- Create: `_layouts/home.html`
- Test: `_site/index.html`

**Interfaces:**
- Consumes: `_includes/new-post-card.html`, `_includes/new-pager.html`, `paginator`
- Produces: 없음 (최종 페이지)

- [ ] **Step 1: `_layouts/home.html` 작성 (Chirpy 동일 이름 레이아웃 오버라이드)**

```html
---
layout: base
---
<section class="home-hero">
  <h1>{{ site.title }}</h1>
  <p class="home-tagline">{{ site.tagline }}</p>
</section>

<ul class="post-list">
  {% for post in paginator.posts %}
    {% include new-post-card.html post=post %}
  {% endfor %}
</ul>

{% include new-pager.html %}
```

- [ ] **Step 2: 빌드**

Run: `bundle exec jekyll build`
Expected: 에러 없이 성공

- [ ] **Step 3: 홈이 새 디자인으로 렌더링됐고 최신 포스트가 리스트에 있는지 확인**

Run: `grep -q 'class="post-card"' _site/index.html && grep -q "AWS" _site/index.html && echo PASS`
Expected: `PASS`

- [ ] **Step 4: 페이지네이션 동작 확인 (포스트가 10개 넘으므로 2페이지 이상 생성됨)**

Run: `test -f _site/page2/index.html && grep -q 'class="pager"' _site/index.html && echo PASS`
Expected: `PASS`

- [ ] **Step 5: 커밋**

```bash
git add _layouts/home.html
git commit -m "feat: 홈(포스트 리스트) 페이지 새 디자인 적용"
```

---

## Task 6: 포스트 상세 레이아웃

**Files:**
- Create: `_layouts/post.html`
- Test: `_site/posts/aws-cloud-computing-basics/index.html` (실제 존재하는 포스트로 검증)

**Interfaces:**
- Consumes: `page.title`, `page.date`, `page.categories`, `page.tags`, `content` (Jekyll 표준 post 변수)
- Produces: 없음 (최종 페이지)

- [ ] **Step 1: `_layouts/post.html` 작성**

```html
---
layout: base
---
<article class="post-detail">
  <header class="post-header">
    <h1 class="post-title">{{ page.title }}</h1>
    <div class="post-meta">
      <time datetime="{{ page.date | date_to_xmlschema }}">{{ page.date | date: '%Y.%m.%d' }}</time>
    </div>
    {% if page.tags %}
      <div class="post-tags">
        {% for tag in page.tags %}{% assign tag_slug = tag | slugify %}<a class="post-tag" href="{{ '/tags/' | append: tag_slug | append: '/' | relative_url }}">#{{ tag }}</a>{% endfor %}
      </div>
    {% endif %}
  </header>
  <div class="post-content">
    {{ content }}
  </div>
</article>
```

- [ ] **Step 2: 빌드**

Run: `bundle exec jekyll build`
Expected: 에러 없이 성공

- [ ] **Step 3: 실제 포스트가 새 디자인으로 렌더링되는지 확인**

Run: `grep -q 'class="post-title"' "_site/posts/aws-cloud-computing-basics/index.html" && grep -q "#aws" "_site/posts/aws-cloud-computing-basics/index.html" && echo PASS`
Expected: `PASS`

- [ ] **Step 4: 이미지가 포함된 포스트 본문이 깨지지 않는지 확인**

Run: `grep -q '<img' "_site/posts/aws-cloud-computing-basics/index.html" && echo PASS`
Expected: `PASS`

- [ ] **Step 5: 커밋**

```bash
git add _layouts/post.html
git commit -m "feat: 포스트 상세 페이지 새 디자인 적용"
```

---

## Task 7: Categories (인덱스 + 개별 카테고리 아카이브)

**Files:**
- Create: `_layouts/categories.html`
- Create: `_layouts/category.html`
- Test: `_site/categories/index.html`, `_site/categories/<슬러그>/index.html`

**Interfaces:**
- Consumes: `site.posts`, `site.categories`(Jekyll 내장 해시), jekyll-archives가 `category.html` 레이아웃에 주입하는 `page.title`(카테고리명), `page.posts`(해당 카테고리 글 목록)
- Produces: 없음 (최종 페이지). `categories.md`가 `layout: categories`를 명시적으로 갖고 있으므로 이 파일이 그 페이지를 오버라이드한다.

**설계 결정:** 기존 Chirpy는 `categories: [백엔드, 스트리밍]`처럼 2~3단계 배열을 사이드바에 계층형으로 그렸다. 뱅크샐러드의 Tech/Culture 탭처럼 단순한 구조로 가기 위해, **인덱스 페이지는 1단계 카테고리(`post.categories[0]`)만 그룹핑**한다. 개별 카테고리 아카이브 URL은 jekyll-archives 설정(`_config.yml`의 `jekyll-archives.permalinks.category: /categories/:name/`)이 `Jekyll::Utils.slugify`로 `:name`을 계산하므로, Liquid의 `slugify` 필터(인자 없이 기본 모드)로 동일하게 링크를 생성한다.

- [ ] **Step 1: `_layouts/categories.html` 작성**

```html
---
layout: base
---
<section class="archive-index">
  <h1 class="archive-title">Categories</h1>
  <ul class="archive-group-list">
    {% assign top_categories = "" | split: "" %}
    {% for post in site.posts %}
      {% assign top = post.categories[0] %}
      {% unless top_categories contains top %}
        {% assign top_categories = top_categories | push: top %}
      {% endunless %}
    {% endfor %}
    {% assign top_categories = top_categories | sort %}
    {% for cat in top_categories %}
      <li class="archive-group">
        {% assign cat_slug = cat | slugify %}<a href="{{ '/categories/' | append: cat_slug | append: '/' | relative_url }}">{{ cat }}</a>
        <span class="archive-group-count">{{ site.categories[cat] | size }}</span>
      </li>
    {% endfor %}
  </ul>
</section>
```

- [ ] **Step 2: `_layouts/category.html` 작성 (jekyll-archives가 카테고리별로 자동 생성하는 페이지)**

```html
---
layout: base
---
<section class="archive-page">
  <h1 class="archive-title">{{ page.title }}</h1>
  <p class="post-meta">{{ page.posts | size }}개의 글</p>
  <ul class="post-list">
    {% for post in page.posts %}
      {% include new-post-card.html post=post %}
    {% endfor %}
  </ul>
</section>
```

- [ ] **Step 3: 빌드**

Run: `bundle exec jekyll build`
Expected: 에러 없이 성공

- [ ] **Step 4: 카테고리 인덱스 확인**

Run: `grep -q "백엔드" _site/categories/index.html && echo PASS`
Expected: `PASS`

- [ ] **Step 5: 개별 카테고리 페이지 URL이 실제로 생성됐는지 확인 (한글 슬러그 검증)**

Run: `find _site/categories -maxdepth 1 -type d`
Expected: 폴더 목록에 "백엔드", "AI", "책-log" 등 슬러그 폴더가 보임. `categories.html`에서 생성한 링크(`{{ cat | slugify }}`)와 실제 폴더명이 일치하는지 눈으로 대조 — 만약 안 맞으면 `category.html`/`categories.html`의 슬러그 계산 방식을 실제 폴더명 기준으로 수정한다.

- [ ] **Step 6: 커밋**

```bash
git add _layouts/categories.html _layouts/category.html
git commit -m "feat: 카테고리 인덱스·개별 카테고리 페이지 새 디자인 적용"
```

---

## Task 8: Tags (인덱스 + 개별 태그 아카이브)

**Files:**
- Create: `_layouts/tags.html`
- Create: `_layouts/tag.html`
- Test: `_site/tags/index.html`, `_site/tags/<슬러그>/index.html`

**Interfaces:**
- Consumes: `site.tags`(Jekyll 내장 해시), jekyll-archives가 `tag.html`에 주입하는 `page.title`(태그명), `page.posts`

- [ ] **Step 1: `_layouts/tags.html` 작성**

```html
---
layout: base
---
<section class="archive-index">
  <h1 class="archive-title">Tags</h1>
  <ul class="tag-cloud">
    {% assign tag_names = site.tags | sort %}
    {% for tag in tag_names %}
      {% assign tag_slug = tag[0] | slugify %}<li><a class="tag-pill" href="{{ '/tags/' | append: tag_slug | append: '/' | relative_url }}">#{{ tag[0] }}<span>{{ tag[1] | size }}</span></a></li>
    {% endfor %}
  </ul>
</section>
```

- [ ] **Step 2: `_layouts/tag.html` 작성**

```html
---
layout: base
---
<section class="archive-page">
  <h1 class="archive-title">#{{ page.title }}</h1>
  <p class="post-meta">{{ page.posts | size }}개의 글</p>
  <ul class="post-list">
    {% for post in page.posts %}
      {% include new-post-card.html post=post %}
    {% endfor %}
  </ul>
</section>
```

- [ ] **Step 3: 빌드**

Run: `bundle exec jekyll build`
Expected: 에러 없이 성공

- [ ] **Step 4: 태그 인덱스 확인**

Run: `grep -q "aws" _site/tags/index.html && echo PASS`
Expected: `PASS`

- [ ] **Step 5: 개별 태그 페이지 존재 확인**

Run: `test -d _site/tags/aws && grep -q 'class="post-card"' _site/tags/aws/index.html && echo PASS`
Expected: `PASS`

- [ ] **Step 6: 커밋**

```bash
git add _layouts/tags.html _layouts/tag.html
git commit -m "feat: 태그 인덱스·개별 태그 페이지 새 디자인 적용"
```

---

## Task 9: Archives

**Files:**
- Create: `_layouts/archives.html`
- Test: `_site/archives/index.html`

**Interfaces:**
- Consumes: `site.posts` (Liquid `group_by_exp`로 연도별 그룹핑)

- [ ] **Step 1: `_layouts/archives.html` 작성**

```html
---
layout: base
---
<section class="archive-index">
  <h1 class="archive-title">Archives</h1>
  {% assign posts_by_year = site.posts | group_by_exp: 'post', 'post.date | date: "%Y"' %}
  {% for year in posts_by_year %}
    <div class="archive-year">
      <h2>{{ year.name }}</h2>
      <ul class="archive-post-list">
        {% for post in year.items %}
          <li>
            <time>{{ post.date | date: '%m.%d' }}</time>
            <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
          </li>
        {% endfor %}
      </ul>
    </div>
  {% endfor %}
</section>
```

- [ ] **Step 2: 빌드**

Run: `bundle exec jekyll build`
Expected: 에러 없이 성공

- [ ] **Step 3: 아카이브 페이지 확인**

Run: `grep -q "2026" _site/archives/index.html && grep -q "class=\"archive-year\"" _site/archives/index.html && echo PASS`
Expected: `PASS`

- [ ] **Step 4: 커밋**

```bash
git add _layouts/archives.html
git commit -m "feat: 아카이브 페이지 새 디자인 적용"
```

---

## Task 10: Chirpy 제거 & 최종 컷오버

**Files:**
- Modify: `Gemfile`
- Modify: `_config.yml`
- Delete: `assets/lib` (git submodule), `.gitmodules`의 해당 항목
- Delete: `_includes/metadata-hook.html` (내용은 이미 Task 2의 `new-head.html`에 네이버 인증 메타태그로 이관 완료, PWA 서비스워커 스크립트는 이번 개편에서 드롭하기로 확정된 기능이라 폐기)
- Test: 전체 사이트 빌드 + html-proofer

**Interfaces:**
- Consumes: Task 1~9에서 만든 모든 레이아웃·인클루드·CSS (더 이상 아무것도 Chirpy에 의존하지 않아야 함)

- [ ] **Step 1: `assets/lib` 서브모듈 제거**

```bash
git submodule deinit -f assets/lib
git rm -f assets/lib
rm -rf .git/modules/assets/lib
```

- [ ] **Step 2: `.gitmodules`에서 `assets/lib` 항목 제거**

`.gitmodules` 파일을 열어 아래 블록을 삭제한다.

```
[submodule "assets/lib"]
	path = assets/lib
	url = https://github.com/cotes2020/chirpy-static-assets.git
```

파일에 다른 서브모듈 항목이 없으면 `.gitmodules` 파일 자체를 삭제해도 된다.

- [ ] **Step 3: `_includes/metadata-hook.html` 삭제**

```bash
git rm _includes/metadata-hook.html
```

- [ ] **Step 4: `Gemfile` 수정 — Chirpy 대신 개별 gem 의존**

`Gemfile`의 `gem "jekyll-theme-chirpy", "~> 7.5"` 줄을 아래로 교체한다.

```ruby
gem "jekyll", "~> 4.4"
gem "jekyll-archives", "~> 2.3"
gem "jekyll-paginate", "~> 1.1"
gem "jekyll-seo-tag", "~> 2.9"
gem "jekyll-sitemap", "~> 1.4"
```

나머지(`html-proofer`, `tzinfo`/`tzinfo-data`, `wdm` 플랫폼 블록)는 그대로 둔다.

- [ ] **Step 5: `_config.yml` 수정**

`theme: jekyll-theme-chirpy` 줄을 삭제하고, 대신 사용할 플러그인을 명시한다 (Chirpy gem이 사라지면 자동으로 로드해주던 플러그인 require가 없어지므로 직접 선언해야 함).

```yaml
plugins:
  - jekyll-archives
  - jekyll-paginate
  - jekyll-seo-tag
  - jekyll-sitemap
```

`pwa:` 블록 전체와 `assets: self_host:` 블록 전체를 삭제한다 (PWA는 이번 개편에서 드롭 확정, self-host는 `assets/lib` 제거로 더 이상 의미 없음). `comments:`, `analytics:`, `webmaster_verifications:` 등 나머지 설정은 그대로 둔다 (2차에서 기능 재추가할 때 재사용).

- [ ] **Step 6: 의존성 재설치**

Run: `bundle install`
Expected: 에러 없이 성공, `Gemfile.lock`에서 `jekyll-theme-chirpy` 항목이 사라짐

- [ ] **Step 7: 전체 사이트 클린 빌드**

```bash
rm -rf _site .jekyll-cache
bundle exec jekyll build
```

Expected: 에러 없이 성공

- [ ] **Step 8: 전체 포스트 개수 검증 (기존 23편 + 초안 1편 = 24편이 모두 빌드됐는지)**

Run: `find _site/posts -mindepth 1 -maxdepth 1 -type d | wc -l`
Expected: `24`

- [ ] **Step 9: 6개 페이지 타입이 전부 새 디자인 마크업을 쓰는지 한 번에 확인**

```bash
for f in _site/index.html _site/about/index.html _site/categories/index.html _site/tags/index.html _site/archives/index.html; do
  grep -q 'class="site-nav"' "$f" && echo "PASS: $f" || echo "FAIL: $f"
done
```

Expected: 5개 파일 모두 `PASS`

- [ ] **Step 10: html-proofer로 깨진 링크·이미지 검사**

Run: `bundle exec htmlproofer ./_site --disable-external`
Expected: 에러 없이 통과 (내부 링크/이미지 경로만 검사, 외부 링크는 네트워크 이슈로 오탐 가능성 있어 제외)

- [ ] **Step 11: 로컬 서버로 최종 육안 확인**

Run: `bundle exec jekyll serve`
`http://localhost:4000/`에서 홈 → 포스트 클릭 → 카테고리 → 태그 → 아카이브 → About 순서로 직접 클릭해보며 깨진 화면 없는지 확인.

- [ ] **Step 12: 커밋**

```bash
git add Gemfile Gemfile.lock _config.yml .gitmodules
git commit -m "feat: Chirpy gem 의존성 제거, 뱅크샐러드 스타일 커스텀 테마로 완전 전환"
```

---

## 완료 후

`.claude/memory/블로그현황.md`에 이번 디자인 개편 완료 사실을 기록한다 (별도 기술 학습이 아니라 인프라성 변경이라 `학습진행도.md`는 건드리지 않는다). 2차 작업(다크모드·Giscus 댓글·PWA 재도입, UI/UX Pro Max 스킬로 포인트 컬러 등 개인화 요소 추가)은 이 플랜 범위 밖이며 별도 브레인스토밍 세션에서 다룬다.
