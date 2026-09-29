# Prompt suggestions

This file describes the **steps followed** to add Concept 38 (**prompt suggestions**) to the Claude Agent SDK Lab:
what was read, what was probed, what was decided, how it was tested, and what the tests changed. To learn the concept
itself, read [Tab38-Prompt-suggestions.md](Tab38-Prompt-suggestions.md) (the same text is in `readme.md`).

| Concept | Topic | Routes | Explanation |
|---|---|---|---|
| 38 | Prompt suggestions: `promptSuggestions`, the `prompt_suggestion` message, the suggestion API call, when it is skipped, the switches, the filters, the cost in the next `total_cost_usd` | `/api/c38/dry`, `/chat`, `/send`, `/end`, `/when`, `/switches`, `/cache`, `/code` | [Tab38-Prompt-suggestions.md](Tab38-Prompt-suggestions.md) |

## How to run it

```powershell
npm run dev        # server on http://localhost:3001, web on the Vite port
```

`node_modules` was copied from sample37, so `npm install` is not needed. Open the **38. Prompt suggestions** tab.
`ANTHROPIC_API_KEY` must be in `.env`. Start it from a normal terminal, not from inside Claude Code (see Tab16). A
scenario costs between $0 (scenario 1) and about $0.045 (scenario 5).

> **Only one sample can run at a time.** Every sample's server uses port **3001**. Stop the other samples'
> `npm run dev` first.

## Step 1: Choose the feature

The request said "implement the following feature sample" with no feature text, as in samples 28 to 37. After
Concept 37, seven `Options` fields of `sdk.d.ts` (`0.3.281`) were still used nowhere in `server/` or `src/`: `betas`,
`maxThinkingTokens`, `onUserDialog`, `projectConfigRoot`, `promptSuggestions`, `resumeDropsTurn` and
`supportedDialogKinds`. **`promptSuggestions`** was chosen:

- It is a real product feature (the grey text in the Claude Code input box) that an app can offer its users.
- It can be shown with real runs, and it has non-obvious rules (the first turn, the cache, the billing).
- The others are deprecated (`maxThinkingTokens`), model-specific (`betas`), about worktrees (`projectConfigRoot`), or
  need a flow the lab cannot trigger on demand (`onUserDialog`, `supportedDialogKinds`, `resumeDropsTurn`).

## Step 2: Read what the course already said

| Read | To learn |
|---|---|
| `Build-steps.md`, `37-permission-prompt-tool.ts`, `Concept37PermissionPromptTool.tsx` | The latest style: strict zod bodies, SSE events, tables of cases run in parallel, `#region` + `/code`, the dry run with a fake process, a summary table |
| `12-streaming-input.ts` | The input queue and the `/session` + `/send` + `/end` routes of a live chat |
| `34-output-styles.ts` | The wire tap (`ANTHROPIC_BASE_URL`), to see what reaches the API |
| `server/index.ts`, `src/App.tsx`, `src/lib/sse.ts`, `src/styles.css` | Mounting, the tab list, SSE, the CSS classes to reuse |

## Step 3: Read the types, the SDK and the CLI

| Found | Used for |
|---|---|
| `Options.promptSuggestions` (at most one per turn, after the `result`; suppressed on the first turn, after API errors, in plan mode, by `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION=false`, by `promptSuggestionEnabled: false`, near the usage limit; "piggybacks on the prompt cache") and `SDKPromptSuggestionMessage` | Every part |
| In `sdk.mjs`: `promptSuggestions` goes into the `initialize` control request, not into the args | Part A |
| In `claude.exe`: the skip reasons (`early_conversation` = fewer than 2 assistant messages, `aborted`, `plan_mode`, `cache_cold` with a limit of `1e4` tokens…) and the filter list (`meta_wrapped`, `evaluative`, `claude_voice`…) | Part F, and the cases of Parts C and E |

## Step 4: Probe before designing

Scratchpad scripts (in a temporary `.probe/` folder) ran real sessions with Haiku 4.5 through a wire tap:

| Probe | Result | Decision |
|---|---|---|
| Three streaming turns | No suggestion after turn 1, one ≈ 0.8 s after the result of turns 2 and 3. The call is the conversation + `[SUGGESTION MODE…]`, same model and tools. Adding the calls' costs shows that each suggestion is billed in the **next** result | Part B shows each result as the sum of its wire calls |
| 12 cases in parallel | Off with the option `false`, the env var, the setting, plan mode; the env var `true` beats the setting; a queued message skips it; a resumed session and a turn with a tool call get one on their first result; `"(silence…)"` is dropped; with a big system prompt the call reads the cache; Sonnet uses Sonnet for the call | Parts C, D, E |
| A single prompt with and without a tool | Without: none. With: a suggestion, although it is the first turn | The "first turn" rule is really "2 assistant replies" (confirmed in the CLI's code) |

## Step 5: Design the concept

- **A wire tap** in the server, so every row and the chat can show whether the suggestion call was made, its usage,
  what it answered, and its cost (Haiku 4.5 prices).
- **A live chat** (Part B), with the suggestion as a chip: *Send it* (the next user message) or *Edit it* (into the
  input box). Bash is limited to `node`, `ls` and `cat`.
- **Three tables** with the same runner: each scripted turn waits up to 6 s for the suggestion (or not, for the
  "already waiting" row). Rows record, per turn, the replies and the suggestion, plus all wire calls.
- **Part F** lists the CLI's rules as they are in the code, marked as undocumented.
- **Routes**: `POST /dry`, `POST /chat` (SSE) + `/send` + `/end`, `POST /when`, `/switches`, `/cache` (SSE, one row
  each), `GET /code`.

## Step 6: Implement it

| File | What was done |
|---|---|
| `server/concepts/38-prompt-suggestions.ts` | New: the tap (`readStream`, `callCost`), `base()`, `inputQueue()`, `recordingProcess()` and `/dry`, `relay()` and the chat routes, `runRow()` and the `WHEN`, `SWITCHES`, `CACHE` rows |
| `src/concepts/Concept38PromptSuggestions.tsx` | New: `WireRow`, `Chat` (the cost breakdown per result, the chip), `RowsTable`, Parts A to H |
| `server/index.ts`, `src/App.tsx`, `src/styles.css`, `.gitignore`, `Tab1-query().md` | Mount, tab, `.tag-suggest` and `.suggestion-chip`, `suggest-lab/`, table row and tree |

`npx tsc --noEmit -p .` passed.

## Step 7: Test the routes

A scratchpad script mounted **only** the Concept 38 router on port **3138**, with `.env` loaded and the `CLAUDE*`
variables removed. `drive.mjs` printed each row, and `chat.mjs` drove `/chat`, sending each suggestion back with
`/send`.

| Test | Result |
|---|---|
| `/dry` | The three rows of the doc's Step 2 |
| `/chat` + `/send` + `/end` | The timeline of the doc's Step 3; *"run it with node hello.js"* ran; the result costs matched the wire calls |
| `/send` with a bad id, an unknown chat; `/end` of an unknown chat | 400, 404, 404 |
| `/when`, `/switches`, `/cache` | The rows of the doc's Steps 4, 5 and 6 |

What the tests changed:

- The cached system prompt was **11,000 tokens**, not the 7,500 the label said. It is now about 5,000 (still over
  Haiku 4.5's 4,096 minimum), and the other two cache rows were resized to stay over 10,000.
- The rows' `total_cost_usd` **missed the last suggestion**. Each row now also shows the sum of its wire calls and
  the part never reported.
- The chat suggested **"commit this"**, and the model tried `git add` in the course's repository. It was denied
  (`git` is not an allowed tool). The hint of Part B and the doc now say so.

## Step 8: Run it in the real app

Vite ran on port **5199** with a scratchpad config whose proxy pointed to the 3138 server. Headless Edge was driven
through the DevTools protocol (profile in `%TEMP%\c38-edge`): tab 38, run 1, run 2 (*Send it* on the first
suggestion, *Edit it* on the second, then *End*), and run 3.

| Check | Page |
|---|---|
| Open tab 38 | 9 code buttons |
| 1 | The three-row table |
| 2 | Chips *"run it"* and *"commit this"*; *Edit it* put the text in the input box; the second result showed *"#4 suggestion + #5 main + #6 main · the previous turn's suggestion is billed here"* |
| 3 | 7 rows; the "already waiting" row caught one suggestion call **aborted** mid-flight |
| Console | No error |

What using the app changed:

- **The chat could not be ended from where you are.** *End the chat* was only next to *Start the chat*, far above a
  long timeline, and Parts C to E stay disabled while the chat is open. There is now an *End the chat* button next to
  *Send*, and a line that says how the chat ends.
- **A dropped suggestion looked like a wait.** After a *"(silence)"* answer nothing more comes, so the tab seemed stuck.
  The server now sends a `dropped` event when a suggestion call brings no `prompt_suggestion` within 2 s, and the chat
  shows it.

All test processes were stopped, and `suggest-lab/`, `.probe/` and the Edge profile were deleted.

Costs: about $0.3 for all the probes, route tests and the browser test.

## Files added or changed

| File | Change |
|---|---|
| `server/concepts/38-prompt-suggestions.ts` | New: the Concept 38 routes, the wire tap, the tables of cases |
| `server/index.ts` | Mounts `/api/c38` |
| `src/concepts/Concept38PromptSuggestions.tsx` | New: the Prompt suggestions tab |
| `src/App.tsx` | Tab |
| `src/styles.css` | Suggestion rows and chip |
| `.gitignore` | `suggest-lab/` |
| `Tab1-query().md` | Adds Concept 38 to the table and the tree |
| `Tab38-Prompt-suggestions.md` | Explanation of the concept |
| `readme.md` | Same content as `Tab38-Prompt-suggestions.md` |
| `Build-steps.md` | This file |
