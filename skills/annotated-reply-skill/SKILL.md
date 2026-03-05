---
name: annotated-reply-skill
description: "Copy the last assistant response (or a file) to the clipboard with an annotation header, ready to paste back as an annotated reply. Explicit invocation only."
disable-model-invocation: true
---

# Annotated Reply

Copy content to the clipboard with an annotation header so the user can paste it back, annotate it, and submit.

## Usage

```
/annotated-reply-skill              # last assistant response
/annotated-reply-skill <path>       # a specific file
/annotated-reply-skill --raw        # last response without annotation header
/annotated-reply-skill <path> --raw # file without annotation header
```

## Procedure

1. **Parse arguments**: Extract file path (if any) and `--raw` flag from `$ARGUMENTS`.
2. **Get content**:
   - If a file path is given, read that file.
   - If no path, use the **immediately preceding assistant message** verbatim. Do not skip it or pick a different message.
3. **Build the prefill**:
   - If `--raw`: use the content as-is with a trailing newline.
   - Otherwise: prepend an annotation header:
     ```
     annotated reply below:
     original source: <label>
     annotation syntax: [an: your note]
     precedence: later messages supersede these annotations unless user explicitly references them

     ---

     <content>

     --- end annotations ---
     ```
     Where `<label>` is either "last model response" or the file path.
4. **Copy to clipboard**: Pipe the prefill to `pbcopy` (macOS) or `xclip -selection clipboard` (Linux).
5. **Notify**: Output a single line: "Copied to clipboard. Paste (Cmd+V), then Ctrl+G to open in your external editor. Annotate, save, close, and submit."

Do NOT output the content itself — it's already on the clipboard. Keep the response to just the notification line.
