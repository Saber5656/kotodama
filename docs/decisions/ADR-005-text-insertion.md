# ADR-005: Text insertion = paste-simulation primary, AX fallback, clipboard-only terminal mode; secure-input refusal

- Status: Accepted (2026-07-08)
- Informed by: [research/2026-07-macos-dictation-integration.md](../research/2026-07-macos-dictation-integration.md)

## Context

macOS has no sanctioned "insert text into another app" API. Practical options: pasteboard + synthetic ⌘V (used by effectively all dictation apps), AX `kAXSelectedTextAttribute` writes (inconsistent in Chromium/Electron), unicode keystroke synthesis (slow for long JA text, IME-entangled), clipboard-only. Password fields activate Secure Input, which synthetic input must respect.

## Decision

1. Strategy chain (auto mode): **preflight** (frontmost unchanged since hotkey release ∧ `IsSecureEventInputEnabled()==false` ∧ AX granted) → **paste-sim** (snapshot pasteboard → write with `org.nspasteboard.TransientType` → CGEvent ⌘V → restore guarded by `changeCount`) → **AX set** fallback → **clipboard-only** terminal fallback with HUD notice.
2. Explicit `clipboardOnly` mode exists for users who never grant Accessibility (persistent pasteboard write, no transient mark, no restore).
3. Hard refusals (never bypassed): secure input active; frontmost app changed. Refusal degrades to clipboard-only + notice, never silent drop (history row records status).
4. Pasteboard restore is best-effort over common types and is skipped if a third party wrote to the pasteboard mid-flight.
5. Unicode-typing insertion is a v2 experiment, not in v1.

## Consequences

- Requires Accessibility permission (onboarding step; app remains usable in clipboard-only mode without it).
- Brief pasteboard occupancy is visible to non-conforming clipboard managers — documented residual risk (DESIGN §14 T2).
- Per-app quirks (Electron, IME composition) are handled by a maintained manual test matrix rather than per-app code in v1.

## Alternatives considered

- **AX-first**: cleaner in theory, too many apps fail writes → demoted to fallback.
- **Keystroke synthesis first**: IME interactions and multi-second insert times for long text. Deferred.
- **IMKit input method (a real IME)**: architecturally purest but a completely different product shape (install/select an IME) with heavy UX cost. Rejected for this product.
