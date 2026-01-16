# sig?!

*[中文](./README-zh.md)*

`sig?!` (`sig-question-bang`, i.e., `sigqb`) is a small utility for "proxying" your Linux signals. It can intercept "capturable" signals (such as `SIGINT`) sent to corresponding programs and choose to either "swallow" these signals or forward them as other signals, while simultaneously invoking a one-line command to execute the functionality you want.

`sig?!` is developed in `Nim` and is open source on [GitHub](https://github.com/Water-Run/sigqb).

## Installation

Download the binary file for your platform from [Releases](https://github.com/Water-Run/sigqb/releases) and add it to your `PATH`:

```bash
cp -f ./sigqb ~/.local/bin/sigqb
chmod +x ~/.local/bin/sigqb
```

Execute:

```bash
sigqb
```

Get usage information:

```bash
sig?! by waterrun, version 0.1.0
https://github.com/Water-Run/sigqb

Usage:
    sigqb
    sigqb <config.ini> -- <command> [args...]
```

## Usage

The workflow of `sig?!` is as follows:

1. First, you need a configuration file that records the corresponding signals to be captured, mapped commands, and accompanying commands to execute (called `SIDEEFFECT`)
2. Launch the command you want to start with this configuration file, and `sig?!` will perform corresponding operations according to the configuration file

### Configuration File

The configuration file uses `.ini` format.

`[SECTION]` is the corresponding signal name to be captured, such as `[SIGINT]`, `[SIGHUP]`, etc., in all uppercase.

Note that the Section cannot be signals that cannot be captured like `[SIGKILL]`. In the configuration file, the same signal Section can only appear once.

Each signal Section has two keys: `MAPPING` and `SIDEEFFECT`. `MAPPING` is the mapped command (with the same naming convention as Section; when empty, the signal is "swallowed"; it can be signals that Section cannot capture like `SIGKILL`). `SIDEEFFECT` is the accompanying command to execute, and you can use `"""` to represent multi-line content. Both items are optional.

Example:

```ini
; sigqb config example (INI) — "example" fully covers all cases
; Convention:
; - [SECTION]: capturable signal name (all uppercase), such as [SIGINT] [SIGHUP] [SIGTERM] ...
; - Empty MAPPING: swallow the signal (do not forward to subprocess)
; - MAPPING=SIGXXX: map the signal corresponding to SECTION to SIGXXX and forward to subprocess
; - SIDEEFFECT: accompanying command to execute; can use """ for multiple lines
; - Missing key: indicates "this behavior is not configured" (determined by sigqb default strategy)

; A) Has MAPPING but no SIDEEFFECT: only performs mapping and forwarding, no accompanying command
[SIGHUP]
MAPPING=SIGTERM

; B) Has SIDEEFFECT but no MAPPING: only executes accompanying command, signal forwarding follows default (commonly forwards as-is)
;    This is used for "logging/cleanup when signal is received" but doesn't actively change forwarding rules
[SIGUSR1]
SIDEEFFECT=sh -c 'echo "[sigqb] SIGUSR1 received: hook only" >&2'

; C) MAPPING forwards as-is (explicitly writes same name): equivalent to "explicitly declaring no signal change", can combine with sideeffect
[SIGALRM]
MAPPING=SIGALRM
SIDEEFFECT=sh -c 'printf "%s [sigqb] SIGALRM forwarded as-is\n" "$(date -Is)" >> "${HOME}/.cache/sigqb.log"'

; D) Swallow signal (empty MAPPING) + has sideeffect: classic "swallow Ctrl+C but do something"
[SIGINT]
MAPPING=
SIDEEFFECT=echo "[sigqb] SIGINT swallowed (Ctrl+C ignored)" >&2

; E) Only swallow (empty MAPPING) and no sideeffect: pure swallow
[SIGUSR2]
MAPPING=

; F) Map to force termination (example): SIGQUIT -> SIGKILL, and leave trace (multiple lines)
[SIGQUIT]
MAPPING=SIGKILL
SIDEEFFECT="""
set -eu
log="${HOME}/.cache/sigqb.log"
mkdir -p "$(dirname "$log")"
printf "%s [sigqb] SIGQUIT -> SIGKILL (force)\n" "$(date -Is)" >> "$log"
"""

; G) Has MAPPING + has sideeffect (single line): SIGTERM -> SIGINT (graceful exit path)
[SIGTERM]
MAPPING=SIGINT
SIDEEFFECT=sh -c 'echo "[sigqb] SIGTERM mapped to SIGINT for graceful shutdown" >&2'

; H) Empty SECTION case (not recommended/usually should be treated as invalid):
;    In INI standard, "[]" should not be treated as a valid section name; shown here for documentation only.
;    If your implementation allows it, interpret it as a "default/fallback rule"; otherwise should error/ignore.
[]
MAPPING=SIGTERM
SIDEEFFECT=echo "[sigqb] default section hit (if supported)" >&2
```

### Using `sig?!`

After having a configuration file, you can use `sig?!` to proxy the command you want to execute:

Syntax:

```bash
sigqb <config.ini> -- <command to execute> [arguments for this command...]
```

For example, assuming the configuration file in the previous subsection's example is `test.ini`, and we want to execute `python main.py`:

```bash
sigqb test.ini -- python main.py
```
