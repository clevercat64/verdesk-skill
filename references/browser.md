# Verdesk — browser (web)

The DOM is your sight. `browser_snapshot` hands you the page as cheap text with `@eN` refs — **act by reference, batch with `run_steps`, pixels ONLY at a wall.** Read this before your first browser action.

## Recipes — find YOUR task here first

Each recipe is the whole move, end to end. Copy the shape, swap the refs. The reference below explains every field; you rarely need it to act.

### Fill a form (any number of fields, including dropdowns) — ONE call

```
browser_snapshot()                       // read the refs once
run_steps({steps:[
  {tool:"browser_fill", args:{target:"@e8",  value:"Diego"}},
  {tool:"browser_fill", args:{target:"@e9",  value:"29"}},
  {tool:"browser_fill", args:{target:"@e10", value:"Plum"}},   // <select>: pass the OPTION, same tool
  {tool:"browser_click",args:{target:"@e16"}},                 // checkbox
  {tool:"browser_click",args:{target:"@e7"}}                   // Submit
]})
```

**A `<select>` / dropdown / combobox is `browser_fill` with the option as `value`.** There is no separate "select option" tool, and `browser_select_tab` is unrelated (it switches BROWSER TABS). Dropdowns batch like any other field — twelve of them still go in one `run_steps`. `value` is the option's `value` attribute OR its visible text when exactly one option has that text (two options with the same text → an error listing their values; no option matching → an error listing the options; either way the select is left as it was). A closed select does not render its options, so read them with `browser_describe({target:"@e10"})` → `options:[{value,text,selected}]` (max 50).

**Never fill one field per call.** Eight fields is one call, not eight. This batching is the single biggest speed edge Verdesk has; walking on eggshells field by field throws it away.

If any field shows a `ctx="…"`, the form's labels are crossed on purpose: **fill EVERY field by its `ctx`, from the first attempt.** Correcting afterwards is a second pass you may run out of time for.

### A control lives inside an iframe (payment widgets, embedded chat, comment plugins, captchas)

```
browser_snapshot()
// - Iframe "payment-frame" [ref=e3]
//   - textbox "Card number" [ref=e5]
//   - button "Pay" [ref=e4]
run_steps({steps:[
  {tool:"browser_fill", args:{target:"@e5", value:"4111111111111111"}},
  {tool:"browser_click", args:{target:"@e4"}}
]})
```

**Iframes are transparent — there is no special tool and nothing to set up.** `browser_snapshot` inlines an iframe's controls right under its `Iframe` line, indented one level, with normal `@eN` refs. Act on them exactly like any other ref — `browser_click`/`browser_fill`/`browser_type` resolve the correct frame on their own; you never need to know a control lives inside an iframe. This works for **same-origin AND cross-origin** iframes (a real Stripe/PayPal Elements widget, a chat plugin, an embedded review widget) — no dedicated setup, no extra call.

A field inside the iframe and a same-named decoy outside it never collide, even if both say e.g. "Card number" — they're different DOM nodes and always get distinct `@eN` refs.

**Two real limits, so you don't chase a ghost:**
- Only **one level** of nesting is inlined. An iframe nested inside another iframe (rare — e.g. a bank's 3-D Secure challenge popped inside a payment iframe) is not expanded; its controls won't appear.
- Tools that take a **bare CSS/xpath selector instead of a `@eN` ref** — `browser_get_text`, `browser_find_text`, `browser_wait_for` — only reach the MAIN document. They cannot read or wait on something that lives only inside an iframe and isn't itself a control with a ref. To read iframe-only text tied to a control, use `browser_describe({target:"@eN"})` on that control's ref instead.

### Log in to a site

The password is the person's. Let them type it on the page; if they gave it to you for this task, fill it once with `browser_fill` and do not repeat it anywhere. Verdesk's Chrome keeps a **persistent profile** (a session survives across launches), so check first whether you are already in. If not, fill the user name, let the person type the password on the page, then click sign-in:

```
browser_snapshot()
browser_fill({target:"@e4", value:"user@example.com"})   // replaces anything the browser autofilled
// the person types the password on the page (or browser_fill once if they gave it to you)
browser_click({target:"@e6"})
browser_wait_for({condition:{kind:"text", text:"Dashboard"}})   // name what proves you're in
```

To repeat the login, record it as a macro whose password slot the person or the vault fills (trains, section 7). While you record, `browser_fill` the user name and password as `{user}` and `{password}` and the person types both in Verdesk's window, never in front of you (trains, section 4).

**Moving a password you must NOT see** (a value sitting in another field, or behind a "Copy" button): `browser_transfer({from, to})` moves it DOM→DOM and returns only `{ok, value_len}` — the secret never passes through you.

### Confirm what you did actually took effect

Doing the action is not proof of the result. After a submit / save / toggle, name what proves it:

```
browser_wait_for({condition:{kind:"text", text:"Saved"}})        // a confirmation appeared
browser_wait_for({condition:{kind:"hidden", target:".spinner"}}) // the pending state cleared
```

For a click, you often need nothing extra — `browser_click` already returns `state` (the element's resulting `checked`/`value`/`selected`/`disabled`) and `changes`. A checkbox or toggle is verified **in the same call**; don't add a read-back you already have.

If nothing confirms and nothing changed, that's `no_effect:true` — read the page's error text (it comes back under `evidence[].texts`) instead of clicking again.

### Get to a section of a site you don't know

```
navigate({url:"https://example.com"})
browser_snapshot()                            // header/nav/search are NEVER demoted — they're in the main tree
browser_find_text({query:"Billing"})          // not on screen? one call finds it anywhere on the page
```

Don't scroll-and-snapshot hunting for a link. And don't screenshot to "see the layout" — the snapshot already lists every interactive control with its role and name.

### Click something and use what appeared — WITHOUT looking again

```
browser_click({target:"@e6"})
// → {changes:{new:[{ref:"@e7",role:"button",name:"Submit"}], summary:"1 new"}}
browser_click({target:"@e7"})            // act on the fresh ref. NO second snapshot.
```

`changes` is the delta. Re-snapshotting to "double-check" what `changes` already told you is the #1 waste of both tokens and time. If the click navigated (`url_changed:true`), `changes` is just `{page_changed:true, gone_count, new:[first 8], new_total?}` — everything from the old page is gone, so snapshot the new page. Otherwise `gone` lists at most 8 (`gone_total` says if there were more) and never `<option>`s.

### Read a code / text the page shows you

```
browser_get_text({target:"#code"})       // exact DOM text — zero misread risk
```

Use this for anything rendered as TEXT: an access code, an order number, a price, instructions next to a field. **Only if the characters are drawn inside a `<canvas>` or an image — a real captcha — is there no DOM text.** Then:

```
look({zone:{kind:"selector", css:"#captcha"}})   // scopes to that element's box, returns TEXT
```

**Scoping by selector is the move — don't guess pixel coordinates.** `look` with a `selector` zone takes the element's bounding box for you and, by default, returns only the text and layout layers (no image pixels at all). The characters come back as a **plain string you can reason on**, so **this works even if your model has no vision whatsoever.**

**If you don't already know the selector, do NOT go guessing them one by one.** `canvas`, `img`, `svg`, `.stage`, `.card` — trying selectors blind is how runs die on the clock: each miss costs a round-trip, and a captcha is just as often a styled `<div>`, a background-image or a web component. **Guessing selectors is the trap here, not reading the characters.**

When you don't know where it is, **stop guessing and just look at the whole screen once**:

```
look()                          // no zone: whole viewport, text + layout layers, no pixels
```

That single call gives you the text it can already read plus the page's **layout zones with ids** — that is, *where things are*. Now scope to the zone that holds what you want, addressed by the id `look` just handed you, with no selector involved:

```
look({zone:{kind:"around_layout_zone", id:"…"}})   // the id came from the call above
```

Use a `selector` zone only when you **already** have the selector for free — it was in the snapshot, or the task named it. One `look()` beats four selector guesses, every time.

Raw coordinates are the last resort, when nothing above gave you a handle:

```
read_text({target:{kind:"region", rect:{x:400,y:300,w:400,h:150}}})
```

Reach for `screenshot` (with `max_dim` to enlarge) **last**, and only when the text layers came back empty or clearly wrong — and remember it returns pixels, which are useless to you if your model can't see images.

### Extract data from a page (read content, not controls)

The snapshot lists **controls only** — not the prose, prices, rows or labels around them. That text lives in the DOM:

```
browser_get_text({target:"#order-summary"})      // exact text of a container, in DOM order
browser_describe({target:"@e12"})                // one control, whole picture: role + text (or name/visible_text+mismatch if they differ) + value
browser_get_value({target:".row", count:true})   // how many rows match, without counting by hand
browser_get_value({target:"@e8", attr:"href"})   // an attribute the a11y tree doesn't expose
```

Scope the read to the **container** of the task (the `<form>`, the card, the section) — not the whole `<body>`, which may carry decoy text from elsewhere on the page. DOM text is exact and arrives in DOM order: cheaper and truer than looking at pixels. Line the container's text up, in order, with the controls in order: if it maps cleanly, act; if a label seems to belong to another field, the DOM order was decoupled from the layout on purpose (the lying-label trap) — only THEN `look()`.

### Work across several tabs

```
browser_tabs()                        // → {tabs:[{id,index,title,url,active}]}
browser_tabs({url:"https://…"})       // opens a NEW tab (http/https/about:blank), NOT selected → {ok,id,index,url,active:false}
browser_select_tab({id:"…"})          // every browser tool now operates on that tab
```

`id` (CDP targetId) is authoritative, `index` is a best-effort 0-based position — pass exactly one. If Chrome had discarded the tab it comes back `revived:true` — its JS state is gone, treat it as a fresh page; `dialog_blocked:true` means the tab has an unresolved native dialog (clear it with `browser_handle_dialog`). **Resolve any pending native dialog BEFORE switching away**, or it can become unresolvable (see below).

### A native dialog appeared — the page is frozen until you answer

```
browser_click({target:"@e5"})
// → {dialog:{kind:"confirm", message:"Delete this item?"}}   ← returns immediately, does NOT hang
browser_handle_dialog({accept:true})              // or accept:false to cancel
```

While an `alert`/`confirm`/`prompt` is open **every other tool on that tab hangs**. The action that triggered it returns right away with `dialog` set instead of waiting on you — answer it first, before anything else. For a `prompt`, pass `prompt_text` (used only when accepting). Returns `{ok, accepted}`; errors if no dialog is open.

### Tell apart controls that share a name

The snapshot already disambiguates them for you with `ctx` — by nearby label, by **color** (`ctx="green #22c55e"`), or by **position** (`ctx="bottom-right"`). Pick by that, never guess between same-name refs, and never spend a screenshot to "see which is which".

```
// three identical "Submit" buttons → the snapshot shows them with different ctx
browser_click({target:"@e7"})            // the one whose ctx names the form you filled
```

### Find something that is NOT on screen

```
browser_find_text({query:"Order #4417"})     // searches the WHOLE page, one call
browser_find_text({role:"button", label:"submit"})   // icon-only button, no visible text
```

Do NOT scroll-and-snapshot hunting for it. `browser_find_text` also returns the actionable controls in each match's row, so "find all X and click their button" is find + one `run_steps`.

### Wait for something instead of guessing a sleep

```
browser_wait_for({condition:{kind:"text", text:"Ready"}})
browser_wait_for({condition:{kind:"hidden", target:".spinner"}})
```

`target` here is a CSS selector, **not** a `@eN` (you're waiting for something not in the snapshot yet). Other kinds: `{kind:"visible", target}`, `{kind:"enabled", target}` (visible and not disabled), `{kind:"text", text, target?}`. Returns `{ok, waited_ms, error_type?}` (`timeout` if it never held; `timeout_ms` default 5000, polls every 100 ms).

### Drag, and confirm it actually landed

```
browser_drag({from:"@e5", to:"[aria-label='drop-zone']"})
// → {ok:true, moved:true}   ← `moved` is the truth, not `ok`
```

`ok:true` only means the gesture ran. **`moved:false` means the drop was rejected.** Before retrying, suspect an overlay: clicks go through the DOM node, but a drag travels real screen coordinates, so a layer that ignores clicks can still eat the drag. Two things are now caught BEFORE any mouse event fires, as a clear error instead of a bad drag: a stale `@ref` (`stale_ref: ...` — re-snapshot and retry, same fix as any other stale ref) and an unrelated element covering `from`/`to`'s center point (named in the error — close/dismiss it, then retry).

### Something failed — read the verdict, don't blind-retry

```
run_steps({...})  // → {ok:false, failed_step:3, tool:"browser_fill", reason:"…"}
```

The result names which step broke and why, and stops there. `stale_ref` means your refs are from a previous screen — re-snapshot and act on fresh ones, UNLESS an earlier step in the same `results` array already flagged `dom_changed:true` (a `browser_fill`/`browser_click` that re-rendered the page): its own `changes.new` already has the fresh refs — read that before paying for a new snapshot. A rejected `fill` means wrong ref, an option that doesn't exist, or a length/format constraint. The failure is information: read it, then change the approach — repeating the same call is the most expensive move available.

## Gotchas (read first)
- **Re-looking after a change is the #1 cost.** A `browser_click` already returns `changes` (what appeared, with fresh `@ref`s). Act on it — do NOT `browser_snapshot`/`look()` to "double-check".
- **The DOM `name` can LIE; the visible label (`ctx`) is the truth.** On a form field, route by `ctx="…"`, never by `name`. **If ANY field carries a `ctx`, the form is crossed on purpose — fill EVERY field by its `ctx` from the FIRST attempt. Never fill by `name` and correct after: the correction is a second pass you may be timed out of.** Don't `look()` to "see" the label — the `ctx` already is it.
- **A person can use this Chrome too.** If a `browser_click`/`browser_type`/`browser_fill`/`browser_transfer`/`browser_drag`/`browser_upload` answers `error_type:"user_changed"`, someone clicked, typed or scrolled in the browser window after your last `browser_snapshot`: it did NOTHING, and a fresh `browser_snapshot` is attached. Read it (a field they filled shows the new value; an `@eN` still on the page keeps its number) and repeat. Reads (`browser_get_*`, `browser_find_text`) do not count as a new view.
- **An overlay over the page does NOT block you.** A DOM action (`browser_click`/`browser_type`/`browser_fill`) operates the node itself and goes THROUGH the visual overlay — and acting on a background field often dismisses the popup (the AliExpress pattern: type in the search, hit Enter, the popup is gone). Act on the background element directly by its `@ref`/selector; do NOT assume it's inert. ONLY if the action reports no visible effect, close the overlay (its close control is named in the error) and retry. Ads/cookie-banners re-appear mid-task — re-check.

## Enter the browser
- `set_view_target("browser")` — Verdesk's own Chrome over CDP: **fast, synthetic input (never touches the user's mouse or clipboard), persistent profile** (log into a site once → the session sticks across launches). This is the default for web work. **Call it FIRST on a fresh session** — until you do, the active surface may be the desktop tier and `navigate`/browser actions come back "operation not supported by this surface backend".
- `navigate({url})` loads a URL · `back()` · `forward()` · `reload()`.

## See — `browser_snapshot()`
The page's accessibility tree: interactive controls as `@eN` + role + name — not selectors. Cheap text, no pixels — **this is your sight, not `look()`**.
- **It's the NOW visible — the windshield, not the whole page.** The snapshot lists only the controls inside the viewport; a trailing line `(+N controls off-screen below the visible viewport — act by @ref (auto-scrolls into view) or scroll to reach them: …)` names up to 8 of the ones below. Act on a named one by its `@ref` (it scrolls into view by itself) or `scroll`; re-snapshotting in place will NOT reveal more. But to FIND a specific thing off-screen — an item, a row, a phrase anywhere on the page — don't scroll-scan: **`browser_find_text({query})`** jumps straight to every match (+ its row controls) in one call (see "Timed stages & feeds"). This is the point: you read the *current* screen, not a tree that grows without bound as a long/stacked page piles up (that growth was the old #1 token sink). A `position:fixed/sticky` element (cookie-bar, FAB, chat widget) stays in the tree no matter the scroll — it's always on screen.
- **A trailing `(secondary [footer·decoy] — demoted; act by @ref if needed: @e30, @e31 …)`** lists controls demoted out of the main tree: footer links and tiny decoys. They're not load-bearing — ignore them, but they're still actionable by the `@ref` shown if you actually need one. The header/search/login and side tools are NEVER demoted; they stay in the main tree.
- **`ctx="…"` disambiguates — text, then color, then position.** It's the nearby visible label (duplicate/icon buttons; on a form field it shows only when it differs from a misleading `name`). When the text can't tell same-name controls apart, `ctx` becomes their distinguishing **color** (`ctx="green #008000"`) or **position** (`ctx="top-left"`, `"top"`, `"top-right"`, `"left"`, `"center"`, `"right"`, `"bottom-left"`, `"bottom"` or `"bottom-right"` — the 3×3 region of the viewport). So three identical "Select" buttons read as three distinct refs: pick "the `green` one" or "the `bottom-right` one" by reference, no pixels. Route by `ctx`; never guess between same-name refs.
- Also listed: **drag/drop targets** (`draggable`/`droptarget` hints) and **aria-labeled containers** like drop-zones (a `<div aria-label="…">` shows up as a named `@ref`) — target them by reference, no selector hunting.
- **A trailing `(batch: N contiguous form fields detected — fill them in ONE call instead of N. ...)` block** appears when a `<form>`/other ARIA landmark has ≥3 fillable fields (`textbox`/`searchbox`/`spinbutton`/`combobox`) — real ownership, not adjacency: two different forms sitting right next to each other never merge into one batch, even with nothing visible between them. (Fields with no landmark at all fall back to simple on-screen adjacency.) It ships a ready-to-fire `run_steps({...})` — one `browser_fill` per field, each `value` a placeholder named after its field (`"<Email>"`) — swap in the real values per field and call it as-is instead of filling one by one. Capped at 5 groups; beyond that a `(+N more contiguous field group(s) detected — not shown, same batching pattern applies to each)` line.
- **A trailing `(twins: N × role "name" indistinguishable by text — ...)` block** fires when ≥2 controls share the same role + name and none resolved a `ctx` (above) to tell them apart, listing up to 8 on-screen refs (`+N more` beyond that). **Reaches the WHOLE page, not just what's on screen** — a duplicate scrolled out of view still counts: the block adds `(+N more off-screen — do not assume these are the only matches)`, or, if every match is currently off-screen, says so outright (`exist further down the page — all N currently off-screen`) with no ref to act on yet. When it fires: don't guess between the listed refs — ask or look (scroll / `browser_find_text`) before acting on either.
- When a control is **still** ambiguous (an icon with no text, or same-name buttons in the *same* region) the snapshot auto-attaches a small **thumbnail** crop so you SEE which is which — the only pixels it spends, and only when neither text nor position disambiguates.
- **A trailing `(page skeleton: ...)` block**, on pages that have ARIA landmarks with real content — `banner`/`navigation`/`main`/`complementary`/`contentinfo`/`search`/`form`/`region`, each numbered and listing the `@ref`s it contains (nested landmarks shown indented). Absent on pages without landmarks — nothing to parse, nothing lost. **The number is a scope filter, not a second way to address a control** — you still act by `@ref`. To narrow a LATER `browser_snapshot({region: N})` call to just that landmark's controls (no skeleton, no off-screen/secondary tails — a smaller read when you already know which region you need): pass the number from a prior call. Numbers can go stale if the page changed — a bad `region` errors asking you to re-snapshot for fresh ones.

## Act
- `browser_click({target, button?})` — `@eN` or a CSS / `xpath=` selector. It always clicks: there is no preview mode (`dry_run` is not a parameter). Operates the DOM node (`element.click()`), so it fires SPA routers and React/delegated handlers a synthetic pixel click would miss. Returns `{ok, url_changed, dom_changed, no_effect, dialog?, state?, changes, url}`:
  - **`changes` = the DOM delta (the killer feature).** `new:[{ref,role,name}]` (controls that APPEARED — act on these directly), `gone`, and a one-line `summary` like `"1 new: button 'Next'"`. `{}` = nothing structural changed; `no_effect:true` = ran but no visible effect (wait/retry or re-check the ref).
  - **On `no_effect:true`, `hint` names the DETERMINED cause** — element detached from the page (re-snapshot for a fresh `@ref`), disabled, covered by another element (named), or the page still loading — never a list of guesses. If none of those checks pinpointed it, `hint` says so plainly instead of pretending.
  - **`state`** — the touched element's RESULTING state right after the click: `{checked, value, aria_pressed, aria_selected, selected, disabled}` (null on a right-click). A checkbox/toggle/radio/select click is verified in the SAME call — no separate read-back needed.
  - **`dialog`** — if the click opens a native `alert`/`confirm`/`prompt`/`beforeunload`, the page freezes and the call returns immediately with `dialog:{kind,message,default_prompt}` set instead of waiting. Resolve it with `browser_handle_dialog` before doing anything else on that tab (see "Native dialogs" below).
  - ex: `browser_click({target:"@e3"})` → `{changes:{new:[{ref:"@e7",role:"button",name:"Next"}],summary:"1 new: button 'Next'"}}` → press `@e7` **with no re-snapshot**.
- `browser_type({target, text, submit?})` — focus + real keystrokes (`submit:true` presses Enter after typing). To overwrite a field, `press_key({key:"a", ctrl:true})` first. Verifies itself against what the field had BEFORE typing (never against the literal text — a mask/normalizer, or typing into a non-empty field, legitimately changes the result): no change at all → `ok:false, no_effect:true, hint` (same shape as `browser_click`'s no-op — a DETERMINED cause, not a guess); some change → `ok:true, no_effect:false, outcome:"confirmed"` (landed exactly) or `"unconfirmed"` (a mask/normalizer reshaped it — read the field with `browser_get_value` if the exact value matters). On a field the browser already autofilled it can answer `no_effect:true`, and a recording keeps the step anyway: use `browser_fill`, which replaces the value. If typing triggers a re-render it returns `dom_changed:true` + `changes`, same as `browser_click`/`browser_fill` — read `changes.new` for fresh `@ref`s instead of re-snapshotting.
- `browser_fill({target, value})` — set `.value` directly + input/change/blur events. The way to load `<input type=date|time>`, `<select>`, and React-controlled fields that ignore typed keys. ex: `browser_fill({target:"@e9", value:"2026-06-22"})`. If the value didn't stick, `evidence[0].note` names why — detached, disabled, read-only, covered, or the page still loading — not just `ok:false`. A mask that only adds separators (`9 3913 8855` shown as `(9) 3913-8855`) still counts as stuck: `ok:true` with `evidence[0].formatted:true`; a changed `.`/`,`/leading `-` is never formatting (`ok:false`).
  - **If the fill triggers a re-render** (live validation, a React state update) it returns `dom_changed:true` + `changes` — same shape as `browser_click`'s. The rest of a `run_steps` batch may now hold stale `@ref`s from BEFORE this fill; read `changes.new` for fresh ones instead of re-snapshotting.
- `browser_transfer({from, to})` — move a value DOM→DOM **without it passing through you** (secrets; "Copy" buttons work too). Returns `{ok, value_len}`, never the value.
- `browser_upload({target, paths})` — attach files: `target` = the `<input type=file>` ITSELF (usually hidden — a CSS selector like `input[type=file]`), `paths` = absolute file paths. A click on an "Attach" button only opens the Windows file dialog: prefer `browser_upload`, which needs no dialog. No reachable input but a drag-and-drop zone? `how:"drop"` with `target` = the zone drops the files with real drag events and answers whether a drop event with the files reached the page (`ok:false, error_type:"no_drop_event"` if not — Verdesk cancels Chrome's default action, so the file is NOT opened in a tab; if a stray `file:` tab still appears it is closed and listed in `closed_tabs`; what the page does with them afterwards is NOT checked, so look at it). Pasting files is not supported. In a recorded macro each path is saved as the slot `{archivo}`, `{archivo_2}`…: the replay asks for the file.
- `browser_drag({from, to})` — **drag one element onto another BY DOM REFERENCE.** `from`/`to` are `@ref`s or CSS/`xpath=` selectors; the engine resolves both, scrolls them in, and synthesizes the mouse drag. **No coordinates.** For mouse-based DnD (sortables, sliders, kanban, drop-zones). A labeled drop-zone now usually shows up in the snapshot as a named `@ref` (so prefer that); an UNlabeled drop-zone the snapshot can't surface is still reachable by selector: `browser_drag({from:"@e5", to:"[aria-label='drop-zone']"})`. Returns `{ok, from, to, from_xy, to_xy, moved, final_xy}`. **`ok:true` means the drag GESTURE completed — NOT that the drop landed.** Check `moved` instead: `false` means `from` is still at its original position — the drop was rejected, retry or investigate; `null` means `from` no longer resolves (common in sortables that replace the source node on success — indeterminate, not a failure).
  - Widgets that listen for **native HTML5 drag events** (`dragstart`/`dragover`/`drop`) report `moved:false` however often you retry: use `drag_path` (OS-level input) — see "No DOM at all".

## Read a control — `browser_describe({target})` (the canonical reader)
One call returns the whole picture of a control: `{role, value?, ...}` — its role, its `.value` when it's an input, and its text. The text field does the comparing FOR you instead of handing you two blocks to check by eye: when the accessible name and the visible text agree (the common case) you get one field, `text`; when they genuinely differ — a field whose visible label doesn't match what a screen reader would announce — you get `name` + `visible_text` + `mismatch:true` + a `note`, because that discrepancy IS the finding, not something to notice yourself. **This is the default when you need to KNOW one control** (what a button really says, what a field holds, whether the visible label matches the accessible name). One round-trip instead of guessing from the snapshot or stacking `get_text` + `get_value`.
- `browser_get_text({target})` — *narrow:* the exact `innerText`/`textContent` of a NON-control node (a paragraph, a code, a number). Never read a code off `look()`'s text layer (`87098`→`87898`, `O`↔`0`).
- `browser_get_value({target, attr?, count?})` — *narrow:* an element's `.value` when the a11y tree masks it (a revealed `<input type=password>`). To move a secret without seeing it → `browser_transfer`. Two extra modes on the same call: `attr` reads an HTML attribute (`data-*`/`aria-*`/`href`/`src`/anything the a11y tree doesn't expose) instead of `.value`. `count:true` treats `target` as a selector and returns how many elements match it (`{ok, target, count}`, ignores `attr`) — "how many rows/items are there" without counting by hand.

## Timed stages & scroll-heavy feeds (where runs bleed time and tokens)
- **Timed stages show a countdown.** Need the seconds left? `browser_get_text` the timer/countdown element. When time is short, trust the DOM read and submit — verification you don't have time for is a loss, not safety. The fields you fill and the code you read came from the exact DOM; they don't need a screenshot to confirm.
- **Act-on-many across a long feed (every item matching X — upvote, open, select, delete).** Don't scroll + snapshot + scan the whole feed — that's the #1 token sink here. Use **`browser_find_text({query?, role?, label?, nth?, exact?, limit?})`**: it searches the WHOLE page (not just the viewport) in ONE call and returns each match's `selector` PLUS the actionable `controls` in its row (the upvote/like/delete button next to it), each with its own selector. So you find all the matches and click their controls in ONE `run_steps` — near-zero tokens, no scrolling. Only RENDERED text is searched, so **expand any collapsed / "show more" sections first** (one `browser_snapshot` finds them, one `run_steps` clicks them all). Match by the EXACT phrase; near-misses (a synonym, an extra or missing word, a flipped negation) are decoys.
- **`role`/`label` — the semantic locator for a control with no visible text.** `query` matches visible text; `role` matches the exact ARIA role (`button`, `textbox`, …, explicit or the tag's implicit one); `label` matches the accessible label (`aria-label`, an associated `<label>`, placeholder, title, or alt) — combine any of the three (all given ones must match, at least one required). Finds an icon-only "Submit" button by `role:"button", label:"submit"` even though it has no `query`-matchable text. `nth` (0-based) grabs one specific occurrence directly instead of a second call.

## Pixels — only at a wall: `screenshot({target|rect, max_dim?})`
When the DOM has nothing to give — a `<canvas>`/captcha, a chart, an image, an icon to know by shape. Returns ONE rendered crop you SEE; a `target` selector/`@ref` is scrolled into view first (scroll-safe). **`max_dim` = zoom** (upscales a small element up to 4×): a tiny captcha → re-take with `max_dim:1200` and read the enlarged image, don't guess. This is Verdesk's own CDP rendering — monitor-agnostic, NOT an OS screen grab and NOT pixel-hunting with fixed coords.

## Recover — read the F1 verdict, then adapt (don't re-do)
The fast path (`snapshot → @eN → run_steps`) is the opening move; a page may be built to break it, and the failure is information you READ:
- `no_effect`/`noop`/`ok:false`/an error → it did NOT do what you assumed. Read `evidence` + the page's error/alert text (alerts/toasts come back under `evidence[].texts`) — they say WHY. Don't blind-retry the same call.
- a form rejected your value / the `name` seems off → route by the field's `ctx` (the visible label), never by `name`. No `ctx` and it still rejects → read the submit/validation error (it comes back in the result).
- a `fill` came back `ok:false` (value didn't read back) → wrong ref, or a `<select>` without that option, or a maxlength/number-constrained field.
- **a `stale_ref` error → that `@ref` is from a PREVIOUS screen, not the one you're on now.** `@eN` refs belong to the snapshot that produced them; after ANY transition (a checkpoint/"Next", a submit, a new stage, a navigation) the old refs are dead. The single fix: `browser_snapshot` the current screen and act on the FRESH refs (or use the refs from the last action's `changes`). Never carry refs across a transition — re-snapshot first, THEN `run_steps`.

## No DOM at all → fall back to the visual path (`desktop`)
`<canvas>`, games, maps, PDFs, custom-drawn UIs have no DOM — there the visual path is the ONLY one that works. Read the **desktop** part (`look` / `click_text` / objects). Also: to DRAG onto a non-interactive target (a drop-zone, a plain `<div>`) — labeled ones now appear in the snapshot as a `@ref`; for an unlabeled one you don't need pixels either, `browser_drag({from, to})` takes a selector for either side (`to:"[aria-label='drop-zone']"`). Fall to `look({zone:{kind:"selector", css}})` + `drag_path` only if the DnD isn't mouse-based.

## When a site BLOCKS automation (bot detection) → the desktop tier is the escape hatch
Some sites (Reddit, anything behind Cloudflare-style bot management) wall off the automated browser on load — a near-empty page, a "whoa there" / "you've been blocked" screen. The main tell is `navigator.webdriver=true`, which a CDP-driven Chrome sets; plus TLS/canvas fingerprint and IP reputation. This hits EVERY automation browser (Playwright included), not Verdesk specifically. Two responses, in order:
1. **Browser-tier stealth** (clears the first hurdle): Verdesk's Chrome already hides `navigator.webdriver` before the page's own JS runs — nothing for you to set. It's one signal of many, so not a guarantee against full bot-management.
2. **The real escape hatch — drive a NORMAL browser like a human, via the DESKTOP tier.** `set_view_target("window:0x…")` onto the user's own Chrome/Firefox window (NOT the CDP one) and operate it visually — `look` + `click_text` + learned objects + UIA. A visual desktop click is a real OS input on a real browser: no `navigator.webdriver`, no CDP, no automation flag — indistinguishable from a person at the keyboard, so webdriver-detection has nothing to catch. This is Verdesk's universal edge: when the *automated* browser is blocked, operate the *real* one the way a human would. (Verdesk is also an assistance tool — for users who can't use a mouse or read the screen — so operating a real browser on their behalf is a first-class use; how a user uses the tool is their own responsibility.)

## Thumbnails on/off

Thumbnails are ON by default. **If you are a model that cannot read images, turn them off** (header `X-Verdesk-Thumbnails: off`, or query param `thumbnails=off`, in your `.mcp.json`): each crop is otherwise unreadable noise in your context.

On a sign-in page take no `screenshot`: the snapshot's thumbnail of a form the browser autofilled can show the saved account, and only that setting turns it off.

With them off, disambiguation falls back to text only: `ctx=` (label, colour, viewport position) and the `(twins: …)` notice. If two controls are still indistinguishable after that, ask rather than guess.

**This is a trade-off, not a free cleanup — know what you're giving up:**
- **Safe**: an icon the DOM already names (`icon="save"`, inferred from `<title>`/class/etc.) never generated a thumbnail in the first place — turning thumbnails off costs you nothing there.
- **Not safe**: a genuinely mute icon (no accessible name, no DOM-inferred `icon=`) DOES lose information when thumbnails are off. You'll see `(N unnamed icons — image suppressed by your config)` instead of the crop — that notice means there is something real you can't see, because of your own setting, not because Verdesk hid it.
- **What to do about it**: the unnamed icons are still in the snapshot as bare `button [ref=eN]` entries near the notice. **If you can read images**, `screenshot({target:"@eN"})` shows one; if you cannot, turn thumbnails back on or ask rather than guess.
