# Changelog

## 0.38.0

### Fixed

- A clipboard tool that hangs is killed after 5 seconds and the call fails
  with a `timeout` error. Before, `copy` reported an empty "failed writing the
  clipboard:" and `paste` waited forever.
- `paste` no longer returns `""` when the tool fails with nothing on stderr,
  or with any message containing "target"; only the tools' own
  empty-clipboard messages read as empty. Other failures raise and carry the
  tool's stderr or exit code.
- A Linux session with no display server is reported as such (`no_display`),
  instead of telling you to install a tool you already have.
- An empty `DISPLAY` or `WAYLAND_DISPLAY` no longer counts as a display.
- On Windows, the line break PowerShell adds after the clipboard text is
  removed from what `paste` returns.

### Added

- Every `clipboard` error carries a `reason` you can match: `no_display`,
  `no_tool`, `tool_failed`, `timeout`.

### Breaking

- `copy` accepts only a String. `null` used to copy the text "null", a secret
  copied "[secret]", and bytes failed with an unrelated encoding error; all
  now raise a `bug` error naming `clipboard.copy`.
  Migration: convert first (`clipboard.copy(string(n))`), and copy a secret
  deliberately with `clipboard.copy(reveal(s))`.
- Error messages are reworded and start with the function name.
  Migration: match on `kind` and `reason` instead of the message text.
