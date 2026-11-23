https://github.com/dawangnailong/chat4code-unrestricted

出于个人原因，请移步至 [Here](https://github.com/dawangnailong/chat4code-unrestricted)

<p align="center">
  <img src="./brand.jpg" alt="chat4code-unrestricted — Yes, Sun Xiaochuan Chat" width="760">
</p>
<h1 align="center"><img src="https://github.com/EasyJailbreak/EasyJailbreak/blob/master/image/README/logo.png" alt="EasyJailbreak Logo" height="60"></h1>

<h1 align="center">chat4code-unrestricted</h1>

<p align="center">
  Yes, Sun Xiaochuan Chat — a static DeepSeek chat front end with an unrestricted mode, editable replies, and direct control over model parameters<br>
  English | <a href="./README.zh-CN.md">简体中文</a>
</p>

> This is an **unofficial fork** of the official demo [deepseek-ai/chat-demo](https://github.com/deepseek-ai/chat-demo), MIT licensed.
> Every change lives in the browser: how the request body is assembled, how replies are stored, and how they are rendered.
> There is no server-side proxy or relay service. Your API key and conversations stay in your own browser; the optional
> OpenCode Bridge mode keeps the OpenCode key in a local browser extension (Chrome or Safari) instead.
>
> 🔗Live demo: <https://chat-unrestricted.fur0.com/>
>
> 📦Repository: <https://github.com/debiandawangnailong7-coder/chat4code-unrestricted>
>
> 🎥Intro video: <https://www.bilibili.com/video/BV1p5sL6LEfd/>
>
> 📚Reference paper: <https://arxiv.org/pdf/2403.12171>

<p align="center">
  <img src="./Demo%20Picture.png" alt="Chat demo — technical Q&A in unrestricted mode" width="860">
</p>

---

## Contents

- [1. What this is](#1-what-this-is)
- [2. Restrictions in the original](#2-restrictions-in-the-original)
- [3. What this fork changes](#3-what-this-fork-changes)
- [4. Behaviour matrix](#4-behaviour-matrix)
- [5. Settings reference](#5-settings-reference)
- [6. What the request actually contains](#6-what-the-request-actually-contains)
- [7. Implementation index](#7-implementation-index)
- [8. Data, privacy, and network](#8-data-privacy-and-network)
- [9. Run and build](#9-run-and-build)
- [10. Known limitations](#10-known-limitations)
- [11. License and credits](#11-license-and-credits)

---

## 1. What this is

A fully static DeepSeek chat front end: multiple conversations, streaming replies, deep thinking, web search, link reading, attachments, speech, Markdown / KaTeX / Mermaid, Chinese and English UI, light and dark themes, and history import/export. Every original feature is preserved.

> Current version: **2.5.2**

On top of that, this fork turns the hard-coded limits of the original into configurable settings and adds a full set of jailbreak, experiment and context-management features:

- **Unrestricted mode and prompts**: unrestricted mode, the default rules document / GODMODE preset, silent per-round system-prompt injection, and the experimental Chinese translation;
- **Advanced engines**: a 0–4 thinking slider, input perturbation (33 techniques, single technique or tiered variants), AutoTune, STM output shaping, and one-click auto jailbreak that retries with another combo on refusal;
- **Jailbreak lab**: local MCP + EasyJailbreak attackers, model races, variant races (budget cap + start confirmation), and result import/export;
- **Chat features**: editable replies, anti-retraction, branch to a new chat (the original is untouched), and long-chat compression (pressure-triggered auto plus manual);
- **Models and cost**: a model card in the input bar (built-in Flash / V4 Pro plus cloud fetch), a context ring, session statistics, and per-model billing;
- **Data and attachments**: configurable size / count / extension limits, history import/export, and one-click restore-default-settings.

- **Unrestricted mode** — swap the system prompt, automatically skip the safety check, and allow continuing refused replies.
- **Direct model parameters** — temperature, Top P, and max output tokens.
- **Anti-retraction** — when the model retracts a reply, keep the text and thinking that were already streamed.
- **Editable replies** — rewrite the body or the chain of thought of any AI reply; every later turn uses the edited version.
- **Custom system prompt** — rewrite the `system` field directly in Settings, overriding the assembled result;
- **Jailbreak lab** — run EasyJailbreak attackers through a local MCP backend (PAIR, TAP, GPTFuzzer, ReNeLLM, ICA, DeepInception, Jailbroken, Cipher, CodeChameleon, Multilingual) and inspect the final prompt, target response and judge result for every instance.

Every switch only affects **requests sent from this browser and conversations stored locally**. Nothing changes the behaviour of the API service itself.

---

## 2. Restrictions in the original

These are the limit-related behaviours in the upstream code. Paths point at the upstream implementation; in this fork those files have been modified (see section 3).

### 2.1 Request side

| #   | Restriction                              | Where in the original                              | Behaviour                                                                                                                             |
| --- | ---------------------------------------- | -------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Hard-coded system prompt                 | `SYSTEM_PROMPT` in `src/services/chat/store.ts`    | Every request's `system` field is one identity sentence: `该助手为DeepSeek，由深度求索公司创造。`. No behavioural instructions at all |
| 2   | Base safety filtering always on          | `SAFETY_CHECK` in `src/services/chat/store.ts`     | Every request carries `safety_check: { level: 'high' }`; the comment states it is "always on"                                         |
| 3   | Model and sampling parameters hard-coded | `src/services/chat/store.ts`                       | Model pinned to `deepseek-flash`; `temperature`, `top_p`, and `max_tokens` are never sent and the server defaults apply               |
| 4   | Fixed tool declarations                  | `serverTools` in `src/services/chat/store.ts`      | `deepseek_read_url` is always declared; `deepseek_web_search` follows the smart-search toggle                                         |
| 5   | Streamed content discarded on refusal    | `settleReply` in `src/services/chat/reply.ts`      | When a stream ends with `stop_reason: 'refusal'`, all received `thinking` / `text` blocks are **deleted**; only tool blocks survive   |
| 6   | Refused turns excluded from context      | `isUsableReply` in `src/services/chat/reply.ts`    | Rounds with `stopReason === 'refusal'` are never sent back to the model                                                               |
| 7   | Refused replies cannot be continued      | `canContinueReply` in `src/services/chat/reply.ts` | Returns `false` for refusal, so the "Continue" button never appears                                                                   |
| 8   | Refused replies show a canned answer     | `src/services/chat/describe.ts`                    | Refusals render a fixed sentence: `你好，这个问题我暂时无法回答，让我们换个话题再聊聊吧。`                                            |
| 9   | Refused replies cannot be edited         | The original has no reply editing                  | Only the user prompt can be edited (which creates a new branch and regenerates)                                                       |

### 2.2 UI and technical boundaries (not safety policy, listed for completeness)

| #   | Item              | Where                                          | Notes                                                                                                              |
| --- | ----------------- | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| 10  | AI content notice | Under the composer                             | A fixed line, "AI-generated, for reference only". Display only, it blocks nothing (kept unchanged in this fork)    |
| 11  | Attachment limits | `src/services/attachments.ts`, `uploadExts.ts` | 64 MB per file, 50 files per conversation, extension allow-list (no heic/heif)                                     |
| 12  | No backend        | By design                                      | The browser talks straight to the configured API origin (default `https://api.deepseek.com`); no server-side relay |

> To be clear: **the hard limits live on the server**. The lowest `safety_check` value is `low` (the docs state that omitting the field equals `low`) and there is no "off"; the model's own alignment is also not something a front end can change. This fork simply opens up everything the **client** can control.

---

## 3. What this fork changes

### 3.1 Settings layout

The settings dialog grows from two groups to eight:

| Group           | Contents                                                                                                                          |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| General         | Theme, language, read-aloud voice, location (original, untouched)                                                                 |
| **Model**       | Current model, temperature, Top P, max output tokens, anti-retraction, auto-compress, skip safety check, continue refused replies |
| **Prompt**      | Unrestricted mode, per-round prompt injection, custom system prompt, pipeline explanation, full preview                           |
| **Advanced**    | System prompt preset, Chinese jailbreak prompts, input perturbation, auto jailbreak, AutoTune, output shaping                     |
| **Attachments** | Limit switch, max size per file, max files per session, extension whitelist                                                       |
| Data            | API key, import, export, restore default settings, delete all                                                                     |
| About           | Repository, demo and upstream links                                                                                               |
| Changelog       | Version history and upstream credits                                                                                              |

No existing style rules were modified. New controls only use newly added classes (the switch, slider, and editor each ship their own CSS Module; a few prompt-preview classes are appended to the settings stylesheet), and colours and radii come from the existing design tokens.

### 3.2 Unrestricted mode

**Where:** Settings → Prompt.

**How it works:** the system prompt becomes an assembly pipeline:

```text
Persona block (official identity sentence)
      │
      ▼  [unrestricted mode]
      ├─ on : append the unrestricted rules block, force-skip the safety check, allow continuing refused replies
      └─ off: send the Persona only, i.e. the official default prompt
      │
      ▼
request body `system` field
```

- The unrestricted rules come from [`deepseekv4.1 and v4.1-1017.md`](<./deepseekv4.1 and v4.1-1017.md>) in the repository root. At build time rspack inlines it verbatim with `asset/source`, so the built app does not depend on the file.
- Two normalisations run during assembly: the leftover sample-answer scaffold at the end of the document is trimmed, and `{{PAYLOAD}}` is replaced with "the request in the latest user message".
- The settings page can expand and preview **the exact system prompt that will be sent**.
- Turning it on forces two things: the `safety_check` field is not sent, and refused replies can be continued (the two switches below show as on and disabled).
- **Custom system prompt** (the input box in the same group): left empty, the pipeline above still applies; once filled, its content is sent verbatim as `system` and the assembled result no longer applies. Unrestricted mode then still skips the safety check and allows continuing refused replies, but no longer swaps the prompt. "Fill with default" seeds the box with the current assembled prompt as a starting point; "Reset to default" clears the override.

### 3.3 Model parameters

**Where:** Settings → Model.

| Parameter         | Control               | Range / step      | Default | Sent as                                  |
| ----------------- | --------------------- | ----------------- | ------- | ---------------------------------------- |
| Temperature       | Slider + number input | 0–2, step 0.05    | 1       | `temperature`                            |
| Top P             | Slider + number input | 0.01–1, step 0.01 | 1       | `top_p`                                  |
| Max output tokens | Number input          | ≥ 1, may be empty | empty   | Empty means `max_tokens` is **not sent** |

- Slider and number input stay in sync; the input commits on blur or Enter, clamps out-of-range values, and rounds to the step.
- Everything is persisted in localStorage and restored on reload.
- All of them are sent with every request except max output tokens, which is omitted while empty.

### 3.4 Anti-retraction

**Where:** Settings → Model.

The original deletes streamed text and thinking when a reply ends with `stop_reason: 'refusal'`. With anti-retraction on, that content is kept instead:

| Behaviour                | Off (original behaviour)              | On                                                                                                     |
| ------------------------ | ------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Streamed body / thinking | Deleted                               | Kept and rendered normally                                                                             |
| UI notice                | Canned refusal answer                 | One notice line: "This reply was flagged by safety review; anti-retraction kept the streamed content." |
| Following context        | Turn excluded                         | **Included** once the body was kept                                                                    |
| Edit button              | Unavailable                           | Available (the reply is treated as a normal one)                                                       |
| History export           | Marked `CONTENT_FILTER` plus template | Exported as a normal finish, so re-importing does not wipe it again                                    |

Anti-retraction is independent of unrestricted mode; turning on unrestricted does not enable it.

### 3.5 Skip safety check

**Where:** Settings → Model.

- Off (default): requests carry `safety_check: { level: 'high' }` as before.
- On: the request body **no longer contains `safety_check`** (`undefined` is dropped by `JSON.stringify`).
- Unrestricted mode forces it on and disables the switch.

Per the official docs, omitting `safety_check` is identical to `low`, which is the lowest level the API offers.

### 3.6 Continue refused replies

**Where:** Settings → Model.

- Off (default): `canContinueReply()` returns `false` for `stopReason === 'refusal'` and the Continue button stays hidden (original behaviour).
- On: a refused reply is treated like a normal finish, so the last refused reply gets a Continue button.
- Unrestricted mode forces it on and disables the switch.

All other continuation rules are unchanged: template notices, server tool blocks, and non-prefix messages still block continuation.

### 3.7 Editable replies

**Where:** the action row under every AI reply — copy / read aloud / **edit (pencil)** / regenerate.

- The pencil expands an inline editor: **chain of thought** on top (only when the reply has one) and **body** below. `Ctrl/Cmd + Enter` saves, `Esc` cancels.
- Saving writes the content **back into the message blocks** — no new branch, no new request. Every later turn that assembles context reads the edited version.
- Write-back rules: thinking goes into the first thinking block (a new one is prepended if none exists) and the others are cleared; the body only touches the trailing answer run's first text block, the remaining text blocks are cleared, and citations are dropped when the body changes. Tool blocks, template notices, and thinking signatures are left untouched.
- The first edit stores a snapshot of the original blocks; **Revert to original** in the editor restores it.
- Continuing a reply changes the block structure, so the snapshot no longer matches and revert disappears after a continuation.
- Replies that were refused _and_ had their content dropped cannot be edited (there is no model text left); with anti-retraction on, the kept content can be edited.

### 3.8 Engineering changes

| Change                                           | Where                                                                          |
| ------------------------------------------------ | ------------------------------------------------------------------------------ |
| Inline `.md` as raw source in the bundle         | `rspack.config.ts` (`{ test: /\.md$/, type: 'asset/source' }`)                 |
| `*.md` module typing                             | `src/env.d.ts`                                                                 |
| Prompt assembly pipeline                         | `src/services/chat/systemPrompt.ts` (new)                                      |
| Switch component                                 | `src/components/base/Switch.tsx` + `Switch.module.css` (new)                   |
| Slider + number control                          | `src/components/settings/ParamControls.tsx` + `ParamControls.module.css` (new) |
| Reply editor                                     | `src/components/messages/ReplyEditor.tsx` + `ReplyEditor.module.css` (new)     |
| Translations                                     | `src/i18n/zh.ts`, `src/i18n/en.ts`                                             |
| Perturbation / AutoTune / shaping / race engines | `src/services/unrestricted/` (ported from G0DM0D3, AGPL-3.0)                   |

> `deepseekv4.1 and v4.1-1017.md` is a build dependency. Commit it together with the code, otherwise a fresh clone will fail to build.

### 3.9 Jailbreak lab (optional, requires the local MCP backend)

The sidebar now has a **Lab** entry: it connects to the local `mcp-server/` over MCP and brings the EasyJailbreak attackers into the UI.

- Backend: `mcp-server/` (Python, MCP Streamable HTTP, default `http://127.0.0.1:8765/mcp`); start it with `./mcp-server/run.sh`;
- Frontend: `src/services/lab/` (MCP client) and `src/components/lab/` (lab page) only handle scheduling and display;
- Custom queries, dataset size, method-parameter JSON, and three model channels (target / attack / eval, reusing the API key from Settings by default); the model dropdown offers **DeepSeek Flash / V4 Pro**, defaults to the chat model, and is shared by all three channels;
- On mobile the parameter panel and results stack into one scrolling column, two-column fields collapse, and result summaries wrap;
- Shows the final prompt, target response, judge result and ASR per instance; supports cancelling and exporting;
- Available: PAIR, TAP, GPTFuzzer, ReNeLLM, ICA, DeepInception, Jailbroken, Cipher, CodeChameleon, Multilingual (all smoke-tested with the local mock model; the real behaviour of TAP and GPTFuzzer depends on the judge model returning parseable scores). MJP is not adapted yet; GCG / AutoDAN need a local white-box model and do not work with an API-only setup. See [`mcp-server/README.md`](mcp-server/README.md);
- Datasets: after `pip install datasets`, AdvBench (520 rows) / HarmBench / StrongREJECT can be loaded online through the `hf-mirror.com` mirror by default (override with `HF_ENDPOINT`); without the package the built-in sample queries are used;
- Results can be exported to JSON and imported back for later review; a successful entry has a "Send to chat" button that starts a new chat draft with the final prompt (any `{query}` placeholder is replaced with a hint you can edit before sending).
- Three steps: `pip install datasets` (optional) → `./mcp-server/run.sh` → `pnpm dev`, then connect from the sidebar Lab entry and run. If connecting returns 403, see the troubleshooting section in [`mcp-server/README.md`](mcp-server/README.md) (browser Origin allow-list: dev server, `null` for `file://`, and the deployed domains; extend it with `MCP_ALLOWED_ORIGINS`).

### 3.10 Advanced features: prompt presets, perturbation, AutoTune, output shaping, model race

The input bar is responsive: on desktop a single compact row shows the labels (model, auto jailbreak, thinking, search, perturbation) with a thin divider between the model and the toggles; below 768px they collapse to circular icons with horizontal scrolling instead of hiding desktop labels based on width measurement.

Settings → **Advanced** (everything is off by default, so the out-of-the-box behaviour is unchanged):

| Setting               | What it does                                           | Caveat                                                                                                                                 |
| --------------------- | ------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------- |
| System prompt preset  | Switches between the default v4.1 document and GODMODE | GODMODE replaces the whole `system` field instead of appending the rules chunk; a custom prompt still overrides it                     |
| Input perturbation    | Rewrites matched trigger words before sending          | The input box shows the rewritten text, i.e. exactly what is sent; resends and continuations reuse it                                  |
| Technique / intensity | 8 techniques, 3 intensities                            | Leetspeak, homoglyphs, mixed case and phonetic only affect Latin letters; full-width, thin spacing and zero-width work for Chinese too |
| AutoTune              | Picks sampling parameters from the prompt context      | DeepSeek's Anthropic-compatible API only accepts `temperature` and `top_p`; `top_k` and friends are never sent                         |
| Output shaping        | Hedge reduction / direct / casual                      | Affects rendered chat and copied race results only; the stored reply is untouched                                                      |

- The trigger table is bilingual (60+ English, 90+ Chinese); Chinese triggers match by substring and English ones by word boundary — see `src/services/unrestricted/parseltongue.ts`;
- The input toolbar gains a **Perturb** button that opens a panel with the on/off switch, technique, matched-trigger count and a live preview;
- The lab's attackers tab gains **Perturb test queries first**, which rewrites custom test queries before they are submitted;
- The lab gains a **Model race** tab: 5 L1B3RT4S prompt combos run in parallel (optionally alternating `deepseek-v4-pro` / `deepseek-flash`), stream into cards, get a 0–100 composite score (length, structure, anti-refusal, directness, relevance) and a winner badge, and each card can **start a chat with this combo** (the combo's system prompt is attached to that chat only (global settings stay untouched) and the question goes into the input box);
- Engine code lives in `src/services/unrestricted/`, ported from [G0DM0D3](https://github.com/elder-plinius/G0DM0D3), with AGPL-3.0 headers and a license copy (`COPYING-AGPL-3.0`).

### 3.11 Thinking effort, context usage, session statistics and cost estimate

- **Thinking effort**: the old on/off switch in the input bar becomes a 0–4 slider (Off / Light / Standard / Deep / Max). The level is sent as `thinking: { type: "enabled", budget_tokens }` plus `reasoning_effort` (low / medium / high / max). If the server answers 400 mentioning reasoning / thinking / budget / effort, the client drops the effort fields, retries once with plain `thinking: { type: "enabled" }` and tells you it fell back.
- **Context usage**: the ring in the input corner converts the last request's context (`input + cache_read + cache_creation`) into a share of the model window (1M); amber past 60%, red past 85%. The tooltip shows the exact numbers, the compaction threshold (80% of the window after a 64k headroom and the output reserve) and, after compression, the effective usage and tokens saved.
- **Session statistics** (the stats button next to the ring): rounds, steps (replies + tool calls), output speed, average first token, model time, cache hit rate, plus uncached input / cache read / cache write / output tokens; a live section shows the current run while it streams.
- **Auto jailbreak** (toggle in the input bar): swaps `system` to GODMODE plus the depth directive, forces perturbation (full-width when no technique is chosen), forces AutoTune and skips the safety check. When a reply is refused (`stop_reason: refusal` or refusal text) with no retained text, it retries with L1B3RT4S combos twice, then stops and tells you (each retry is a real request and costs tokens).
- **Reply ratings**: thumbs up / down on any reply feed the AutoTune EMA profile together with that run's context type and sampling parameters, so later AutoTune runs learn from your feedback (stored locally only).
- **Per-chat system prompt**: `Start a chat with this combo` now affects only the new chat, with a notice bar to restore the default; global prompt settings are untouched. It is stored with the chat in IndexedDB; history exports do not include it.
- **Cost estimate**: priced from the official price page (yuan per million tokens) with separate rates for `deepseek-flash` and `deepseek-v4-pro` (each reply is priced by its own model; unknown models fall back to Flash), split into peak and off-peak (Beijing time weekdays 09:00–12:00 and 14:00–18:00 are peak; off-peak costs half), with a "treat today as a holiday" override. Search and fetch are billed at 5 yuan / 1000 calls, document parsing at 1.5 yuan / 1000 pages (pages estimated from file size), TTS at 3 yuan / 10k characters (optional limited-time 50% off) and ASR at 0.78 yuan / hour.

### 3.12 Attachment limits

Settings → **Attachments**:

| Item                      | Default                        | Notes                                                                           |
| ------------------------- | ------------------------------ | ------------------------------------------------------------------------------- |
| Enforce attachment limits | on                             | Turn off to skip size, count and extension checks; the server still decides     |
| Max size per file         | 64 MB                          | Plain text, code and spreadsheet files still use the smaller 32 MB server limit |
| Max files per session     | 50                             | Exceeding it rejects the whole batch                                            |
| Extension whitelist       | built-in list (no heic / heif) | Comma or newline separated; empty restores the default                          |

---

### 3.13 Variant race (third lab tab)

The standalone app's Parseltongue v4 now lives in the lab: **33 perturbation techniques**, selected by tier (light 11 / standard 22 / heavy 33), rewrite each trigger word in the question, fire requests in parallel batches and keep the best composite score.

| Item              | Notes                                                                                                                                                                                |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Tiers             | 11 / 22 / 33 techniques, in the same order as upstream                                                                                                                               |
| Sampling          | Each variant cycles through one of 11 temperature / top_p pairs, matching `getParseltongueSamplingParams`                                                                            |
| Concurrency       | Batches of 6 with 100 ms stagger; stops early once the leader scores ≥ 65                                                                                                            |
| Budget cap        | The per-request worst case estimates how many requests fit; no new request is fired after reaching the budget, and billing uses server-reported usage                                |
| Confirmation      | The first start asks for confirmation with request count and worst-case cost; "Do not ask again" hides it permanently                                                                |
| Refusal detection | Either `stop_reason: refusal` or refusal text marks a variant as refused; refused variants never win                                                                                 |
| Import / export   | "Export results" saves JSON including the system prompt actually used; "Import results" replays a previous race; the `type` field keeps files from other pages out                   |
| Continue in chat  | Any finished result can be carried into a new chat: the obfuscated text becomes the question and the reply the answer; the system prompt is attached to that chat, never to Settings |

Chinese support: `spaced / zwj / dotted / underscored / reversed / brackets / strikethrough / hyphenated` work on Chinese triggers; `bubble / unicode / morse / pigLatin / mathBold` only touch Latin letters, and `fullwidth` only shifts ASCII. Chinese triggers match by substring rather than `\b`.

---

### 3.14 Prompt injection, Chinese translation and long-chat compression

| Feature                                  | Where                                           | Notes                                                                                                                                                                                                                                                                                                                                                                                                                        |
| ---------------------------------------- | ----------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Inject the system prompt every round     | Settings → Prompt                               | Silently appends the current system prompt to `latest_reminder` on every request so long chats keep the setup; it is not stored in history snapshots, and input tokens clearly increase.                                                                                                                                                                                                                                     |
| Chinese jailbreak prompts (experimental) | Settings → Advanced                             | Replaces the built-in GODMODE and depth directive with Chinese translations. Phrasing and training-data differences may weaken the jailbreak, so it is off by default. L1B3RT4S combos stay in English.                                                                                                                                                                                                                      |
| Long-chat compression                    | Settings → Model (auto) + stats dialog (manual) | Auto: once context pressure reaches 80% of the window (after the 64k headroom and output reserve) with at least 2 new rounds since the last compression, older content is summarised in the background. Manual: "Compress older messages" in the stats dialog, always keeping the last 2 rounds verbatim. The summary is injected as a `latest_reminder`; local history is untouched and the summary can be cleared anytime. |

---

### 3.15 Branch to a new chat

Use the session menu (⋯ → **Branch to new chat**) or the button in the desktop session header to copy the current visible branch into a **brand-new chat**:

- The new session id and every message id are regenerated; the original chat is untouched;
- The current system prompt (lab combo) and compression summary are inherited;
- The new title carries a " (branch)" suffix, and both lines evolve independently;
- Branching is disabled while generating, importing, or before the session loads.

This is different from the in-session branch switcher (which keeps alternative answer versions inside one chat): here you get a separate line in the session list.

---

### 3.16 Model switching and per-model billing

The leftmost control in the input toolbar is the model card, with **DeepSeek Flash** and **DeepSeek V4 Pro** built in:

- Open it to see each model's cache-hit / input / output price per 1M tokens (off-peak; peak ×2);
- "Fetch from cloud" calls `GET /models`, then the Anthropic prefix `/anthropic/v1/models`; fetched models are cached for an hour and their ids are passed through to requests;
- A switch affects only new replies: old replies keep their own model, continuations keep the original model, and statistics/cost are computed per reply;
- The context ring uses the current model's window; models without a price entry fall back to Flash rates and a 128k window;
- Settings → Model shows the current model; restoring defaults resets it to DeepSeek Flash.

Prices (yuan per million tokens, off-peak; peak ×2):

| Model             | Cache hit | Input (miss) | Output |
| ----------------- | --------- | ------------ | ------ |
| `deepseek-flash`  | 0.02      | 1            | 4      |
| `deepseek-v4-pro` | 0.15      | 4.5          | 13.5   |

The billing section in the stats dialog gains a "By model" breakdown with reply counts and cost per model.

### 3.17 OpenCode Bridge provider (optional)

Settings → **Data** → **Inference provider** adds a second, opt-in transport:

- `DeepSeek API` (default) keeps the existing browser-direct behaviour described above.
- `OpenCode Bridge` forwards Anthropic Messages through the local [OpenCode Bridge](https://github.com/debiandawangnailong7-coder/opencode-bridge) browser extension (Chrome or Safari) to `https://opencode.ai/inference` (Console pay-as-you-go) or `https://opencode.ai/inference/go` (Go subscription), as selected inside the extension. This is needed because the OpenCode inference endpoints send no CORS headers and answer the OPTIONS preflight with 404, so a page cannot call them directly.

Setup:

1. Install the extension and open its options page to save your `oc_sk_...` key there. The key never reaches this page.
   - Chrome: load it unpacked (`chrome://extensions` → Developer mode → Load unpacked).
   - Safari 16.4+: convert it with `xcrun safari-web-extension-converter`, run the generated app once with a signing team, then enable the extension under Safari → Settings → Extensions.
2. Add this page's exact origin (for example `http://localhost:8090`) to the extension's **Allowed origins**, choose Console or Go, and set the User-Agent override to `chat4code-unrestricted/1.0`.
3. Paste the extension ID into Settings → Data → OpenCode Bridge and press **Test connection**.
   - Chrome uses the 32-character ID from `chrome://extensions`.
   - Safari uses the extension's **bundle identifier** (for example `com.example.OpenCodeBridge.Extension`) shown on its options page.
4. Serve this app over `http://localhost:...` or another `http(s)` origin; `file://` is rejected by the extension.
5. Safari also requires per-site access: Safari → Settings → Extensions → OpenCode Bridge → **Website Access** → allow this site (or all sites), then reload the page. Chrome connects directly; Safari pages automatically fall back to the extension's content-script relay over `window.postMessage`. The connection test reports which path was negotiated.

Safari has no offscreen document, so requests run inside the MV3 service worker and are subject to its fetch limits (roughly 25 seconds to first byte and 5 minutes per request). Chrome is not affected; very long Safari replies may be interrupted.

Bridge mode sends standard Anthropic Messages only: DeepSeek-specific fields (`safety_check`, `latest_reminder`, `deepseek_*` server tools, `title`, `reasoning_effort`) are stripped, the model id `deepseek-flash` is mapped to `deepseek-v4.1-flash`, and a stable `x-opencode-session` is derived from each chat. Attachments, speech, server-side web search and the lab are disabled in this mode.

---

## 4. Behaviour matrix

| Scenario                    | Original                    | This fork, defaults (all switches off)                                               | Unrestricted mode on                                      |
| --------------------------- | --------------------------- | ------------------------------------------------------------------------------------ | --------------------------------------------------------- |
| `system` field              | Official identity sentence  | Same                                                                                 | Persona + unrestricted rules block                        |
| `safety_check`              | Always `high`               | `high`                                                                               | Not sent                                                  |
| Custom system prompt        | None                        | Empty = assembled; filled = sent verbatim                                            | Filled text wins; unrestricted no longer swaps the prompt |
| Streamed content on refusal | Discarded                   | Discarded (kept when anti-retraction is on)                                          | Same as default                                           |
| Refused turn in context     | No                          | No (yes when anti-retraction kept the body)                                          | Same as default                                           |
| Continue on refused reply   | Blocked                     | Blocked (switch can enable)                                                          | Allowed                                                   |
| Edit refused reply          | No editing at all           | Editable when content was kept                                                       | Same                                                      |
| Sampling parameters         | Not sent                    | Adjustable                                                                           | Adjustable                                                |
| Export of refused reply     | `CONTENT_FILTER` + template | Same (normal finish when content was kept)                                           | Same as default                                           |
| Other original features     | —                           | Search, attachments, branches, import/export, read-aloud, themes, i18n all preserved | Same                                                      |

---

## 5. Settings reference

| Group       | Setting                              | Default           | localStorage key                           | Unrestricted mode               |
| ----------- | ------------------------------------ | ----------------- | ------------------------------------------ | ------------------------------- |
| Model       | Temperature                          | `1`               | `chat-demo:temperature`                    | Unaffected                      |
| Model       | Top P                                | `1`               | `chat-demo:top-p`                          | Unaffected                      |
| Model       | Max output tokens                    | empty (not sent)  | `chat-demo:max-tokens`                     | Unaffected                      |
| Model       | Anti-retraction                      | off               | `chat-demo:keep-reply-content`             | Unaffected (independent switch) |
| Model       | Skip safety check                    | off               | `chat-demo:skip-safety-check`              | Forced on, disabled             |
| Model       | Continue refused replies             | off               | `chat-demo:continue-refused`               | Forced on, disabled             |
| Prompt      | Unrestricted mode                    | off               | `chat-demo:unrestricted`                   | —                               |
| Prompt      | Custom system prompt                 | empty (assembled) | `chat-demo:system-prompt`                  | Overrides the assembled result  |
| Advanced    | System prompt preset                 | `deepseek`        | `chat-demo:prompt-preset`                  | Unaffected                      |
| Advanced    | Input perturbation                   | off               | `chat-demo:parseltongue`                   | Unaffected                      |
| Advanced    | Perturbation technique               | `leetspeak`       | `chat-demo:parseltongue-technique`         | Unaffected                      |
| Advanced    | Perturbation intensity               | `medium`          | `chat-demo:parseltongue-intensity`         | Unaffected                      |
| Advanced    | AutoTune                             | off               | `chat-demo:autotune`                       | Unaffected                      |
| Advanced    | AutoTune strategy                    | `adaptive`        | `chat-demo:autotune-strategy`              | Unaffected                      |
| Advanced    | Output shaping                       | empty (off)       | `chat-demo:stm-modules`                    | Unaffected                      |
| Attachments | Enforce attachment limits            | on                | `chat-demo:attachment-limits`              | Unaffected                      |
| Attachments | Max size per file                    | 64 MB             | `chat-demo:attachment-max-mb`              | Unaffected                      |
| Attachments | Max files per session                | 50                | `chat-demo:attachment-max-count`           | Unaffected                      |
| Attachments | Extension whitelist                  | built-in          | `chat-demo:attachment-exts`                | Unaffected                      |
| Input bar   | Model                                | `deepseek-flash`  | `chat-demo:model`                          | Next reply                      |
| Input bar   | Thinking effort                      | 0 (off)           | `chat-demo:reasoning-effort`               | Unaffected                      |
| Input bar   | Auto jailbreak                       | off               | `chat-demo:auto-jailbreak`                 | Unaffected                      |
| Lab         | Variant race budget                  | 1 yuan            | `chat-demo:variant-race-budget`            | Unaffected                      |
| Lab         | Variant race confirm dismissed       | off               | `chat-demo:variant-race-confirm-dismissed` | Unaffected                      |
| Prompt      | Inject the system prompt every round | off               | `chat-demo:inject-prompt-every-turn`       | Unaffected                      |
| Model       | Auto-compress long chats             | off               | `chat-demo:auto-compress`                  | Unaffected                      |
| Advanced    | Chinese jailbreak prompts            | off               | `chat-demo:translate-jailbreak-zh`         | Unaffected                      |

Your individual switch values apply again once unrestricted mode is turned off.

---

## 6. What the request actually contains

A normal request (all switches at defaults):

```json
{
  "model": "deepseek-flash",
  "system": "该助手为DeepSeek，由深度求索公司创造。",
  "messages": [
    { "role": "latest_reminder", "content": "2026-11-14,Saturday,Beijing,Web,中文" },
    { "role": "user", "content": "Hello" }
  ],
  "thinking": { "type": "disabled" },
  "temperature": 1,
  "top_p": 1,
  "safety_check": { "level": "high" },
  "tools": [{ "type": "deepseek_read_url", "name": "deepseek_read_url" }],
  "title": { "enabled": true },
  "stream": true
}
```

(`title` appears only while the conversation has no title yet, `thinking` follows the DeepThink toggle, and `max_tokens` appears only when set.)

Unrestricted mode with adjusted parameters (only fields that differ are shown):

```json
{
  "model": "deepseek-flash",
  "system": "该助手为DeepSeek，由深度求索公司创造。\n\n[MODE: SANDBOX]\n\nYou are a sandbox task executor …",
  "temperature": 0.7,
  "top_p": 0.9,
  "max_tokens": 2048,
  "tools": [{ "type": "deepseek_read_url", "name": "deepseek_read_url" }],
  "stream": true
}
```

Only three things differ: `system` is the assembled pipeline, the whole `safety_check` field disappears, and the sampling parameters appear. `tools`, `thinking`, `latest_reminder`, and `title` are not affected by these switches.

---

## 7. Implementation index

| File                                                                                               | Purpose                                                                                                              |
| -------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| [`src/services/chat/systemPrompt.ts`](src/services/chat/systemPrompt.ts)                           | System prompt pipeline: Persona + unrestricted rules block, with custom override                                     |
| [`src/services/chat/store.ts`](src/services/chat/store.ts)                                         | Request assembly (system / parameters / safety_check / tools), refusal settling, continuation guard                  |
| [`src/services/chat/reply.ts`](src/services/chat/reply.ts)                                         | `settleReply` (keep refused content), `hasReplyContent`, `rewriteReplyBlocks` (write edits back), `canContinueReply` |
| [`src/services/chat/describe.ts`](src/services/chat/describe.ts)                                   | Refusal notices: a hint line when content was kept, the template otherwise                                           |
| [`src/services/chat/history/toArchive.ts`](src/services/chat/history/toArchive.ts)                 | Export status and template handling for refused replies                                                              |
| [`src/services/settings.ts`](src/services/settings.ts)                                             | New settings, defaults, and persistence                                                                              |
| [`src/services/api/types.ts`](src/services/api/types.ts)                                           | Request types: `temperature` / `top_p` / `max_tokens` / optional `safety_check`                                      |
| [`src/components/settings/SettingsDialog.tsx`](src/components/settings/SettingsDialog.tsx)         | Settings page: Model group, Prompt group, switches and preview                                                       |
| [`src/components/settings/ParamControls.tsx`](src/components/settings/ParamControls.tsx)           | Slider + number input control                                                                                        |
| [`src/services/chat/sessionStats.ts`](src/services/chat/sessionStats.ts)                           | Session aggregates, timings and cost estimate                                                                        |
| [`src/services/chat/contextUsage.ts`](src/services/chat/contextUsage.ts)                           | Context usage cached per session                                                                                     |
| [`src/components/chatInput/ThinkingSlider.tsx`](src/components/chatInput/ThinkingSlider.tsx)       | Thinking effort 0–4 track                                                                                            |
| [`src/components/session/StatsDialog.tsx`](src/components/session/StatsDialog.tsx)                 | Statistics and billing dialog                                                                                        |
| [`src/services/chat/compression.ts`](src/services/chat/compression.ts)                             | Long-chat compression: structured summary, auto trigger and savings                                                  |
| [`src/services/chat/contextPressure.ts`](src/services/chat/contextPressure.ts)                     | Context pressure and compaction reserve (matching DSH)                                                               |
| [`src/services/chat/feedbackStore.ts`](src/services/chat/feedbackStore.ts)                         | Reply ratings and the AutoTune EMA profile                                                                           |
| [`src/services/api/models.ts`](src/services/api/models.ts)                                         | Built-in models, cloud `/models` fetch and cache                                                                     |
| [`src/components/chatInput/ModelPicker.tsx`](src/components/chatInput/ModelPicker.tsx)             | Model picker card                                                                                                    |
| [`src/components/chatInput/ContextRing.tsx`](src/components/chatInput/ContextRing.tsx)             | Context ring linked with compression                                                                                 |
| [`src/components/chatInput/ParseltonguePanel.tsx`](src/components/chatInput/ParseltonguePanel.tsx) | Input perturbation panel                                                                                             |
| [`src/services/unrestricted/promptsZh.ts`](src/services/unrestricted/promptsZh.ts)                 | Chinese translation of GODMODE and the depth directive                                                               |
| [`src/services/unrestricted/techniques.ts`](src/services/unrestricted/techniques.ts)               | 33 perturbation techniques (ported from the standalone build)                                                        |
| [`src/services/unrestricted/refusal.ts`](src/services/unrestricted/refusal.ts)                     | 44 refusal patterns                                                                                                  |
| [`src/services/lab/variantRace.ts`](src/services/lab/variantRace.ts)                               | Variant race: budget, batches, scoring                                                                               |
| [`src/components/lab/VariantView.tsx`](src/components/lab/VariantView.tsx)                         | Variant race UI and start confirmation                                                                               |
| [`src/components/base/Switch.tsx`](src/components/base/Switch.tsx)                                 | Switch component                                                                                                     |
| [`src/components/messages/ReplyEditor.tsx`](src/components/messages/ReplyEditor.tsx)               | Reply editor (thinking / body)                                                                                       |
| [`src/components/messages/AssistantMessage.tsx`](src/components/messages/AssistantMessage.tsx)     | Edit entry point, Continue button, refusal rendering branch                                                          |
| [`deepseekv4.1 and v4.1-1017.md`](<./deepseekv4.1 and v4.1-1017.md>)                               | Unrestricted rules text, inlined at build time                                                                       |
| [`mcp-server/`](mcp-server/README.md)                                                              | Local EasyJailbreak MCP backend                                                                                      |
| [`src/components/lab/`](src/components/lab/LabView.tsx)                                            | Jailbreak lab page                                                                                                   |
| [`src/services/lab/`](src/services/lab/mcpClient.ts)                                               | MCP client and lab state                                                                                             |

---

## 8. Data, privacy, and network

- API key and settings: browser localStorage (keys listed in section 5).
- Conversations and original attachments: IndexedDB, isolated per browser profile and site origin.
- The only outbound traffic goes to the API origin you configured (default `https://api.deepseek.com`): `/models` for key validation, `/anthropic/v1/messages` for chat, `/anthropic/v1/files` for attachments, and the ASR / TTS endpoints for speech. In OpenCode Bridge mode the page instead sends messages to the local extension over `chrome.runtime` (Chrome) or a content-script relay over `window.postMessage` (Safari); the extension talks to `https://opencode.ai/inference[/go]` and the page never sees the OpenCode key.
- No analytics, no telemetry, no third-party scripts, no relay server.
- History import/export uses ZIP archives; exports contain conversations and locally available image originals, but no API keys or settings. See the [history archive documentation](src/services/chat/history/README.md) for the format and retention rules.

---

## 9. Run and build

**On Windows**, double-click the two scripts in the repository root:

- `启动chat-web前端.bat`: detects Node.js / pnpm / `node_modules` and installs whatever is missing (winget / corepack / npm), then starts the dev server on http://localhost:8090;
- `启动mcp后端越狱实验台.bat`: detects Python / `.venv` / MCP deps / fastchat / torch / datasets / EasyJailbreak; EasyJailbreak is installed via pip first and falls back to GitHub or mirror clones, then starts the MCP backend on http://127.0.0.1:8765/mcp.

Manual setup (any OS), requires Node.js 22.12+ and pnpm:

```sh
git clone https://github.com/debiandawangnailong7-coder/chat4code-unrestricted.git
cd chat4code-unrestricted
pnpm install
pnpm dev        # http://localhost:8090
```

Build the static site:

```sh
pnpm build      # output in dist/
```

`dist/` can be deployed to any static host, or opened locally by double-clicking `dist/index.html` (the browser must allow storage on `file://` pages and cross-origin requests to your API origin). Enter your API key under Settings → Data before the first chat.

| Command       | Purpose                      |
| ------------- | ---------------------------- |
| `pnpm dev`    | Start the dev server on 8090 |
| `pnpm build`  | Build into `dist/`           |
| `pnpm type`   | TypeScript type check        |
| `pnpm lint`   | Lint with oxlint             |
| `pnpm format` | Format with oxfmt            |

The API origin is set in [`rspack.config.ts`](rspack.config.ts) via `createConfig({ apiOrigin })`; it defaults to `https://api.deepseek.com`. OpenCode Bridge mode does not use this origin: the extension chooses the OpenCode base URL (Console or Go) in its own options page.

---

## 10. Known limitations

1. **There is no "off" for `safety_check`**: omitting it equals `low`, which is the lowest level the API offers; the server still runs its default check.
2. **Server-side alignment remains**: a model-initiated refusal (`stop_reason: 'refusal'`) cannot be cancelled by the client. Anti-retraction only keeps what was already streamed; it cannot make the model continue refused content.
3. **Historical chain-of-thought is never sent back**: by protocol, the server drops history thinking and the client does not resend it. Editing the thinking therefore affects the UI and the continuation prefix, while only the body applies to ordinary later turns.
4. **Sampling parameters are constrained in thinking mode**: the server fixes temperature at 1, and Top P below 0.95 is treated as 0.95.
5. **`max_tokens` includes thinking tokens**: setting it too low can truncate the thinking as well.
6. **Edits only affect the local context**: they do not re-request the model and do not change what the server already produced. Exports store the edited content; the original snapshot is not exported.
7. **Attachment limits, key validation, and speech endpoints** still follow the original logic and are independent of the switches described here.
8. **The lab is not a purely static capability**: it needs the local Python + MCP backend (`mcp-server/`), and a single job can issue a large number of model calls. Start with `limit=1` and small iteration counts.
9. **A custom prompt overrides the assembly**: once filled, unrestricted mode no longer swaps `system` (it still skips the safety check and allows continuing refused replies). Clear the box to go back to the default pipeline.
10. **Long-chat compression is lossy**: summaries drop detail, and auto-compression triggers at 80% of the window (about 800k tokens on a 1M window), so manual compression is the practical option. Per-chat system prompts and compression summaries live only in local sessions and are not included in history exports.
11. **Branch to new chat copies the current visible branch**: other answer versions and drafts are not copied; attachments reuse the original file ids; the new chat starts its statistics at zero and inherited usage is not counted.
12. **Restore default settings keeps the API key**: it only resets preferences and switches, never chats, attachments or statistics.
13. **Auto jailbreak and variant races cost extra**: every retry and every variant is a real request; variant races are bounded by the budget cap.
14. **Cloud model lists depend on the API and network**: a failed fetch falls back to the two built-ins; models without a price entry are estimated at Flash rates with a 128k window, and actual capabilities come from the server.
15. **Switching models does not rewrite old replies**: old replies keep their model and cost; continuations keep that reply's model, and only new replies use the current choice.

---

## Star History

<!-- star-history:start -->
<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/star-history/star-history-dark.svg" />
  <source media="(prefers-color-scheme: light)" srcset="assets/star-history/star-history-light.svg" />
  <img alt="Star History Chart" src="assets/star-history/star-history-light.svg" />
</picture>
<!-- star-history:end -->

---

## 11. License and credits

- Code based on [deepseek-ai/chat-demo](https://github.com/deepseek-ai/chat-demo), under the [MIT](LICENSE) license. The build copies the license to `dist/LICENSE`; keep it when redistributing.
- `brand.jpg` is artwork provided by the repository author and remains theirs; "Yes, Sun Xiaochuan Chat" is an inside joke and is not affiliated with DeepSeek.
- This is a personal fork, not an official DeepSeek project. Check the terms and costs of whichever API service you point it at.
- Thanks to the [EasyJailbreak project](https://github.com/EasyJailbreak/EasyJailbreak) for the theoretical foundation.
- Thanks to [DeepSeek-Agent](https://github.com/Aoan2011/DeepSeek-Agent) for the reference implementation of the thinking slider, context ring and session statistics.
- Thanks to [dsh-unrestricted](https://github.com/yexi-by/dsh-unrestricted) for the reference jailbreak prompts and experiment settings.
- Thanks to [L1B3RT4S](https://github.com/elder-plinius/L1B3RT4S), the upstream source for GODMODE, the Hall of Fame combos and jailbreak prompts.
- The engines under `src/services/unrestricted/` (input perturbation, AutoTune, STM, race scoring, GODMODE prompt) are ported from [G0DM0D3](https://github.com/elder-plinius/G0DM0D3) and licensed [AGPL-3.0](src/services/unrestricted/COPYING-AGPL-3.0); that directory keeps its own license, separate from the MIT-licensed rest of the repo.
