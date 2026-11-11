# clipboard

Read and write the OS clipboard, so a terminal workflow can hand its answer
straight to whatever you paste into next. Needs `exec` and `env`; grant them
when you import the package.

## Install

```bash
ecko get github.com/ecko-lang/clipboard
```

`ecko get` writes a dependency with no grant. Give the entry both
capabilities in your `ecko.json`: `exec` to run the clipboard tool, and `env`
to read `DISPLAY` and `WAYLAND_DISPLAY`, which is how the session is
identified.

```json
"dependencies": {
  "clipboard": { "path": "github.com/ecko-lang/clipboard", "version": "0.38.0", "grant": ["exec", "env"] }
}
```

```ecko norun
import clipboard   # granted exec and env in ecko.json
```

## Usage

```ecko norun
import clipboard   # granted exec and env in ecko.json

print(clipboard.available())  # true  (false where there is no clipboard)
print(clipboard.backend())  # xclip  ("wl-copy", "xsel", "pbcopy", "clip", or null)

clipboard.copy("hello")
print(clipboard.paste())  # hello

# The reason this package exists: hand an answer straight to the clipboard,
# and degrade politely where there is none.
answer = ai "Summarise this error in one line: connection refused"
if clipboard.available() {
    clipboard.copy(answer)
    print("copied")
} else {
    print(answer)
}
```

These blocks are marked `norun`: they need a display and they would overwrite
your clipboard.

## API

| function | returns |
|---|---|
| `copy(text)` | `null`, having put the String `text` on the clipboard |
| `paste()` | the clipboard's contents as a String, or `""` when it is empty |
| `available()` | whether copy and paste will work here, as a Bool; never raises |
| `backend()` | the tool this machine would use, or `null` |

## Notes

### Errors

| kind | reason | when |
|---|---|---|
| `clipboard` | `no_display` | a Linux (or BSD) session has neither `WAYLAND_DISPLAY` nor `DISPLAY` set, or they are empty |
| `clipboard` | `no_tool` | there is a display but no clipboard tool is installed; the message names what to install |
| `clipboard` | `tool_failed` | the tool exits non-zero for a reason other than an empty clipboard; the message carries its stderr or exit code |
| `clipboard` | `timeout` | the tool does not finish within 5 seconds; it is killed |
| `bug` | `bad_argument` | `copy` is given something that is not a String, a secret included |

`copy(secret(...))` is refused rather than copying the text `[secret]`; copy a
secret with `clipboard.copy(reveal(s))`, deliberately.

### On Linux you need a tool installed

macOS and Windows ship theirs. Linux ships neither: the clipboard lives in the
display server, and reaching it needs `wl-copy` (from `wl-clipboard`) on
Wayland, or `xclip` or `xsel` on X11.

### Which backend gets chosen

The **session** decides, not what happens to be installed. A machine can carry
`wl-copy` with no Wayland compositor running, where it fails at the socket
rather than being absent, so choosing on `PATH` alone picks a tool that cannot
work.

| session | order tried |
|---|---|
| macOS | `pbcopy` (and `pbpaste` to read) |
| Windows | `clip`, and PowerShell `Get-Clipboard` to read |
| Wayland (`WAYLAND_DISPLAY` set) | `wl-copy`, then `xclip`, then `xsel` |
| X11 only (`DISPLAY` set) | `xclip`, then `xsel` |
| neither | none - `available()` is `false` |

Wayland is tried before X11 because a Wayland session usually also runs
XWayland. `xclip` appears to work there, and writes to a clipboard the
compositor's own applications never read.

An empty clipboard reads as `""`: xclip's "target STRING not available" and
wl-paste's "Nothing is copied" are recognised as emptiness. Any other failure
raises. On Windows the line break PowerShell adds after the text is removed.

### Why it drives a tool instead of talking to the display server

On X11 the clipboard content lives in the **owning process**. Whoever set it
must stay alive to serve it, so a program that writes and exits leaves an empty
clipboard behind. `xclip` and `wl-copy` already fork a holder to solve exactly
this. Linking a clipboard library instead would mean keeping the interpreter
alive after your program ended, or shipping a daemon, and on Linux it would add
X11 and Wayland libraries to a binary that is otherwise self-contained.

## Testing

```bash
ecko test
```

The tests never touch your clipboard: they put a fake `xclip` alone on `PATH`
and drive `copy` and `paste` against it, including its failures and a tool
that hangs (that case waits out the 5-second deadline twice).

## License

[MIT](LICENSE)
