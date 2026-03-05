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
- Before rendering, convert any `[an: ...]` annotation markers to HTML highlights with bordered styling:
  ```bash
  cat > /tmp/preview-annotations.css << 'CSSEOF'
  mark {
    background: #fff3cd;
    border: 1px solid #e0c36a;
    border-radius: 3px;
    padding: 1px 5px;
    font-size: 0.9em;
  }
  CSSEOF
  sed 's/\[an: \([^]]*\)\]/<mark>\1<\/mark>/g' <input> | pandoc -t html -s --metadata title="Preview" --css /tmp/preview-annotations.css -o /tmp/preview-browser-skill.html && open /tmp/preview-browser-skill.html
  ```
