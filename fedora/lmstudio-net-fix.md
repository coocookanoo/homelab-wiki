# LM Studio — Enable Network Access for JS Code Sandbox

> **DEPRECATED** — LMStudio has been replaced by Ollama. See [[fedora/ollama-setup]].

This document is kept for historical reference only.

---

## Problem

The `run_javascript` tool (used by local models to execute code) runs in a Deno sandbox
with `--deny-net` and `--deny-run` hardcoded. Every `fetch()` attempt fails with:

```
Requires net access to "host:443", run again with the --allow-net flag
```

## Fix

Edit **both** files (the compiled bundle is what actually runs):

```
~/.lmstudio/extensions/plugins/lmstudio/js-code-sandbox/.lmstudio/production.js  ← runs this
~/.lmstudio/extensions/plugins/lmstudio/js-code-sandbox/src/toolsProvider.ts     ← source
```

Find the Deno spawn args and change:

```ts
// Before
"--deny-net",
"--deny-run",

// After
"--allow-net",
"--allow-run",
```

**Restart LM Studio completely** after saving (plugin is loaded at startup).

## Warning

LM Studio updates will overwrite this file and re-lock it. If the model loses
web access after an update, re-apply this fix.
