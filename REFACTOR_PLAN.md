# CMessageBox — design record

Why the control is shaped the way it is, and what the build turned up. The `README.md` is how to
use it; this is why it is like that, and what not to re-litigate.

---

## The interview (2026-07-23)

Eight decisions were settled with the author before any code was written. All eight went to the
recommended option.

| | Decision | The alternative that was rejected |
|---|---|---|
| **API shape** | Blocking `DoModal` returning the clicked id, with its own nested pump | Modeless + result callback. Rejected: the host would have to restructure any code wanting an inline answer, and it stops being a "message box". |
| **Title bar** | Owner-drawn `WS_POPUP`, no system caption | A real `WS_CAPTION`. Rejected: the OS would draw it, so the caption band would ignore the colour struct entirely and a dark box would get a light system caption. |
| **Message text** | `DrawTextW` with `DT_WORDBREAK`; `DT_CALCRECT` gives the height | A read-only multiline `CTextBox`. Rejected: vendors two more files, adds a mandatory `CTextBox_FilterMessage` inside our pump, and the RichEdit is focusable so it would compete with the buttons for Tab. The cost of the choice is no selection and no Ctrl+C. |
| **Icon** | Glyph + a per-kind `COLORREF` | An `HICON` path. Rejected: the system icons never follow a theme, which is the whole reason this family exists. |
| **Buttons** | Free-form `AddButton` ×1–3, plus presets that call it | Win32-style flags only. Rejected: you could never write "Don't Save" or localise. |
| **Cancel path** | Explicit `SetCancelID`, defaulting to the last button | A fixed sentinel distinguishing "dismissed" from "clicked Cancel". Rejected as an extra concept for a distinction hosts rarely act on. |
| **Default vs focus** | Two setters, focus follows the default unless set after | One setter. Rejected: you could never focus Cancel while Delete stays default. |
| **Sizing** | Auto-size against `SetMaxWidth`, centred on the parent | User-resizable. Rejected: real message boxes are not, and it adds a whole re-wrap geometry path. |

Decided without asking, all reversible: `MessageBeep` fires for the kind (settable off); buttons
are sized to their own captions above a floor rather than equalised; `DoModal` destroys the
window; nested boxes are allowed.

---

## Decisions worth not re-litigating

**1. The nested message loop, and why the pointer is re-fetched every iteration.**
The loop condition is not `pBox->bRunning` on a cached pointer. A host that destroys the *parent*
takes this owned window with it and frees the state, so a cached pointer would be a
use-after-free on the very next test. The loop checks `IsWindow` first, then re-fetches. The
result is read while the window is still alive, before the destroy.

**2. `WM_QUIT` is re-posted, never consumed.** `GetMessage` returning 0 removes the quit from the
queue. AfxNova's own `AfxInputBox` ends its nested loop exactly that way, which is safe only
because nothing runs after it; here, swallowing the host's shutdown would leave the outer pump
spinning forever.

**3. The parent is re-enabled BEFORE the box is destroyed.** Destroying an owned window while its
owner is still disabled hands activation to another application — the classic "my app went behind
Explorer when the dialog closed" bug. `hParent` is captured at the top of `DoModal` precisely
because the state may not survive the loop.

**4. `WS_EX_CONTROLPARENT` is CORRECT here, and that is the opposite of the CButton fix.**
`CWindow.Create` defaults `dwExStyle` to `WS_EX_CONTROLPARENT OR WS_EX_WINDOWEDGE`. For a leaf
control that flag is fatal — CButton, CToggle and CComboBox all shipped unreachable by Tab
because of it. This window genuinely **is** a container: its tabstops are its child buttons, so
the dialog manager must descend into it. It is passed explicitly (without `WS_EX_WINDOWEDGE`,
which means nothing to a window drawing its own chrome) and asserted **both ways** in the
self-test — the box has it, the buttons do not.

**5. The close X is not a `CButton`.** It would be a fourth tabstop on a box whose tab order is
supposed to be exactly the choices. It is a hit-tested rect with its own three colour pairs, and
it takes mouse capture for the same press/cancel gesture CButton has.

**6. `WM_NCHITTEST` returns `HTCAPTION` over the caption band but `HTCLIENT` over the X.**
`HTCAPTION` is the entire drag implementation — `DefWindowProc` does the move. Excluding the
close rect is not optional: without it the X is both undraggable and unclickable, which is the
natural mistake in this area.

**7. Enter is intercepted only when a button does NOT have focus.** A focused `CButton` claims
Enter itself through `WM_GETDLGCODE`, and clicking *that* button is the right answer. Stealing
Enter unconditionally in the pump would fire the default button no matter which one the user had
tabbed to.

**8. One dismissal path.** A button click, Esc, the X, Alt+F4 and a host `WM_CLOSE` all funnel
through `CMessageBox_Dismiss`, so no route can end the loop without also setting a result. The
message callback's answer is **ignored** for `WM_CLOSE`: a modal box that cannot be dismissed is
a hang with the parent disabled, not a refused action.

**9. Two glyph fonts.** The message icon sits in a ~32px cell; the close X is ~10px caption
chrome. One font cannot serve both, and this family never *creates* fonts, so the control cannot
derive a smaller one. `SetCloseGlyphFont` falls back to the icon font, which is right only if the
host sized that font for the X.

**10. No rounded corners.** Win11 rounding needs `DwmSetWindowAttribute`; AfxNova's
`AfxSetWindowCornerPreference` carries no `#inclib`, so using it would force a `dwmapi` link
dependency on every host of this control. A host that wants it can add the one call itself.

---

## What the build turned up

**It built clean first time, `-w all`, zero warnings, and passed 74 of 75 assertions on the first
run.** That is worth stating plainly, because the interesting findings below were all found by
things *other* than the build.

### 1. `CBufferPaint.PaintText` cannot draw wrapped text — found by reading, not by failing

`PaintText` unconditionally ORs in `DT_NOPREFIX or DT_VCENTER or DT_SINGLELINE`. `DT_SINGLELINE`
and `DT_WORDBREAK` cannot both hold, so the message body — the whole point of this control — was
structurally undrawable through the family's renderer. `EnsureGdiReady` is `private`, so
reaching for `getMemDC()` and calling `DrawTextW` directly was not available either: GDI+ batches,
and an unflushed GDI text call intermittently loses the shapes underneath it.

Fixed **upstream and additively**: `CBufferPaint.PaintTextEx` is `PaintText` with only
`DT_NOPREFIX` forced, leaving the layout flags to the caller. `PaintText` is untouched and no
existing call site changed behaviour. The canonical repo and this vendored copy are in step; the
other eleven consumers will match at their next sync.

Had this been noticed one layer later, the obvious workaround would have been to call
`PaintText("")` purely to trigger the flush and then draw by hand — which would have worked, and
would have been the wrong thing to leave in twelve repos.

### 2. The tone probe's `> 1` threshold does not generalise — and it took two attempts to prove it

The probe was copied from CButton, where "a wiped control is one flat colour" makes `> 1` a
sound assertion. The A/B is mandatory in this family precisely because a probe that has never
been made to fail has not been tested. Both attempts are worth recording:

- **First attempt: swap the footer divider to `PaintBorderRect`.** The counts did not move at
  all. The band had just been filled with `FooterBackColor`, so flooding it *with that same
  colour* is invisible. The experiment was badly chosen, not the probe broken — but it is a real
  limit of the technique, and it is now written into both the self-test and the README: **this
  probe catches floods that cross bands, not a band repainted in its own colour.**
- **Second attempt: swap the window border to `PaintBorderRect`.** The render was genuinely
  destroyed — 50/64/3/3 collapsed to 2/2/2/2 — **and every assertion still passed.** This
  control's bands span the full width, so the border strokes through all of them and a totally
  flooded box still reads 2 tones per band.

The floors are now calibrated between the measured healthy and flooded rows (`>= 10` for the
caption and body, `>= 3` for the footer and the close button) and were confirmed to go red on the
flooded build and green on the healthy one. The demo's own host-callback probe was calibrated the
same way and A/B'd the same way.

**The transferable lesson: a threshold copied from a sibling is an assumption about that
sibling's geometry.** `> 1` was not wrong in CButton; it was wrong here, and it was wrong
*silently*, on a build whose output was visibly destroyed.

### 3. The one failing assertion was a scaffolding bug, and it failed honestly

"Removing the icon gives back the cell AND the gap" reported `with icon 128, without 128`. The
control was right: at that size the box's width is the **footer's** (`2*footerPadX` plus one
button's min-width), so the 48px icon block never reached the total and the assertion was
measuring the `max()` rather than the icon.

Fixed by giving the test a message long enough that the text block decides the width — and by
adding a **precondition assertion** that the message does not re-wrap between the two cases,
since removing the icon widens the wrap box. Without that, a future edit to the default
`nMaxWidth` could make the delta silently stop being the icon block.

Same class as the CButton demo's alignment rows, which were sized to their own ideal width and so
demonstrated nothing: **an assertion that is not isolated measures the wrong quantity while
looking entirely reasonable.**

---

## Verification status

**Verified:**
- Builds clean, `fbc64 -w all`, zero warnings.
- `CMESSAGEBOX_SELFTEST=1` — **75/75**, plus the demo's host-painter probe.
- The tone probe **A/B'd in both directions**, twice (control painter and demo painter).
- Tab navigability asserted end to end through `GetNextDlgTabItem`, not inferred from style bits.

**NOT verified — the interactive pass is the author's, and nothing below has been exercised:**
- **The modal loop has never run.** Every assertion deliberately avoids it, so `DoModal`,
  the parent disable/restore, activation handling and the `PostQuitMessage` re-post path are
  **build- and reasoning-verified only**. This is the highest-risk item in the repo.
- Dragging by the caption; the X's hover, press and press-cancel; Esc / Alt+F4 answering the
  cancel id; Enter firing the default button from each focus position; Tab and Shift+Tab in a
  live box.
- The parent being genuinely dead while the box is up, and alive after.
- **Nested boxes** (demo row 9) — two modal loops on the stack at once.
- Closing the host while a box is open.
- Pixel appearance of anything, including whether the light theme in demo row 4 actually matches
  the reference screenshot. The control's own defaults are dark and have never been seen.
- Nothing has run above 100% DPI, and no multi-monitor or negative-coordinate placement has been
  tested — `CMessageBox_ComputeOrigin` clamps for it, and that clamp is unexercised.
- **No host uses this control.**
