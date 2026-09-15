# Clipboard - Ecko Std Lib Package

Read and write the OS clipboard, so a terminal workflow can hand its answer
straight to whatever you paste into next.

## Install

```bash
ecko get github.com/ecko-lang/clipboard
```

```ecko
import clipboard
```

Needs Ecko 0.23.0 or later, and two capabilities: `exec` to run the clipboard
tool, and `env` to read `DISPLAY` and `WAYLAND_DISPLAY`, which is how the
session is identified.

## Usage

```ecko
import clipboard

clipboard.copy("hello")
clipboard.paste()            # "hello"

clipboard.available()        # false on a machine with no clipboard tool
clipboard.backend()          # "xclip", "wl-copy", "pbcopy", "clip", or null
```

The reason this package exists, rather than printing and letting you select:

```ecko
answer = ai "Summarise this error in one line: {log}"
clipboard.copy(answer)
```

## On Linux you need a tool installed

macOS and Windows ship theirs, so `copy` and `paste` always work. Linux ships
neither: the clipboard lives in the display server, and reaching it needs
`wl-copy` (from `wl-clipboard`) on Wayland, or `xclip` or `xsel` on X11.

`available()` answers `false` rather than raising, so a program can degrade
politely instead of wrapping every call in `try`:

```ecko
if clipboard.available() {
    clipboard.copy(answer)
} else {
    print(answer)
}
```

A `copy` or `paste` with nothing installed raises kind `clipboard` and names
what to install.

## Why it drives a tool instead of talking to the display server

On X11 the clipboard content lives in the **owning process**. Whoever set it
must stay alive to serve it, so a program that writes and exits leaves an empty
clipboard behind. `xclip` and `wl-copy` already fork a holder to solve exactly
this.

Linking a clipboard library instead would mean either keeping the interpreter
alive after your program ended, or shipping a daemon, and on Linux it would add
X11 and Wayland libraries to a binary that is otherwise self-contained. Driving
the tool that already solved the problem is the smaller answer.

It also keeps this a Layer 3 package rather than something in `ecko-std`: it is
expressible over `os.exec` and `std.proc`, which is the bar for staying out of
the core.

## Which backend gets chosen

The **session** decides, not what happens to be installed. A machine can carry
`wl-copy` with no Wayland compositor running, where it fails at the socket
rather than being absent, so choosing on `PATH` alone picks a tool that cannot
work.

| session | order tried |
|---|---|
| macOS | `pbcopy` |
| Windows | `clip`, and PowerShell `Get-Clipboard` to read |
| Wayland (`WAYLAND_DISPLAY` set) | `wl-copy`, then `xclip`, then `xsel` |
| X11 only (`DISPLAY` set) | `xclip`, then `xsel` |
| neither | none - `available()` is `false` |

Wayland is tried before X11 because a Wayland session usually also runs
XWayland. `xclip` appears to work there, and writes to a clipboard the
compositor's own applications never read.

## API

| call | what it does |
|---|---|
| `copy(text)` | Put `text` on the clipboard. Raises kind `clipboard` if no tool is installed. |
| `paste()` | The clipboard's contents, or `""` when empty. |
| `available()` | Whether copy and paste will work here. Never raises. |
| `backend()` | The tool this machine would use, or `null`. Useful in a bug report. |

## Testing

```bash
ecko test tests/
```

The tests cover backend selection, which is where this actually goes wrong. The
clipboard itself needs a display server, so it is exercised by `example.ecko`
rather than by the offline suite.

## License

MIT
