# Notes

## Steps to reproduce

1. Install Neovim v0.12.x (or any version with built-in LSP support).
2. Configure pyright via `nvim-lspconfig` with a minimal setup (e.g. `vim.lsp.enable("pyright")`).
3. Open any Python file in Neovim.
4. Press several keys to trigger text edits.
5. Observe the notification/message area or a status plugin (e.g. `fidget.nvim`) that surfaces `lsp.progress` events.

## Observed

A brand-new `lsp.progress` (WorkDoneProgress) notification is created and immediately completed on **every single keystroke**. Neovim receives a `window/workDoneProgress/create` request followed by a `$/progress { kind: "begin" }` and then `$/progress { kind: "end" }` for every incremental analysis cycle. The screenshot in the issue shows entries #59 through #67 accumulating in rapid succession. Root cause: `createProgressReporter()` in `packages/pyright-internal/src/server.ts` called `this.connection.window.createWorkDoneProgress()` unconditionally on every `begin()` invocation, even when the previous analysis ended just milliseconds earlier.

## Expected

Rapid, back-to-back analysis cycles triggered by keystrokes should coalesce into a single `WorkDoneProgress` session. The progress indicator should open once when analysis begins and close only after a brief idle period (e.g. 500 ms) with no new analysis work queued. Clients like Neovim should therefore see at most one progress item per typing session rather than one per keystroke.
