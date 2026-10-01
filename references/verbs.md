# Verdesk — six verbs

Verdesk gives you **six verbs**. You say *what* you want; Verdesk picks the channel (the page's DOM, a window's controls, or on-screen text) and tells you which in the first block of every answer: `{"via":"browser_click","rule":"act.browser.click.target","tier":"browser"}`. Read `via` when something fails — it says what Verdesk tried.

| Verb | You say | Example |
|---|---|---|
| `see` | what is here | `see()` · `see({target:"#total"})` · `see({pixels:true})` |
| `act` | do this | `act({target:"@e7"})` · `act({label:"Save"})` · `act({target:"@e4", value:"Ana"})` · `act({key:"Ctrl+S"})` |
| `go` | take me to | `go({to:"example.com"})` · `go({to:"back"})` · `go({to:"window-title:Excel"})` |
| `wait_until` | until this is true | `wait_until({text:"Saved"})` · `wait_until({target:".spinner", state:"hidden"})` |
| `remember` | record / replay a task | `remember({op:"suggest"})` · `remember({op:"replay", task:"send invoice", args:{to:"Ana"}})` |
| `status` | what can I do here | `status()` · `status({about:"targets"})` |

## The loop — this is where runs are won or lost

**See once → act → read what the action returned → see again only if you must.**

- **`act` already tells you what changed.** On the web its result carries `changes`: the controls that APPEARED, with fresh `@eN` refs. Act on those directly. Calling `see()` again to "double-check" what `changes` already said is the #1 waste of time and tokens.
- **Batch.** Several actions you already know go in ONE `act({steps:[…]})`. A form of 8 fields is **one call, not eight** — this is the biggest speed edge you have:
  ```
  see()
  act({steps:[
    {target:"@e8",  value:"Diego"},
    {target:"@e9",  value:"29"},
    {target:"@e10", value:"Plum"},     // a <select>/dropdown: pass the OPTION as value
    {target:"@e16"},                   // a checkbox: just click it
    {target:"@e7"}                     // Submit
  ]})
  ```
  It stops at the first step that did not land and says which (`failed_step`, `reason`).
- **Pixels only at a wall.** `see({pixels:true})` is for what text cannot tell you: a canvas, a captcha, an icon with no name, a color. Everywhere else the text answer is cheaper and exact.

## Web

- `go({to:"https://…"})` or just the domain. It switches to Verdesk's browser if you were on the desktop. The browser keeps a **persistent profile**: check whether you're already logged in before typing a password.
- `see()` = the page as text: every control on screen with an `@eN` ref, role and name. Controls below the fold are counted in a trailing line — `act({scroll:"down"})` to reach them.
- **`ctx="…"` is the truth.** When controls share a name, or a field's `name` differs from its visible label, the snapshot shows `ctx` (nearby label, color, or position). If ANY field has a `ctx`, the form is crossed on purpose: fill EVERY field by its `ctx`, from the first attempt.
- **Refs die when the screen changes.** After a submit, "Next" or navigation, use the refs from the last `changes`, or `see()` once. Never fire an old ref into a new screen.
- Read exact text (a code, a price, an order number): `see({target:"#code"})` — straight from the DOM, no misreads. Never read digits off an image.
- No ref? `act({label:"Billing"})` finds it anywhere on the page, even off-screen. If the label matches several things it **fails with the candidates** — pick one with `target`, or pass `nth`.
- Something appears later: `wait_until({text:"Ready"})` or `wait_until({target:".spinner", state:"hidden"})`. `target` here is a CSS selector, not an `@eN`. Never sleep blind.
- Overlays don't block you: acting on a field operates the DOM node itself. Only if the result says `no_effect`, close the overlay (the result names it) and retry.
- Tabs: `status({about:"tabs"})` → `go({to:"tab:2"})`.

## Desktop

- `go({to:"window-title:Notepad"})` points your view at that window **and gives it keyboard focus**. `status({about:"targets"})` lists windows and monitors.
- `see()` = on-screen text grouped by region, each with an id. `see({target:"<id>"})` zooms into that region.
- `button:true` = a real button (Verdesk saw its box). `text:""` + `button:true` = an **icon button**: click it with `act({target:"<id>"})`, see what it is with `see({target:"<id>", pixels:true})`.
- `act({label:"Guardar"})` clicks by the visible name: a learned object first, then the window's real control, then the on-screen text. When your view is a window, Verdesk gives it keyboard focus before every `act`.
- Waiting: `wait_until({label:"Guardar como", state:"visible"})` waits for that dialog/control to appear; `wait_until({label:"Aceptar", state:"enabled"})` until it can be pressed.
- Writing: `act({label:"Nombre", value:"Ana"})` sets the field whose control is named that. If no single field has that name it fails and lists the names it did find — use one of those. `act({value:"hello"})` types into whatever has focus.

## Anywhere

- Keys: `act({key:"Enter"})`, `act({key:"Ctrl+Shift+Tab"})`. Scroll: `act({scroll:"down", amount_px:800})` (default 500).
- **One agent at a time.** If an answer says Verdesk is in use by another agent, nothing was done: wait (it frees itself 3 minutes after that agent's last call) and check `status()`. When your task is done, `status({release:true})` frees it for the next agent.
- A failure is information. Read `via` and `reason`, change the approach — repeating the same call is the most expensive move there is.
- Repeated tasks: `remember({op:"suggest"})` first. If a saved train fits, `remember({op:"replay", task, args})`. Otherwise do it once between `remember({op:"record", task})` and `remember({op:"save"})`.

## Rare cases — force a channel

Every verb takes `via` + `via_args` to run one specific underlying tool with its raw arguments. You rarely need it. The cases that do:

- A native `alert`/`confirm`/`prompt` froze the page (the action's result has `dialog`): `act({via:"browser_handle_dialog", via_args:{accept:true}})`. Answer it before anything else.
- Move a password without seeing it: `act({via:"browser_transfer", via_args:{from:"@e3", to:"@e9"}})`.
- Upload a file: `act({via:"browser_upload", via_args:{target:"input[type=file]", paths:["C:\\f.pdf"]}})`.
- Drag: `act({via:"browser_drag", via_args:{from:"@e5", to:"@e9"}})` — check `moved`, not `ok`.
- Find every item matching a phrase plus the buttons in its row (upvote all, delete all): `see({via:"browser_find_text", via_args:{query:"…"}})`, then one `act({steps:[…]})` with their selectors.

Each verb's description lists the tools it accepts in `via`.

## Tone
The user wants the task done, not narrated. When it's done, say what you did in one sentence plus the result.
