# Which Python rendering stack produces the HTML, PDF, SVG, and Mermaid output without a fragile toolchain?

Type: research
Status: resolved
Blocked by: 

## Question

Research from primary sources: WeasyPrint versus Playwright/Chromium for HTML to PDF (Norwegian characters, embedded SVG, fonts); options for rendering Mermaid to SVG in Python without a browser (mermaid-cli needs Node; check pure-Python or headless alternatives); SVG to PNG rasterization for email (cairosvg, resvg); what runs cleanly in a container on a GPU cluster node. Recommend one stack. Write findings to research/render-stack.md with sources.

## Answer

Recommended stack: **Jinja2 → HTML → WeasyPrint → PDF**, with **headless
Playwright + bundled mermaid.js** used only to turn Mermaid text into inline
SVG (no Node/npm, no mermaid-cli), and **cairosvg** for any SVG → PNG needed
for email. WeasyPrint handles Norwegian æøå, inline SVG, web fonts, and
CSS Paged Media headers/footers natively via Pango/Cairo — no browser
process needed for the PDF step itself. There is no maintained pure-Python
Mermaid renderer (mermaid-py/python_mermaid/pymermaid are Jupyter-display
helpers, not headless SVG generators); every real option (mermaid-cli,
mermaid.ink, Kroki) ultimately runs mermaid.js in a Chromium/Node
toolchain, so using Playwright's own Chromium for that one step avoids
adding a second runtime or an external network dependency. cairosvg reuses
WeasyPrint's Cairo/Pango libs for rasterization, adding no new system
dependencies. Neither WeasyPrint nor headless Chromium needs a GPU or
display server, so this runs cleanly on a headless Linux cluster node; the
Chromium binary (Mermaid step only) is the dominant container-size cost.

Full findings and sources: [research/render-stack.md](../research/render-stack.md)
