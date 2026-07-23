# CMessageBox

A reusable owner-drawn **modal message box** for FreeBASIC / Win32, built on
[AfxNova](https://github.com/PaulSquires) and rendered through
[CBufferPaint](https://github.com/PaulSquires/CBufferPaint).

An owner-drawn caption band with a title and a close X, a body holding an optional icon and a
wrapped left-justified message, and a footer holding one to three
[CButtons](https://github.com/PaulSquires/CButton). It blocks, and it returns the id of whatever
dismissed it.

```
   +-- caption ------------------------------------+
   |  Warning                                 [X]  |
   +-- body ---------------------------------------+
   |   /!\   This buffer contains unsaved edits.   |
   |         Do you want to save it?               |
   +-- footer -------------------------------------+
   |          [ Save ] [ Don't Save ] [ Cancel ]   |
   +-----------------------------------------------+
```

---

## Files

| File | What it is |
|---|---|
| `CMessageBox.bi` | Types, colours, layout, public declarations |
| `CMessageBox.inc` | WndProc, the built-in painter, the modal loop, the API, the self-test |
| `CButton.bi` / `.inc` | Vendored, byte-identical from `PaulSquires/CButton` |
| `CBufferPaint.bi` / `.inc` | Vendored from `PaulSquires/CBufferPaint` — see the note below |
| `SegoeFluentIcons.ttf` | The glyph font the **demo** loads privately — the control has no opinion about which font its glyphs come from |
| `main.bas`, `frmMain.bi`, `frmMain.inc` | The demo harness |

Build (64-bit, from the repo directory):

```bash
fbc64.exe -i "C:\dev" -w all main.bas
```

> **The vendored `CBufferPaint` is one method ahead of its siblings.** This control needed
> `PaintTextEx` — `PaintText` forces `DT_VCENTER or DT_SINGLELINE` and so structurally cannot
> draw wrapped text. The addition is upstream in the canonical repo and is purely additive:
> `PaintText` is untouched and no existing call site changed. The other consumers will match it
> the next time they sync.

---

## Host obligations

Both come from `CBufferPaint`, not from this control:

1. **Bracket the message loop with `AfxGdipInit` / `AfxGdipShutdown`.**
2. **Never name an identifier `ok`.** GDI+'s `Status` enum defines `Ok = 0` in namespace
   `AfxNova`. The family convention is `bOK`.

### There is NO pump obligation

`CMessageBox_DoModal` runs **its own nested `GetMessage` loop** and calls `IsDialogMessage`
itself, so Tab, the focus ring and Space/Enter all work with **nothing added to your pump**.
Unlike `CComboBox`, `CNumericUpDown` and `CTextBox` there is no `CMessageBox_FilterMessage`.

That is the one big departure from the rest of this family, which is passive and pumped by its
host.

---

## Minimal use

```freebasic
dim as HWND hBox = CMessageBox_Create( hMain )      ' hMain is disabled until this returns

CMessageBox_SetFont( hBox, ghFont(GUIFONT_10) )            ' message + measuring + buttons
CMessageBox_SetCaptionFont( hBox, ghFont(GUIFONT_10) )
CMessageBox_SetGlyphFont( hBox, ghFont(SYMBOLFONT_20) )    ' the 32px message icon
CMessageBox_SetCloseGlyphFont( hBox, ghFont(SYMBOLFONT_10) ) ' the small close X

CMessageBox_SetCaption( hBox, "Warning" )
CMessageBox_SetText( hBox, "This buffer contains unsaved edits. Do you want to save it?" )
CMessageBox_SetIconKind( hBox, MBX_ICON_WARNING )
CMessageBox_AddPreset( hBox, MBX_BTN_SAVE_DONTSAVE_CANCEL )

select case CMessageBox_DoModal( hBox )     ' BLOCKS. The handle is dead afterwards.
case IDYES    : SaveIt()
case IDNO     : DiscardIt()
case IDCANCEL : ' also what Esc, the X and Alt+F4 answer
end select
```

Free-form buttons instead of a preset:

```freebasic
CMessageBox_AddButton( hBox, "Delete", IDYES )      ' up to three, left to right
CMessageBox_AddButton( hBox, "Cancel", IDCANCEL )
CMessageBox_SetDefaultButton( hBox, 0 )             ' accent border + Enter + initial focus
CMessageBox_SetFocusButton( hBox, 1 )               ' ...unless you call this AFTER it
```

And the one-liner, for the common case:

```freebasic
dim as long r = CMessageBox_Show( hMain, "Build succeeded.", "Information", _
                                  MBX_ICON_INFO, MBX_BTN_OK )
```

> `CMessageBox_Show` sets **no fonts**, so the icon comes out as a missing-glyph box. That is
> deliberate — the control never creates or owns a font. If you want icons, use the long form or
> wrap it yourself.

---

## The layout model

```
iconBlock   = hasIcon ? iconW + iconGap : 0
availW      = maxWidth - 2*bodyPadX - iconBlock             the WRAP width
textW/textH = DrawTextW( DT_CALCRECT or DT_WORDBREAK ) at availW
btnW(i)     = max( CButton ideal width , buttonMinWidth )
buttonH     = the TALLEST button's ideal height

idealW      = max( 2*bodyPadX + iconBlock + textW ,
                   2*footerPadX + sum(btnW) + (n-1)*buttonGap ,
                   captionPadX + captionTextW + closeWidth ,
                   minWidth )
idealH      = captionHeight + (2*bodyPadY + max(textH, iconH)) + (2*footerPadY + buttonH)
```

Five things worth knowing before you use it:

**`SetMaxWidth` constrains the TEXT, not the window.** Three long-captioned buttons legitimately
make the box wider than the wrap width. It is a `max()`, not a clamp.

**The box shrinks to fit.** `DT_CALCRECT` reports the width the text actually *used*, so
"Delete this file?" gives a small box rather than one padded out to the maximum — which is what
a real `MessageBox` does.

**The icon and the text share a vertical centre.** Each is centred on the taller of the two, so
a one-line message sits level with a 32px icon instead of clinging to its top edge.

**The buttons share one height but not one width.** A row must sit on one baseline or it reads
as broken; widths come from each caption above a minimum, exactly as in the reference
screenshot (Save / Don't Save / Cancel are three different widths).

**Absent parts give EMPTY rects**, not zero-width ones tucked against an edge — so a paint
callback can test `IsRectEmpty` instead of re-deriving presence from the strings.

---

## The dismissal contract

| Route | Answers |
|---|---|
| A button click | that button's id |
| Space / Enter on a focused button | that button's id |
| Enter with focus elsewhere | the **default** button's id |
| **Esc**, the **X**, **Alt+F4**, any `WM_CLOSE` | `SetCancelID`, or — unset — the **LAST button's id** |

The last row is the one to think about. "The last button" is Cancel in every preset and in the
reference screenshot, but it is a *convention*, not a law: **if your last button is destructive,
call `CMessageBox_SetCancelID` yourself.** The presets all set it explicitly, including
`MBX_BTN_YESNO`, which has no Cancel button and points Esc at `IDNO`, and `MBX_BTN_OK`, where the
only button is also the cancel path so that Esc is not a silent no-op.

---

## Colours

`MBX_COLORS` is 17 flat `COLORREF` fields: the window border, the caption band and its title,
three close-button pairs, the body and its message, the footer and its divider hairline, and one
colour per icon kind.

**The defaults are DARK**, inherited from `CBUTTON_COLORS`' palette rather than matched to the
light reference screenshot — a box built with no colour calls looks like the rest of this family.
The demo's `ApplyLightTheme` is the light set, and is the only place the screenshot's exact
palette lives.

- **The close X is invisible until you touch it** — its idle background defaults *equal to* the
  caption's. Its hot pair is the Windows close-red rather than the theme accent.
- **There is no disabled mood anywhere in the struct.** A message box is modal and everything on
  it is live by definition; a disabled button on a box you cannot leave would be a trap.
- Buttons are themed separately with `CMessageBox_SetButtonColors`, which applies to every button
  that exists now *and* every one added later, so call order does not matter.

---

## What it deliberately does not do

| | |
|---|---|
| **No `HICON` path** | Icons are glyph strings drawn with a font, so they follow your theme. You cannot get the multi-colour system triangle. |
| **No Ctrl+C** | The real Win32 `MessageBox` copies its text to the clipboard. This one draws with `DrawTextW`; there is no text object to select from. |
| **No tooltips** | By request. No tooltip window, no callback, no hover text anywhere. |
| **No text selection or scrolling** | A message far taller than the screen will be clipped. `CTextBox` was considered and rejected: it would add a mandatory pump call and a fourth tabstop competing with the buttons. |
| **Not resizable** | Movable by its caption; no sizing border. |
| **More than 3 buttons** | `AddButton` returns 0 past three. A fourth choice wants a real dialog. |
| **No rounded corners** | Square, with a 1px border. Win11's DWM rounding needs `DwmSetWindowAttribute`, and AfxNova's `AfxSetWindowCornerPreference` has no `#inclib`, so a control library that used it would force a `dwmapi` link dependency on every host. Add it in your own host if you want it. |
| **`CS_DBLCLKS` is off** | The only double-clickable thing is the X, and a rapid second click there is a legitimate second click on a window that may already be gone. |
| **No `WM_SETTEXT` alias** | Unlike `CButton`, the caption is not the window text — a message box is not something generic dialog code walks looking for captions. |

---

## Two things that surprise people

**`DoModal` destroys the box.** The handle is dead when it returns. A box you create and then
abandon *without* calling `DoModal` is yours to `DestroyWindow`.

**Two glyph fonts, not one.** `SetGlyphFont` is the ~32px message icon; `SetCloseGlyphFont` is
the ~10px close X. One font cannot serve both, and since nothing in this family ever *creates* a
font, the control cannot derive a smaller one itself. Leave the second unset and the X is drawn
at icon size.

---

## Writing your own painter

`CMessageBox_SetPaintCallback` replaces the built-in painter wholesale. The control has already
filled the client with `BackColor`, and every rect in `MBX_PAINTINFO` is precomputed — never
re-derive one from another. **The buttons are real child windows and paint themselves**; your
callback draws the box around them.

> **Do not reach for `PaintBorderRect` to draw the window outline or the footer divider.** It
> *fills* unconditionally before it strokes, so used as an outline it erases everything beneath.
> That has shipped **three times** in this family (CToggle, CComboBox, CNumericUpDown), every
> time from a copied callback. `PaintRoundOutline` and `PaintLine` stroke without filling.

Also: draw the message with the **same font and the same `DT_WORDBREAK`** the control measured
with, through `PaintTextEx` — `PaintText` would force `DT_SINGLELINE` and give you one clipped
line inside a tall box.

**Assert it rather than eyeballing it.** `CMessageBox_CountRenderedTones( hBox, MBX_PART_* )`
renders the box offscreen with your painter and counts distinct colours in one of its rects. The
demo does exactly this to its own callback at startup, which is the pattern to copy.

> **Do not copy `CButton`'s `> 1` threshold.** It works there because a wiped button really is
> one flat colour. It does **not** work here: this control's bands span the full width, so the
> window border strokes through every one of them and a *fully flooded* box still measures **2**
> tones per band. Measured, by deliberately breaking the border and rebuilding:
>
> | | caption | body | footer | close |
> |---|---|---|---|---|
> | healthy | 50 | 64 | 3 | 3 |
> | flooded | 2 | 2 | 2 | 2 |
>
> Calibrate your floors between those rows. One further limit: flooding a single band *in its
> own colour* is invisible to this probe — it catches floods that cross bands.

---

## Self-test

```bash
CMESSAGEBOX_SELFTEST=1 main.exe
```

**75 assertions**, none of which enter the modal loop (a self-test that went modal would block
the host that called it). `CMessageBox_LayoutForTest` is the way in: it sizes the box to its
ideal size without showing it.

Covered: the pure `ResolveGlyph` / `ResolveIconColor` truth tables; the three bands tiling the
client exactly; close-button pinning and the title never running under it; the icon cell, the
text wrap box, and their shared vertical centre; button right-alignment, exact gaps, the
min-width floor, one shared baseline; auto-height (a wrapped message is measurably taller);
shrink-to-fit; the footer widening the box past the wrap width; the icon block giving back cell
*and* gap — with its no-re-wrap precondition asserted rather than assumed; cancel-id resolution
for one and three buttons; the default/focus precedence in both directions; and the offscreen
tone probe with calibrated floors.

Plus **tab navigability, end to end**: that the box **has** `WS_EX_CONTROLPARENT` and is not
itself a tabstop, that every child button **has** `WS_TABSTOP` and does **not** have
`WS_EX_CONTROLPARENT`, and that `GetNextDlgTabItem` actually reaches all three buttons and
cycles.

> That flag is the **opposite** of the `CButton` fix. `CWindow.Create` defaults `dwExStyle` to
> `WS_EX_CONTROLPARENT OR WS_EX_WINDOWEDGE`; for a *leaf* control that is fatal, and CButton
> shipped broken with it. This window genuinely **is** a container — its tabstops are its
> children — so the dialog manager must descend into it. The test is *"does this window have
> children the dialog manager should find?"*, not *"is this a control?"*. Full write-up in
> `C:\dev\Learnings.md`.

---

## The control family

`CMessageBox` is the seventeenth of these. The shape is shared: one real `HWND`, all state in a
`TYPE` in the `CWindow` UserData area, one `WndProc`, host callbacks for paint and messages, one
`CBufferPaint` per `WM_PAINT`, rects derived and never set, lazy layout.

Two firsts, both forced by what a message box is: **it runs its own nested message loop**, and
**it is a top-level window that hosts other siblings** as tab-navigable children.

Seeded from **CButton** (its focus, capture and mood machinery) with **CPopupMenu**'s `WS_POPUP`
window contract and monitor-clamping arithmetic.

Callback prefix is `MBX_`. The family shares one typedef namespace, so
`SB_ TB_ TXT_ HDR_ MB_ PM_ SPL_ IP_ SEL_ TOG_ SCP_ CBO_ NUD_ BTN_` are already taken.
