---
name: preview-browser-skill
description: "Render the last response (or a specified markdown file) as HTML and open in the default browser. Explicit invocation only — do not auto-trigger."
disable-model-invocation: true
---

# Preview in Browser

Render markdown to HTML and open in the default browser.

## Behavior

- If `$ARGUMENTS` is a file path, render that file
- If `$ARGUMENTS` is empty, write your **immediately preceding assistant message** (the very last thing you said before this skill was invoked) verbatim to `/tmp/preview-browser-skill.md` and render that. Do not skip it, summarise it, or pick a different message.

## Rendering

1. Write the stylesheet below to `/tmp/preview-style.css` (overwrite each time).
2. Choose a short, descriptive `<title>` from the content (first heading, topic, or "Preview" as fallback).
3. Convert `[an: ...]` annotation markers to `<mark>` tags, then render with pandoc:

```bash
sed 's/\[an: \([^]]*\)\]/<mark>\1<\/mark>/g' <input> \
  | pandoc -t html -s --mathjax --metadata title="<TITLE>" \
    --css /tmp/preview-style.css \
    -o /tmp/preview-browser-skill.html \
  && open /tmp/preview-browser-skill.html
```

## Stylesheet

Write this to `/tmp/preview-style.css`. You may use judgement to make minor additions (e.g. content-specific accent colours, extra utility classes) but keep the core design intact.

```css
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');

:root {
  --bg: #f8f9fa;
  --card-bg: #ffffff;
  --text: #1a1a2e;
  --text-muted: #6c757d;
  --accent: #4361ee;
  --accent-light: #eef0ff;
  --urgent: #e63946;
  --urgent-light: #fef2f2;
  --success: #2d6a4f;
  --success-light: #ecfdf5;
  --warning: #f59e0b;
  --warning-light: #fffbeb;
  --border: #e9ecef;
  --shadow: 0 1px 3px rgba(0,0,0,0.08), 0 1px 2px rgba(0,0,0,0.06);
  --shadow-md: 0 4px 6px rgba(0,0,0,0.07), 0 2px 4px rgba(0,0,0,0.06);
  --radius: 10px;
}

* { box-sizing: border-box; }

body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
  background: var(--bg);
  color: var(--text);
  line-height: 1.65;
  max-width: 800px;
  margin: 0 auto;
  padding: 2rem 1.5rem 4rem;
  font-size: 15px;
}

h2 {
  font-size: 1.75rem;
  font-weight: 700;
  margin-top: 0;
  padding-bottom: 0.5rem;
  border-bottom: 3px solid var(--accent);
  display: inline-block;
}

h3 {
  font-size: 1.15rem;
  font-weight: 600;
  color: var(--accent);
  margin-top: 2rem;
  margin-bottom: 0.75rem;
  letter-spacing: -0.01em;
}

hr {
  border: none;
  border-top: 1px solid var(--border);
  margin: 1.5rem 0;
}

table {
  width: 100%;
  border-collapse: separate;
  border-spacing: 0;
  background: var(--card-bg);
  border-radius: var(--radius);
  box-shadow: var(--shadow);
  overflow: hidden;
  margin: 1rem 0;
  font-size: 0.9rem;
}

thead {
  background: var(--accent);
  color: white;
}

th {
  padding: 0.7rem 1rem;
  text-align: left;
  font-weight: 600;
  font-size: 0.8rem;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

td {
  padding: 0.6rem 1rem;
  border-bottom: 1px solid var(--border);
}

tr:last-child td { border-bottom: none; }
tbody tr:hover { background: var(--accent-light); }

ol {
  padding-left: 0;
  list-style: none;
  counter-reset: item;
}

ol > li {
  counter-increment: item;
  background: var(--card-bg);
  border-left: 4px solid var(--urgent);
  border-radius: var(--radius);
  box-shadow: var(--shadow);
  padding: 1rem 1.25rem;
  margin-bottom: 0.75rem;
  position: relative;
}

ol > li::before {
  content: counter(item);
  position: absolute;
  top: 0.85rem;
  right: 1rem;
  background: var(--urgent);
  color: white;
  width: 1.5rem;
  height: 1.5rem;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 700;
  font-size: 0.75rem;
}

ul {
  padding-left: 0;
  list-style: none;
}

ul > li {
  background: var(--card-bg);
  border-radius: var(--radius);
  box-shadow: var(--shadow);
  padding: 0.75rem 1.25rem;
  margin-bottom: 0.5rem;
  border-left: 4px solid var(--accent);
}

ul > li::before {
  content: "\2192";
  color: var(--accent);
  font-weight: 700;
  margin-right: 0.5rem;
}

pre {
  background: linear-gradient(135deg, #1a1a2e 0%, #16213e 100%);
  color: #e0e0e0;
  border-radius: var(--radius);
  padding: 1.25rem 1.5rem;
  font-family: 'SF Mono', 'Fira Code', monospace;
  font-size: 0.85rem;
  line-height: 1.8;
  box-shadow: var(--shadow-md);
  overflow-x: auto;
}

code {
  background: var(--accent-light);
  color: var(--accent);
  padding: 0.15em 0.4em;
  border-radius: 4px;
  font-size: 0.88em;
}

pre code {
  background: none;
  color: inherit;
  padding: 0;
}

strong { font-weight: 600; }

p { margin: 0.5rem 0; }

mark {
  background: var(--warning-light);
  border: 1px solid var(--warning);
  border-radius: 3px;
  padding: 1px 5px;
  font-size: 0.9em;
}
```
