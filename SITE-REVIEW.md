# Ralph Site v1 — Full Code Review

> **Цель документа:** Полный разбор сайта, созданного за 201 итерацию автономным агентом Ralph. Документ служит учебником для следующей версии — фиксирует что работает, что сломано, и конкретные решения.
>
> **Файл:** `index.html` — 4039 строк (632 CSS + 405 HTML + ~3000 JS), single-file SPA.
> **Стек:** Vanilla JS, CSS, self-hosted Bodoni Moda + IBM Plex Mono.

---

## Оглавление

1. [Общая оценка](#1-общая-оценка)
2. [Что хорошо (сохранить)](#2-что-хорошо-сохранить)
3. [Критические проблемы](#3-критические-проблемы)
4. [CSS: архитектура и проблемы](#4-css-архитектура-и-проблемы)
5. [JavaScript: перформанс и архитектура](#5-javascript-перформанс-и-архитектура)
6. [HTML и семантика](#6-html-и-семантика)
7. [Типографика и читаемость](#7-типографика-и-читаемость)
8. [Адаптивность и мобильная версия](#8-адаптивность-и-мобильная-версия)
9. [Accessibility (a11y)](#9-accessibility-a11y)
10. [SEO](#10-seo)
11. [Контент и визуальная иерархия](#11-контент-и-визуальная-иерархия)
12. [Конкретные рекомендации для v2](#12-конкретные-рекомендации-для-v2)

---

## 1. Общая оценка

| Аспект | Оценка | Комментарий |
|--------|--------|-------------|
| Визуальное впечатление | 7/10 | Сильная типографика, editorial-стиль чувствуется |
| Перформанс | 2/10 | 25+ rAF-лупов, 15 канвасов, лаги на любом устройстве |
| Читаемость текста | 4/10 | Базовый шрифт 11px/300, лейблы 7-9px |
| Мобильная версия | 3/10 | Навигация полностью сломана (display:none без альтернативы) |
| Accessibility | 5/10 | Хороший reduced-motion, но много критичных проблем |
| Качество кода | 3/10 | Дубликаты, 52 !important, z-index хаос, нет модульности |
| SEO | 4/10 | Базовые meta есть, нет OG/Twitter/favicon/structured data |

**Вердикт:** Визуально сайт впечатляет на первый взгляд, но за фасадом — неподдерживаемый код, лаги, и критические UX-проблемы. Ralph накладывал эффект за эффектом 201 итерацию, не удаляя предыдущие. Результат — 3000 строк JS с дублирующейся логикой и 25+ параллельных анимационных лупов.

---

## 2. Что хорошо (сохранить)

### 2.1 Дизайн-система (основа)

```css
:root {
  --bg: #f8f7f4;        /* тёплый off-white */
  --fg: #0c0b09;        /* почти чёрный */
  --muted: #888780;     /* тёплый серый */
  --accent: #8b2e1a;    /* терракота/burnt sienna */
  --display: 'Bodoni Moda', Georgia, serif;
  --mono: 'IBM Plex Mono', 'Courier New', monospace;
  --ease: cubic-bezier(.16,1,.3,1);  /* агрессивный ease-out */
  --pad: clamp(28px, 6vw, 88px);     /* адаптивный отступ */
}
```

**Почему хорошо:** Минимальная палитра (1 accent color), два контрастных шрифта (serif display + mono body), единый easing для coherent motion feel, `--pad` как единый адаптивный gutter. Это сильная основа — сохранить и расширить.

### 2.2 Типографический подход

- **Extreme scale contrast** — 360px заголовки рядом с 10px лейблами. Это именно то, что создаёт editorial feel.
- **clamp()** для всех размеров — fluid typography без JS. Около 50 уникальных `font-size:clamp()`.
- **Consistent easing** — почти все transition используют `var(--ease)`, что даёт единое "ощущение" движения.

### 2.3 Reduced motion support

```css
@media(prefers-reduced-motion:reduce) {
  *, *::before, *::after {
    animation: none !important;
    transition: none !important;
  }
  [data-r] { opacity:1; transform:none; }
}
```

20+ отдельных media query блоков + JS-гард `var reducedMotion = window.matchMedia('(prefers-reduced-motion:reduce)').matches`. Это образцовая реализация — **обязательно сохранить**.

### 2.4 Scroll reveal pattern

```javascript
// Один IntersectionObserver для всех data-r элементов
var rvObs = new IntersectionObserver(function(entries) {
  entries.forEach(function(e) {
    if (e.isIntersecting) { e.target.classList.add('v'); rvObs.unobserve(e.target); }
  });
}, { threshold: 0.02 });
document.querySelectorAll('[data-r]').forEach(function(el) { rvObs.observe(el); });
```

**Почему хорошо:** Один observer вместо десятков scroll listeners, `unobserve` после срабатывания, маленький threshold. Паттерн `data-r` + класс `.v` простой и рабочий.

### 2.5 Отдельные визуальные решения

- **`::selection` styling** — accent color на выделение текста
- **`:focus-visible`** — outline только для клавиатурной навигации
- **`font-display:swap`** — нет FOIT
- **`min-height:100svh` с fallback 100vh** — правильная работа с мобильными браузерами
- **`pointer:coarse`/`pointer:fine`** media queries — разное поведение для touch/mouse
- **SVG stroke-dasharray анимации** — GPU-friendly line-drawing
- **Spring physics** — консистентная формула `velocity = (velocity + (target - current) * k) * friction` во всех анимациях

### 2.6 Content structure

Логичный flow: Hero → About → Work → Statement → Network → Education → Stats → Contact → Footer. От идентичности к деятельности, к аргументам, к контакту.

---

## 3. Критические проблемы

### 3.1 ПЕРФОРМАНС: 25+ параллельных rAF-лупов

Это **главная причина лагов**. На каждом кадре браузер выполняет:

| Луп | Что делает | Останавливается? |
|-----|-----------|-----------------|
| Scroll velocity tracker | CSS-переменные каждый кадр | Нет, всегда |
| Cursor ring follow | Позиция + хроматический сплит | Нет, всегда |
| Trail canvas (DNA helix) | Полноэкранный канвас | Нет, всегда |
| Hero canvas (mesh) | 150 нод, ~252 bezier, частицы | Нет, всегда |
| Hero char repulsion | Физика на каждый символ | Нет, всегда |
| Hero word spring | Пружинная физика | Нет, всегда |
| Liquid distortion | SVG filter mutation | Нет, всегда |
| Statement orbit | Маска радиальный градиент | Нет, всегда |
| Statement canvas | До 864 частиц | Нет, всегда |
| Network canvas | 1296 звёзд + 9 нод + 21 ребро | Нет, всегда |
| Flow canvas (tlude) | Bezier линии | Нет, всегда |
| Work panel canvas | 4 визуализации | Нет, всегда |
| Bio Venn bg | 3 круга | Да, IO |
| Bio body warmth | CSS-переменная | Нет, всегда |
| Bio lead spotlight | CSS-переменная | Нет, всегда |
| Stats canvas | Data flow | Да, IO |
| Edu constellation | Хабы + сателлиты | Да, IO |
| DNA helix canvas | Спираль | Нет, всегда |
| Shockwave overlay | Ударная волна | Нет, всегда |
| Powerful liquid filter | SVG feTurbulence | Нет, всегда |
| Gravity field canvas | ~836 точек fullscreen | Нет, всегда |
| Footer char spring | Физика символов | Нет, всегда |
| Bio word drift | Физика слов | Нет, всегда |
| Tlude char attraction | Физика символов | Нет, всегда |
| Magnetic CTA | Пружины для ссылок | Нет, всегда |
| + дубль magnetic buttons | То же самое! | Нет, всегда |

**Бюджет на 60fps: 16.6ms на кадр. С 25+ лупами это невозможно.**

**Решение для v2:**
- Один master rAF loop, вызывающий подсистемы
- Все канвасы за IntersectionObserver — рисовать только когда видны
- Максимум 3-4 канваса одновременно
- Dirty-flag: если ничего не изменилось — не перерисовывать

### 3.2 МОБАЙЛ: Навигация сломана

```css
@media(max-width:560px) { .nav-links { display:none } }
```

**Нет гамбургер-меню, нет альтернативы.** Мобильные пользователи видят только "Matvei Tkhai" в шапке и должны скроллить вслепую.

### 3.3 КУРСОР: Скрыт без fallback

```css
@media(pointer:fine) { *, *::before, *::after { cursor:none !important } }
```

Если JS не загрузится — пользователь остаётся без курсора. Нужен fallback timeout.

### 3.4 НЕВАЛИДНЫЙ HTML: Дублированные ID

Два элемента с `id="grain"` (div на строке 636 и svg на строке 648). `getElementById` найдёт только первый — SVG grain filter может не работать.

---

## 4. CSS: архитектура и проблемы

### 4.1 Архитектура

632 строки CSS в одном `<style>` блоке. Организация по секциям (hero, bio, work, stmt...), но деградирует после строки ~186: новые фичи просто append-ились в конец без рефакторинга.

### 4.2 Хардкод цветов вместо переменных

Определён `--accent:#8b2e1a`, но в CSS **135 раз** написано `rgba(139,46,26,...)` вместо использования переменной. Аналогично:
- `rgba(12,11,9,...)` (--fg с альфой) — **76 раз**
- `rgba(248,247,244,...)` (--bg с альфой) — **75 раз**

**Решение для v2:** Использовать `color-mix()` или определить alpha-варианты:
```css
:root {
  --accent-20: color-mix(in srgb, var(--accent) 20%, transparent);
  --fg-10: color-mix(in srgb, var(--fg) 10%, transparent);
}
```

### 4.3 Дублированные селекторы

| Селектор | Строки | Проблема |
|----------|--------|----------|
| `#grain` | 229, 620 | Разные z-index (500 vs 99995) и opacity (.12 vs .45) |
| `.strip` | 122, 241 | Конфликтующие padding и flex-direction |
| `.stmt-wi` | 505, 527 | Второе определение расширяет первое, но нечитаемо |

### 4.4 Z-index хаос

Значения от 0 до **100000** без системы:

```
0         → канвасы, фоны
1-10      → контент, оверлеи
49-100    → навигация
400-900   → подсказки, прогресс
9985-9999 → курсор
10000     → терминал
99995     → grain (второй!)
100000    → entry curtain
```

**Решение для v2:** Токены:
```css
:root {
  --z-base: 0;
  --z-content: 1;
  --z-overlay: 10;
  --z-nav: 100;
  --z-cursor: 500;
  --z-modal: 1000;
}
```

### 4.5 52 × `!important`

Основная концентрация в work horizontal scroll (строки 428-492), где горизонтальная раскладка борется с вертикальными стилями тех же классов. Это архитектурная проблема — нужны отдельные компоненты вместо override войны.

### 4.6 `will-change` overuse

25 объявлений `will-change` в CSS. Каждое создаёт отдельный compositor layer, потребляя GPU-память.

**Критичные ошибки:**
- `will-change:background` (строки 234, 300) — **не работает!** Только `transform` и `opacity` реально compositable
- `will-change:transform` на каждом символьном `<span>` (`.tc-c`, `.fc`, `.ht-c`) — десятки compositor layers на символы

**Решение:** Добавлять `will-change` через JS класс перед анимацией, убирать после.

### 4.7 Breakpoints без системы

7 разных breakpoints: 480, 560, 600, 660, 700, 760, 900px. Можно объединить в 3: `sm:480`, `md:768`, `lg:1024`.

### 4.8 Тяжёлые CSS-анимации

| Анимация | Проблема |
|----------|----------|
| `bioSpin` (108s infinite) | Вращение conic-gradient на min(62vw,800px) элементе |
| `powerfulBloom` | filter:blur(14px) + perspective + scale |
| `inkBlob` | filter:blur(110px) на полном разрешении |
| `termScan` (0.18s steps(4)) | CRT-scanline перерисовка viewport 22 раза/сек |
| `fromWithinGlow`, `stmtLum` | text-shadow анимации — дорогие |

---

## 5. JavaScript: перформанс и архитектура

### 5.1 Архитектура

~3000 строк в одном `<script>`. Основа — IIFE (строка 1041), но **код после строки 3781 вне IIFE** — создаёт implicit globals.

Внутри — ~60 вложенных IIFE, каждая для одной фичи. Нет модулей, нет классов.

Глобальный state через `window._hBurst`, `window._hBurstPing`, `window._hClockPing`, `window._setLqIntro`, `window._stmtBurst`, `window._hFirstMove`.

### 5.2 Scroll listeners: 18 штук без координации

Каждый работает независимо, многие вызывают `getBoundingClientRect()`:

```javascript
// Строка 1070 — ДУБЛИРУЕТ IntersectionObserver со строки 1068!
window.addEventListener('scroll', function() {
  document.querySelectorAll('[data-r]:not(.v)').forEach(function(el) {
    var r = el.getBoundingClientRect();
    if (r.top < window.innerHeight && r.bottom > 0) el.classList.add('v');
  });
}, {passive:true});
```

**Этот listener полностью дублирует IO и должен быть удалён.**

### 5.3 `getBoundingClientRect()` в горячих путях

**Это вторая главная причина лагов.** Вызов `getBoundingClientRect()` форсирует synchronous layout:

| Место | Когда вызывается | Сколько элементов |
|-------|-----------------|-------------------|
| Bio char repulsion (1118) | Каждый mousemove | 50-100+ символов |
| Bio word drift (1137) | Каждый mousemove | 20-30 слов |
| Strip spans (1326) | Каждый mousemove | 10-20 spans |
| Hero chars (1976) | Каждый rAF кадр! | 30-50 символов |
| Footer chars (2851) | Каждый rAF кадр! | 20-30 символов |
| Tlude chars (2923) | Каждый rAF кадр! | 20-30 символов |
| Bio lead words (3645) | Каждый rAF кадр! | 5-10 слов |
| Magnetic buttons (3717) | Каждый rAF кадр! | 5-10 ссылок |
| Contact chars (3885) | Каждый rAF кадр! | 20-30 символов |

**Решение для v2:** Кешировать позиции, обновлять только на scroll/resize:
```javascript
let cachedRects = new Map();
function updateCache() {
  elements.forEach(el => cachedRects.set(el, el.getBoundingClientRect()));
}
// Обновлять на resize (с debounce) и scroll (с throttle)
```

### 5.4 Canvas: 15 штук, большинство всегда активны

**Самые тяжёлые:**

1. **Gravity field** (`#gf-cv`) — ~836 точек с spring-физикой + O(N²) constellation connections на КАЖДОМ кадре на ПОЛНОМ экране
2. **Trail canvas** — DNA helix: 160 точек, 2 strand, кольца, искры — каждый кадр
3. **Hero mesh** — 150 нод, 252 bezier, 15 текстовых лейблов, N² lightning arcs
4. **Statement** — до 864 частицы (!!), не использует clearRect
5. **Network** — 1296 звёзд + 9 нод + 21 ребро + radial gradients каждый кадр

**Из 15 канвасов только 3 имеют visibility gating через IntersectionObserver.** Остальные 12 рисуют всегда.

### 5.5 Дублированная логика

| Функционал | Количество реализаций | Строки |
|-----------|----------------------|--------|
| Magnetic buttons | 3 (!) | 2866-2887, 3699-3728, 4006-4035 |
| Nav active tracker | 2 | 2476-2484, 3489-3501 |
| Stats 3D tilt | 2 | 2750-2753, 3669-3697 |
| Scroll [data-r] reveal | 2 (IO + scroll listener) | 1068 + 1070 |

### 5.6 createRadialGradient каждый кадр

В hero canvas, network canvas и work panel canvas — `createRadialGradient()` вызывается на КАЖДОМ кадре для каждой ноды. Это аллоцирует новые объекты. Нужно кешировать.

### 5.7 SVG filter mutation каждый кадр

```javascript
// Строки 1942-1943 — каждый кадр!
lqDmap.setAttribute('scale', ...);
lqTurb.setAttribute('seed', ...);
```

Изменение атрибутов SVG filter инвалидирует кеш фильтра и форсит рекомпозицию.

### 5.8 Resize handlers: 16 штук без debounce

Все 16 resize handlers срабатывают одновременно, каждый запрашивает `offsetWidth/offsetHeight`.

**Решение для v2:**
```javascript
let resizeTimer;
window.addEventListener('resize', () => {
  clearTimeout(resizeTimer);
  resizeTimer = setTimeout(() => {
    const W = window.innerWidth;
    const H = window.innerHeight;
    // Обновить все подсистемы с новыми размерами
    subsystems.forEach(s => s.onResize(W, H));
  }, 150);
});
```

### 5.9 Mousemove handlers: 25+ штук

Раздельные mousemove listeners на document, hero, bio, bio-lead, bio-body, stmt, tlude, strip, edu, net-sec, stats, footer, contact... Некоторые используют rAF-gating, многие — нет.

### 5.10 Memory concerns

- `inkDrops` — нет cap, только fade removal. Быстрые клики = рост массива
- `hPings` — нет cap, только age filter
- `setInterval` (clock, terminal clock, grain) — никогда не очищаются
- `statBurst()` создаёт 12 DOM-элементов на body при каждом вызове

---

## 6. HTML и семантика

### 6.1 Структура секций

```
<body>
  ├── Decorative overlays (canvas, grain, curtains, cursor)
  ├── <nav> — основная навигация
  ├── <section> Hero
  ├── <section> Bio (About)
  ├── <section> Work
  ├── <div> Work Bridge (декоративный)
  ├── <div> Tlude (декоративный, aria-hidden)
  ├── <div> Statement scroll container
  │   └── <section> Statement
  ├── <div> Strip (marquee)
  ├── <section> Network
  ├── <section> Education
  ├── <section> Stats
  ├── <div> Vertical Ticker
  ├── <section> Contact
  ├── <footer>
  └── <nav> Section indicator dots
```

### 6.2 Проблемы

- **Нет `<main>`** — нет основного landmark для screen readers
- **Heading hierarchy неполная:** h1 (Matvei Tkhai) → h2 (Work, Education, Contact) — но нет h2 для About, Stats, Network
- **`<nav aria-hidden="true">`** (строка 1032) — landmark с aria-hidden — противоречие
- **Bio label — `<div>`, не heading** (строка 716): используется как `aria-labelledby` target, но это не `<h2>`
- **Blockquote без cite** (строка 893): используется для личного statement, а не цитаты
- **320vh scroll container** для statement (строка 106): 3.2 viewport heights на одну фразу — враждебный UX

---

## 7. Типографика и читаемость

### 7.1 Проблемные размеры

| Элемент | Размер | Вес | Проблема |
|---------|--------|-----|----------|
| Body text | 11px | 300 (light) | Ниже порога читаемости |
| Hero meta | 9px | normal | Почти нечитаем |
| Bio label | 9px | normal | Мелко |
| Bio stat label | 8px | normal | Самый мелкий текст на странице |
| Work org | 9px, opacity 0.3 | normal | Контраст ~1.9:1 — FAIL |
| Edu status | 9px, opacity 0.3 | normal | Контраст ~1.9:1 — FAIL |
| Section labels | 7-9px | normal | Ниже всех рекомендаций |

### 7.2 Контраст

| Текст | Цвет на фон | Ratio | WCAG AA |
|-------|-------------|-------|---------|
| Body (#0c0b09 на #f8f7f4) | 18:1 | PASS ✓ |
| Muted (#888780 на #f8f7f4) | 3.4:1 | FAIL ✗ (нужно 4.5:1) |
| Accent (#8b2e1a на #f8f7f4) | 6.7:1 | PASS ✓ |
| Work org (rgba 0.3 на cream) | ~1.9:1 | FAIL ✗ |
| Dark labels (rgba .25 на dark bg) | ~2:1 | FAIL ✗ |

### 7.3 Рекомендации для v2

- **Минимум 13-14px** для body text
- **Минимум 11px** для самых мелких лейблов
- **font-weight:400** (regular) вместо 300 (light) для body
- **Muted text: поднять до 4.5:1 ratio** — например `#737269` вместо `#888780`
- line-height .82 на display text обрезает descenders — минимум .88

---

## 8. Адаптивность и мобильная версия

### 8.1 Навигация — полностью сломана

На экранах < 560px видно только "Matvei Tkhai". Нет гамбургер-меню, drawer, или любой альтернативы. Пользователь должен скроллить вслепую.

### 8.2 Breakpoints — 7 разных без системы

480, 560, 600, 660, 700, 760, 900px — хаотичный набор. 560-700px можно объединить.

### 8.3 Что скрывается на мобильном

Правильно скрыто (декоративное, экономит перформанс):
- Bio Venn diagram (760px)
- Hero vertical mark (900px)
- DNA helix canvas (760px)
- Section progress gauge (760px)
- Vertical ticker (760px)
- Work panel sparklines (760px)
- Contact signal radar (900px)

### 8.4 Work section

Horizontal scroll carousel корректно деградирует в стековую раскладку на мобильном (строка 481) — но override блок содержит 25+ `!important` свойств в одной строке.

---

## 9. Accessibility (a11y)

### 9.1 Что хорошо

- `aria-hidden="true"` на всех декоративных элементах (canvas, watermarks, folio numbers, cursor)
- `aria-label` на секциях (network, stats, contact, statement)
- `aria-labelledby` на bio и work секциях
- `:focus-visible` с outline (строка 23)
- `rel="noopener"` на всех внешних ссылках
- Touch targets 44px minimum на nav links, CTAs, footer links
- `prefers-reduced-motion` — образцовая реализация

### 9.2 Что плохо

- **Нет skip-to-content link** — нет способа перепрыгнуть hero и декоративные элементы
- **Нет `<main>` landmark**
- **Навигация скрыта на мобильном без альтернативы**
- **Курсор скрыт без JS fallback**
- **Muted text не проходит WCAG AA** (3.4:1 при требовании 4.5:1)
- **Label text 7-9px** — ниже любых рекомендаций по читаемости
- **`<nav aria-hidden="true">`** — landmark, скрытый от AT

---

## 10. SEO

### 10.1 Что есть

- `<title>Matvei Tkhai</title>` — присутствует, но неинформативный
- `<meta name="description">` — хороший текст
- `<html lang="en">` — корректно
- Heading hierarchy: h1 → h2 → h3 (частично)

### 10.2 Что отсутствует

- Open Graph tags (`og:title`, `og:description`, `og:image`)
- Twitter Card meta tags
- `<link rel="canonical">`
- Structured data (JSON-LD Person schema)
- `<link rel="icon">` / favicon
- Полная heading hierarchy (нет h2 для About, Stats, Network)
- Sitemap
- Robots.txt

### 10.3 Рекомендации для v2

```html
<meta property="og:title" content="Matvei Tkhai — VC Researcher & Venture Builder">
<meta property="og:description" content="...">
<meta property="og:image" content="...">
<meta property="og:type" content="website">
<meta name="twitter:card" content="summary_large_image">
<link rel="canonical" href="https://...">
<link rel="icon" href="data:image/svg+xml,...">

<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Person",
  "name": "Matvei Tkhai",
  "jobTitle": "VC Researcher",
  "affiliation": { "@type": "Organization", "name": "HSE University" },
  "sameAs": ["https://linkedin.com/in/...", "https://x.com/motapotav"]
}
</script>
```

---

## 11. Контент и визуальная иерархия

### 11.1 Контент — корректен

- Matvei описан как "part of the core team" (не founder) — ✓
- School 21 — "recently invited, likely bioinformatics → neurobiology" — ✓
- ISSEK HSE research — ✓
- Ссылки LinkedIn и X — корректны

### 11.2 Избыточность

- LinkedIn и X повторяются **5 раз**: nav, hero CTAs, bio socials, contact, footer
- Folio numbers ("01", "02"...) в каждой секции — декоративно, но занимает пространство
- Tlude секция — полный viewport height на "Capital flows everywhere" — чистый padding

### 11.3 Statement scroll-jacking

320vh контейнер на одну фразу. Пользователь скроллит через 3.2 экрана для одной цитаты. Это враждебный UX — большинство пользователей не поймут, что нужно продолжать скроллить.

### 11.4 Отсутствует

- Email / контактная форма — только LinkedIn и X
- Описание проектов — только названия и одна строка описания
- Нет ссылок на публикации, исследования, материалы

---

## 12. Конкретные рекомендации для v2

### 12.1 Перформанс (приоритет #1)

1. **Один master rAF loop** вместо 25+ отдельных:
```javascript
const systems = [];
function masterLoop(t) {
  for (const sys of systems) {
    if (sys.active) sys.update(t);
  }
  requestAnimationFrame(masterLoop);
}
```

2. **Все канвасы за IntersectionObserver** — рисовать ТОЛЬКО когда видны
3. **Кешировать getBoundingClientRect()** — обновлять на resize/scroll, не каждый кадр
4. **Максимум 3-4 канваса** одновременно (hero OR network OR edu, не все сразу)
5. **Один scroll listener**, один resize listener с debounce, один mousemove listener
6. **Убрать gravity field canvas** — самый дорогой эффект с минимальной отдачей
7. **Убрать trail canvas DNA** — дорогой fullscreen эффект
8. **Statement canvas: clearRect вместо fillRect fade** — не накапливать draw state

### 12.2 CSS архитектура

1. **Расширить CSS custom properties:**
```css
:root {
  /* Colors with alpha */
  --accent-10: color-mix(in srgb, var(--accent) 10%, transparent);
  --accent-20: color-mix(in srgb, var(--accent) 20%, transparent);
  --fg-10: color-mix(in srgb, var(--fg) 10%, transparent);
  --fg-40: color-mix(in srgb, var(--fg) 40%, transparent);

  /* Spacing scale */
  --space-xs: clamp(4px, 0.5vw, 8px);
  --space-sm: clamp(8px, 1.5vw, 16px);
  --space-md: clamp(16px, 3vw, 32px);
  --space-lg: clamp(32px, 6vw, 64px);
  --space-xl: clamp(64px, 12vw, 128px);

  /* Z-index tokens */
  --z-base: 0;
  --z-content: 1;
  --z-overlay: 10;
  --z-nav: 100;
  --z-cursor: 500;
  --z-modal: 1000;

  /* Breakpoints (for reference) */
  /* sm: 480px, md: 768px, lg: 1024px */
}
```

2. **Нулевой `!important`** — если нужен, архитектура неправильная
3. **`will-change` только через JS** перед анимацией, убирать после
4. **3 breakpoints максимум**

### 12.3 JS архитектура

1. **Модульная структура** (даже в одном файле):
```javascript
// === CONFIG ===
const CONFIG = { ... };

// === UTILS ===
function observeOnce(el, cb, opts) { ... }
function springPhysics(current, target, velocity, k, friction) { ... }
function debounce(fn, ms) { ... }

// === ANIMATION COORDINATOR ===
class AnimationCoordinator {
  register(system) { ... }
  update(timestamp) { ... }
}

// === SYSTEMS ===
class HeroSystem { ... }
class BioSystem { ... }
class WorkSystem { ... }
```

2. **Нет дубликатов** — одна magnetic button функция, один nav tracker, один stats tilt
3. **Кеш DOM-ссылок** — все querySelector в init, не в event handlers

### 12.4 UX

1. **Мобильная навигация** — hamburger menu или bottom nav
2. **Skip-to-content link**
3. **Cursor fallback** — если JS не загрузился за 3 секунды, показать системный курсор
4. **Statement section** — максимум 150vh вместо 320vh, или убрать scroll lock
5. **Body text минимум 13px, weight 400**
6. **Все label text минимум 11px**
7. **Muted text с контрастом ≥ 4.5:1**

### 12.5 HTML

1. Добавить `<main>`
2. Полная heading hierarchy: h2 для About, Stats, Network
3. Skip-to-content link: `<a href="#main" class="skip-link">Skip to content</a>`
4. Уникальные ID
5. SEO meta tags (OG, Twitter, structured data, favicon)

---

## Итого: что взять в v2, что выбросить

### Взять ✓

- Палитра и CSS-переменные (расширить)
- Bodoni Moda + IBM Plex Mono
- clamp() для fluid typography
- Единый easing `cubic-bezier(.16,1,.3,1)`
- `prefers-reduced-motion` подход
- IntersectionObserver + `data-r` reveal
- `pointer:coarse/fine` разделение
- `font-display:swap`
- Spring physics формула
- Editorial feel, extreme scale contrast
- Content structure (Hero → About → Work → Statement → Contact)

### Выбросить ✗

- Gravity field canvas (дорого, мало отдачи)
- Trail canvas DNA helix (дорого, отвлекает)
- 864 particles в statement (сократить до 100-200)
- 1296 звёзд в network (сократить до 200-300)
- Character-level splitting с physics на каждый символ (оставить только для 1-2 элементов)
- Terminal mode easter egg (интересно, но 200+ строк на скрытую фичу)
- Tlude секция (чистый декоративный padding)
- 320vh statement scroll lock (сократить или убрать)
- Duplicate magnetic buttons (одна функция)
- Folio watermarks в каждой секции (оставить 1-2)
- Vertical ticker (мало информационной ценности)
- `cursor:none` без fallback
- 7-9px label text
- 11px/300 body text
