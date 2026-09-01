# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Personal budget tracker for a Colombian freelancer: monthly income is entered in USD, in COP, or both (the two are summed), the USD part converted with the day's TRM, and spent against expenses recorded in COP. COP is the currency everything is normalized to. The whole app is **one self-contained file**, `index.html` — no build step, no dependencies, no tests, no package manager.

## Running

Open `index.html` directly in a browser (`start index.html` on Windows). Reload after editing. There is nothing to install, compile, or serve — but note the TRM fetch hits remote APIs, so a file:// page needs network access and the remote endpoints must allow CORS (both currently do).

## File layout inside index.html

| Lines (approx) | Contents |
| --- | --- |
| `<style>` ~7–408 | All CSS. Theme is driven by CSS custom properties on `:root`, overridden by `:root[data-theme="dark"]` and a `prefers-color-scheme: dark` media query guarded with `:root:not([data-theme="light"])`. Any new color must be defined in **all three** blocks, never only inside the media query. |
| `<body>` ~410–570 | Static markup for every card. Every dynamic value has a stable `id`; JS never generates layout markup, only rows and chips. |
| `<script>` ~572–1271 | A single IIFE in `'use strict'`, plain ES5 (`var`, `function`, no modules/classes/arrow functions). Match that style. |

## Architecture

**One state object, one render pass.** `state` holds `incomeUsd` and `incomeCop` (independent, additive - always read total income through `currentIncomeCop()`, never off one field), `trm` (+ `trmSource`/`trmDate`), `expenses[]` (`{id, label, value, detail}`, plus `{ibc, ibcAuto}` on a Planilla), and `tags[]`. Every mutation ends with a call to `renderAll()`, which recomputes tiles, meter, summary banner and the expense table from scratch. There is no diffing and no per-widget update path — **add new derived UI inside `renderAll()`**, don't wire a separate listener that writes to the DOM.

**No persistence.** State lives only in memory; there is no localStorage. The Export/Import JSON buttons are the only way to carry data across a reload, and the footer text tells the user so. `buildExportData()` emits `version: 3`; `applyImportedData()` validates each field defensively, re-assigns fresh `id`s from `nextId`, and tolerates older files by leaving absent fields at their defaults (a v1 file simply has no `incomeCop`). A v2 file has no `ibc`/`ibcAuto` either: `restorePlanillaExpense()` divides the stored value back out by the rates and only calls the result automatic when it still matches a suggested IBC for the imported income. If you change the state shape, update both functions and bump `version`.

**TRM fetching and the `trmLocked` flag.** `fetchTRM()` tries the official Superfinanciera dataset on `datos.gov.co` (Socrata, newest `vigenciadesde` first), and falls back to `open.er-api.com` on any failure. `trmLocked` is set when the user types a manual TRM or imports a file, and `setTrm()` bails out early when it is set — this exists so a slow in-flight fetch cannot silently overwrite a value the user just chose. Only the explicit "Actualizar" button clears it. Preserve this guard when touching TRM code.

**The `Planilla` tag is special-cased.** `PLANILLA_TAG` ('Planilla') is Colombian independent-contractor social security. Selecting it in the expense form hides the plain value input and shows the planilla panel, which computes IBC (via `defaultIbc()`, editable), Salud = IBC × `SALUD_RATE` and Pensión = IBC × `PENSION_RATE`; the expense's `value` becomes their sum and `detail` records the breakdown. `ibcOverridden` tracks whether the user hand-edited the IBC so `syncPlanillaIbcDefault()` stops overwriting it; it is reset every time Planilla is (re)selected and after each submit, so a second contract starts from the default again. Tags are otherwise free-form and user-editable, so never key behavior off any other label string.

**The IBC floor and automatic planillas.** `defaultIbc()` is income × 0.4 but never below `SALARIO_MINIMO` (the legal minimum IBC for an independent contractor); `clampIbc()` applies the same floor to a hand-typed value, on `change` rather than `input` so it doesn't fight someone mid-keystroke. A stored planilla keeps its own `ibc` and an `ibcAuto` flag: while automatic, `syncIncomeDependentExpenses()` — called at the top of `renderAll()`, so a refreshed TRM, a manual TRM or an income edit all reach it — re-derives `value` and `detail` through `applyPlanillaIbc()`. **At most one planilla may be automatic at a time**, enforced by `setSoleAutoPlanilla()` on submit, on ↺ and after an import: two rows cannot each be 40% of the same total income, and letting them try both doubles the IBC and overwrites a row the user had left at the minimum wage. The others keep the IBC they have as manual values. Editing the per-row IBC field built by `buildIbcEditor()` clears `ibcAuto` (the row then stays put across TRM changes) and the row's ↺ restores it. That editor re-renders on every keystroke, so `editingIbcId`/`pendingIbcFocus` hand the focus back after `renderExpenses()`.

## Conventions

- **UI language is Spanish (Colombia).** All user-facing strings, labels, and `aria-label`s are in Spanish; keep new ones in Spanish. Amounts go through the module-level `fmtCOP` / `fmtUSD` / `fmtDateLong` `Intl` formatters — don't hand-roll formatting.
- Build rows and chips with `createElement` + `textContent` and attach listeners directly; `innerHTML` is used only to clear a container (`= ''`). Keep user-supplied tag names out of any HTML string.
- No external assets — no CDN scripts, fonts, or images. Everything stays inline in the one file.
