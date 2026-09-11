# Render stack for meeting reports: HTML, PDF, SVG, Mermaid

Research for issue `09-render-stack.md`. Scope: pick one Python-first rendering
stack that produces HTML, PDF, embedded SVG/Mermaid diagrams, and a rasterized
PNG for email, running headless in a Docker container on a Linux GPU cluster
node with no display server.

## 1. HTML → PDF: WeasyPrint vs Playwright/Chromium

### WeasyPrint

- WeasyPrint is a pure "HTML/CSS to PDF" library — "a smart solution helping
  web developers to create PDF documents" — free/open source, ~25M downloads/month,
  15 years of development. ([weasyprint.org](https://weasyprint.org/))
- Core Python dependencies: `pydyf`, `cffi`, `tinyhtml5`, `tinycss2`,
  `cssselect2`, `Pyphen`, `Pillow`, `fontTools`, plus the system library
  **Pango ≥ 1.44.0** for text shaping/layout. Requires Python ≥ 3.10.
  ([doc.courtbouillon.org/weasyprint/stable/first_steps.html](https://doc.courtbouillon.org/weasyprint/stable/first_steps.html))
- On Debian/Ubuntu, installing the wheel additionally requires the system
  packages `libpango-1.0-0`, `libpangoft2-1.0-0`, `libharfbuzz-subset0`
  (Alpine and Fedora have their own equivalent lists). No browser engine,
  no Node, no GPU/display needed — it's a layout engine, not a browser.
  ([doc.courtbouillon.org/weasyprint/stable/first_steps.html](https://doc.courtbouillon.org/weasyprint/stable/first_steps.html))
- Fonts are resolved through **Fontconfig** (system font matching), and
  WeasyPrint automatically subsets embedded fonts; `@font-face` (web fonts)
  and OpenType features are supported. If a glyph isn't covered by any
  installed/`@font-face` font it renders as a box — "you have to install
  fonts and make them available to WeasyPrint."
  ([doc.courtbouillon.org/weasyprint/stable/first_steps.html](https://doc.courtbouillon.org/weasyprint/stable/first_steps.html),
  [doc.courtbouillon.org/weasyprint/stable/features.html](https://doc.courtbouillon.org/weasyprint/stable/features.html))
- Norwegian æøå are ordinary Latin-1 Supplement / Latin Extended-A
  codepoints covered by virtually every general-purpose font (DejaVu,
  Noto, Liberation, etc.), so they are not a special case for WeasyPrint —
  they only fail if no font in the container covers Latin extended glyphs.
  The GitHub issues that surface as "unicode broken" in WeasyPrint concern
  narrower edge cases: full (non-subset) font embedding
  ([Kozea/WeasyPrint#359](https://github.com/Kozea/WeasyPrint/issues/359)),
  WOFF fonts not being embedded
  ([Kozea/WeasyPrint#1237](https://github.com/Kozea/WeasyPrint/issues/1237)),
  and rare/unsupported glyph ranges producing `.notdef`
  ([Kozea/WeasyPrint#2868](https://github.com/Kozea/WeasyPrint/issues/2868)) —
  not everyday Latin diacritics. Practical takeaway: ship a Unicode-complete
  font (e.g. Noto Sans) in the image/`@font-face` rather than relying on
  whatever the base image happens to have.
- SVG: WeasyPrint renders embedded `<img src=svg>` / inline SVG through its
  own SVG engine, and the docs explicitly flag that SVG rendering "suffers
  from the same problems" as HTML/CSS rendering (i.e. same security/feature
  caveats, not a separate broken code path).
  ([doc.courtbouillon.org/weasyprint/stable/features.html](https://doc.courtbouillon.org/weasyprint/stable/features.html))
- Page headers/footers: supported via CSS Paged Media (`@page`, margin
  boxes, running elements) — this is a first-class WeasyPrint feature set
  (part of its CSS 2.1 + Paged Media support) documented under
  "Notable Features" (PDF bookmarks/forms, PDF/A, PDF/UA, hyphenation).
  ([doc.courtbouillon.org/weasyprint/stable/features.html](https://doc.courtbouillon.org/weasyprint/stable/features.html))
- No official first-party Docker image; a community image is maintained by
  Luca Vercelli.
  ([doc.courtbouillon.org/weasyprint/stable/first_steps.html](https://doc.courtbouillon.org/weasyprint/stable/first_steps.html))

### Playwright (Python) driving headless Chromium

- Official Python bindings; typical setup is `pip install playwright` (or
  `pytest-playwright` for the test runner) followed by `playwright install`
  to download the Chromium/WebKit/Firefox binaries.
  ([playwright.dev/python/docs/intro](https://playwright.dev/python/docs/intro))
- Officially supported Linux targets are glibc-based: Debian 12/13, Ubuntu
  22.04/24.04/26.04 (x86-64 or arm64) — **Alpine is not supported** because
  Firefox/WebKit builds need glibc.
  ([playwright.dev/python/docs/intro](https://playwright.dev/python/docs/intro),
  [playwright.dev/python/docs/docker](https://playwright.dev/python/docs/docker))
- Microsoft publishes an official Docker image (Microsoft Artifact
  Registry) with "the Playwright browsers and browser system dependencies"
  preinstalled, based on Ubuntu 22.04/24.04/26.04. For headless Chromium,
  the docs recommend `--ipc=host` ("Without it, Chromium can run out of
  memory and crash") and, for untrusted content, a non-root user plus a
  seccomp profile for sandboxing.
  ([playwright.dev/python/docs/docker](https://playwright.dev/python/docs/docker))
- Because it drives a real browser engine, Chromium via Playwright gets
  "browser-grade" text shaping, embedded-SVG, web-font, and print-CSS
  fidelity for free — but at the cost of shipping a full Chromium binary
  plus its native library dependency set (fonts, X11/graphics libs used
  headlessly) in the image, i.e. several hundred MB extra vs. WeasyPrint's
  Pango/Cairo footprint, and a browser process to manage (crashes, zombie
  processes, `--ipc=host` tuning) instead of a plain Python call.

### Verdict for this axis

WeasyPrint is the lower-moving-parts choice: no browser process, no
`--ipc=host`/sandboxing concerns, just Pango/Cairo system libraries plus a
pip install. Norwegian characters, inline SVG and web fonts are handled by
its own supported feature set as long as a Unicode-complete font is present
in the image; page headers/footers are a native CSS Paged Media feature.
Playwright/Chromium gives strictly higher HTML/CSS fidelity (it's a real
browser) but pulls in a full browser runtime and its container/hardening
concerns, which is unnecessary weight for a report that is standard HTML
laid out for print.

## 2. Mermaid → SVG from Python

| Option | Needs | Notes |
|---|---|---|
| **mermaid-cli (`mmdc`)** | Node.js + Puppeteer/Chromium under the hood | Official CLI: `npm install -g @mermaid-js/mermaid-cli`, invoked as a subprocess from Python; "takes a mermaid definition file as input and generates an SVG/PNG/PDF file as output" via Puppeteer's headless browser. Ships a Docker image (`minlag/mermaid-cli` on Docker Hub/GHCR) with Chromium baked in. Adds a full Node + headless-Chromium toolchain alongside the Python one. ([github.com/mermaid-js/mermaid-cli](https://github.com/mermaid-js/mermaid-cli)) |
| **mermaid-py** | Pure Python wrapper, but rendering happens by embedding mermaid.js and is "only works in notebooks that rendered the HTML" | `pip install mermaid-py`; it's an interface around the mermaid-js *script*, targeted at Jupyter display, not a standalone headless SVG-file generator. ([pypi.org/project/mermaid-py](https://pypi.org/project/mermaid-py/0.2.6), via search) |
| **python_mermaid / pymermaid / py2mermaid** | Vary | These are thin helpers to *build/emit* Mermaid diagram *text* or to fetch images from Jupyter; `pymermaid` (PyPI, "get images output from mermaid markdown tags") hasn't been updated since June 2021 — stale, notebook-oriented, none of them are self-contained headless renderers. (search results: [pypi.org/project/pymermaid](https://pypi.org/project/pymermaid/), [pypi.org/project/python_mermaid](https://pypi.org/project/python_mermaid/)) |
| **mermaid.ink (hosted service)** | Network call to a public server | Free public API: encode the diagram (same encoding as the Mermaid Live Editor URL) and GET `https://mermaid.ink/svg/<encoded>` (also `/img/`, `/pdf/`) with theme/size/background query params. Zero local toolchain, but requires outbound internet access and trusting a third-party service with report content — usually unacceptable for an internal meeting-report pipeline on a cluster node, and a bad dependency for something that must render deterministically offline. ([mermaid.ink](https://mermaid.ink/)) |
| **Kroki (self-hosted)** | Docker container(s) | Kroki provides "a unified API" across many diagram languages including Mermaid; self-host with `docker run -d -p 8000:8000 yuzutech/kroki` (companion containers needed for some diagram types, orchestrated via Docker Compose). This still renders Mermaid via a headless-Chromium-backed companion service internally — it moves the Node/Chromium toolchain into a *separate* container rather than removing it. ([github.com/yuzutech/kroki](https://github.com/yuzutech/kroki)) |
| **Headless Playwright + bundled mermaid.js** | Playwright + Chromium (already needed if choosing Playwright for HTML→PDF) | Load a minimal HTML page with `mermaid.js` from a local/CDN bundle, call `mermaid.render()` in-page via `page.evaluate`, pull the resulting SVG string out of the DOM. No Node/npm needed (mermaid.js runs inside Playwright's own Chromium), but it does require the Chromium runtime — the same cost already paid if Playwright is chosen for PDF generation, redundant if WeasyPrint is chosen instead. |

Every non-trivial Mermaid-to-SVG path other than the hosted mermaid.ink
service ultimately depends on a headless Chromium instance, because
mermaid.js is a browser-JS library that needs a DOM/canvas/layout engine to
compute diagram geometry — there is no maintained pure-Python Mermaid
renderer. This is confirmed by mermaid-cli's own dependency on
Puppeteer/Chromium being the *official* rendering path.
([github.com/mermaid-js/mermaid-cli](https://github.com/mermaid-js/mermaid-cli))

## 3. SVG → PNG rasterization (for email)

- **cairosvg**: "an SVG converter based on Cairo. It can export SVG files to
  PDF, EPS, PS, and PNG files"; LGPL, Python ≥ 3.10, maintained by Kozea
  (same org as WeasyPrint) with CourtBouillon support — i.e. it reuses the
  same Cairo/Pango system libraries WeasyPrint already needs, so choosing
  WeasyPrint + cairosvg shares one system-dependency footprint instead of
  two. `pip install cairosvg`. ([cairosvg.org](https://cairosvg.org/), README via [github.com/Kozea/CairoSVG](https://raw.githubusercontent.com/Kozea/CairoSVG/master/README.rst))
- **resvg-py** (`resvg_py` on PyPI): Python binding (via PyO3) around the
  Rust `resvg`/`usvg` libraries; "zero-copy Rust rendering... no Python GIL
  bottleneck," supports fonts/DPI/zoom/CSS, ships prebuilt wheels for
  Linux/Windows/macOS/Android so **no Rust toolchain or system Cairo/Pango
  needed** — a single self-contained wheel. Simple API:
  `resvg_py.svg_to_bytes(svg_string=...)`. This is arguably lighter-weight
  than cairosvg if SVG rasterization were the *only* need, but since
  WeasyPrint already requires Cairo/Pango in the image, cairosvg adds zero
  new system dependencies while resvg-py would add a second, separate SVG
  engine. (search results: [pypi.org/project/resvg_py](https://pypi.org/project/resvg_py/), [github.com/briceyan/resvg-py](https://github.com/briceyan/resvg-py))
- **svglib**: "a pure-Python library for reading and converting SVG," built
  on **ReportLab** + **lxml**; converts to ReportLab Drawing objects, then
  to PDF/EPS/bitmap via a `svg2pdf` CLI. Documented limitations: CSS support
  is "still limited," `@import` is ignored, `clipPath`/`mask`/
  `foreignObject` support is partial/absent. Weaker fidelity than
  cairosvg/resvg for anything beyond simple diagram SVGs (which Mermaid
  output mostly is, but still a fidelity risk).
  ([pypi.org/project/svglib](https://pypi.org/project/svglib/))
- **Playwright screenshot**: `page.screenshot()` on a page containing the
  SVG — full browser-grade fidelity, but only sensible if Chromium is
  already resident in the container for another reason (e.g. Mermaid
  rendering via bundled mermaid.js); not worth adding solely for
  rasterizing an SVG that cairosvg already handles.

## 4. Jinja2 for HTML templating

- Official description: "a fast, expressive, extensible templating engine.
  Special placeholders in the template allow writing code similar to
  Python syntax." Distributed on PyPI, standard `pip install jinja2`, no
  non-Python system dependencies. Provides autoescaping (HTML-injection
  protection) and integrates with the same ecosystem WeasyPrint sits in
  (Flask/Pallets). ([jinja.palletsprojects.com](https://jinja.palletsprojects.com/en/stable/))
- This is an uncontroversial choice: it's the de facto standard Python
  HTML templating engine and has no bearing on the container-size/toolchain
  tradeoffs discussed above — recommended without reservation.

## 5. Recommendation

**Stack: Jinja2 → HTML → WeasyPrint → PDF, with headless-Playwright-as-a-
Mermaid-renderer used only to turn Mermaid text into inline SVG, and
cairosvg for any SVG → PNG rasterization needed for email.**

Concretely:

1. Render report content with **Jinja2** templates to an HTML string.
2. For each Mermaid code block, render the diagram to an SVG string using
   **headless Playwright + a locally bundled mermaid.js** (`page.evaluate`
   calling `mermaid.render()`), and inline the resulting `<svg>` directly
   into the Jinja2 template. This is chosen over mermaid-cli because it
   avoids adding a second runtime (Node/npm) on top of the Python
   container — Playwright's own Chromium already suffices, and it avoids
   depending on network access to mermaid.ink or standing up a separate
   Kroki service.
3. Feed the final HTML (with inline SVGs, `@font-face` web fonts, and
   `@page` CSS for headers/footers) into **WeasyPrint** to produce the PDF.
   WeasyPrint does not need Chromium at all for this step — it is a
   separate, lightweight HTML/CSS-to-PDF layout engine (Pango/Cairo only).
4. If a PNG is needed for email clients that render inline images instead
   of attachments, rasterize the same inline SVGs with **cairosvg**
   (`pip install cairosvg`), reusing the Cairo/Pango system libraries
   already installed for WeasyPrint — no extra system dependency.

### Why this beats the alternatives

- **All-Playwright (Chromium for both Mermaid *and* HTML→PDF)** was
  considered: it would mean a single runtime, but Playwright/Chromium
  headless PDF generation is nonetheless a full browser process per
  render (memory/`--ipc=host` tuning, non-root/seccomp hardening for
  headless Chromium per Playwright's own Docker docs), which is heavier
  and more failure-prone than a pure library call to WeasyPrint for the
  PDF step. Using Chromium *only* for the narrow, unavoidable Mermaid step
  keeps the browser's blast radius small.
- **mermaid-cli** was rejected as the default because it requires a second
  language runtime (Node.js + npm) purely to drive the same
  Puppeteer/Chromium dependency Playwright already provides in Python —
  redundant toolchain surface for no fidelity gain.
- **mermaid.ink** was rejected because it depends on outbound internet
  access and a third-party service holding (potentially confidential)
  meeting-report diagram content — unsuitable for a pipeline that should
  run deterministically and privately on a cluster node.
- **Kroki self-hosted** was rejected as unnecessary: it re-adds a
  Chromium-backed Mermaid companion container behind a second HTTP
  service, adding orchestration complexity without removing the
  underlying Chromium dependency this stack already has via Playwright.
- **resvg-py / svglib** were not chosen over cairosvg for PNG rasterization
  because cairosvg shares WeasyPrint's existing Cairo/Pango system
  libraries (zero new footprint), whereas resvg-py would add a second,
  separate SVG engine and svglib has documented CSS/clip-path/mask gaps
  that are a fidelity risk for anything beyond very simple diagrams.

### Install steps (summary)

```
pip install jinja2 weasyprint cairosvg playwright
playwright install --with-deps chromium   # only the Chromium browser+deps, not all 3 engines
```

System packages (Debian/Ubuntu base image): `libpango-1.0-0`,
`libpangoft2-1.0-0`, `libharfbuzz-subset0` for WeasyPrint/cairosvg, plus
whatever `playwright install --with-deps` pulls in for a headless Chromium
(fonts, graphics libs — no X server/display needed, Chromium runs in
`--headless` mode). A Unicode-complete font family (e.g. `fonts-noto` or
`fonts-dejavu`) should be installed explicitly rather than relying on the
base image's default fonts, to guarantee Norwegian æøå and other diacritics
render correctly in both the Mermaid SVGs and the WeasyPrint PDF.
([doc.courtbouillon.org/weasyprint/stable/first_steps.html](https://doc.courtbouillon.org/weasyprint/stable/first_steps.html),
[playwright.dev/python/docs/docker](https://playwright.dev/python/docs/docker))

### Container size implications

- WeasyPrint + cairosvg + Jinja2: adds only Pango/Cairo/Fontconfig native
  libraries (tens of MB) on top of a Python base image — no browser binary.
- Playwright + Chromium (`--with-deps chromium`, not the full 3-engine
  install): still a full Chromium binary plus its native dependency set,
  historically several hundred MB; this is the dominant size cost of the
  whole stack, but it is a cost already accepted by using Playwright for
  Mermaid, and installing Chromium-only (skip Firefox/WebKit) keeps it to
  one browser instead of three.
  ([playwright.dev/python/docs/docker](https://playwright.dev/python/docs/docker))
- Running on a GPU cluster node is a non-issue either way: neither
  WeasyPrint nor headless Chromium requires a GPU or a display server —
  Chromium's headless mode and WeasyPrint's Pango/Cairo layout are both
  purely CPU/software-rendering paths, so no `Xvfb`, `--gpu` flags, or
  driver wiring is required.

## Sources

- WeasyPrint homepage — https://weasyprint.org/
- WeasyPrint "First steps" (install, system deps, fonts) — https://doc.courtbouillon.org/weasyprint/stable/first_steps.html
- WeasyPrint "Features" (CSS/SVG/fonts/PDF features) — https://doc.courtbouillon.org/weasyprint/stable/features.html
- WeasyPrint GitHub issue #359 (full font embedding) — https://github.com/Kozea/WeasyPrint/issues/359
- WeasyPrint GitHub issue #1237 (WOFF font not embedded) — https://github.com/Kozea/WeasyPrint/issues/1237
- WeasyPrint GitHub issue #2868 (.notdef glyph for unsupported Unicode) — https://github.com/Kozea/WeasyPrint/issues/2868
- Playwright for Python docs (install) — https://playwright.dev/python/docs/intro
- Playwright Docker docs — https://playwright.dev/python/docs/docker
- mermaid-cli GitHub repo — https://github.com/mermaid-js/mermaid-cli
- mermaid.ink — https://mermaid.ink/
- Kroki GitHub repo — https://github.com/yuzutech/kroki
- mermaid-py on PyPI — https://pypi.org/project/mermaid-py/0.2.6
- pymermaid on PyPI — https://pypi.org/project/pymermaid/
- python_mermaid on PyPI — https://pypi.org/project/python_mermaid/
- CairoSVG homepage — https://cairosvg.org/
- CairoSVG README — https://raw.githubusercontent.com/Kozea/CairoSVG/master/README.rst
- resvg_py on PyPI — https://pypi.org/project/resvg_py/
- resvg-py GitHub repo — https://github.com/briceyan/resvg-py
- svglib on PyPI — https://pypi.org/project/svglib/
- Jinja documentation — https://jinja.palletsprojects.com/en/stable/
