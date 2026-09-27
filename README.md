# Ghi config

Explicit typed environment loading for Ghi. Package identity: `arm092/config`.
Source namespace: `arm092.config`. MIT licensed; Go standard library only.

## Install

```sh
mojave add arm092/config https://github.com/arm092/ghi-config.git v0.1.0
```

Verified with the published Windows amd64 Ghi v0.2.6 and Mojave v0.1.0.
Other compiler versions and platforms have not been verified.

## Use

```ghi
namespace main

import arm092.config.Env
import arm092.config.EnvError
import time "go:time"

func main() {
    env := new Env()
    try {
        address := env.String("JOURNAL_ADDR", "127.0.0.1:8080")
        database := env.RequiredString("JOURNAL_DB")
        pageSize := env.Int("JOURNAL_PAGE_SIZE", 20)
        debug := env.Bool("JOURNAL_DEBUG", false)
        timeout := env.Duration("JOURNAL_TIMEOUT", 5 * time.Second)
        // Pass these values to your application; do not log database secrets.
        println(address, pageSize, debug, timeout)
        if database == "" { panic("unreachable") }
    } catch err EnvError {
        // Name(), Code() and Error() never include the environment value.
        println(err.Name(), err.Code(), err.Error())
    }
}
```

The application owns its settings struct and validation, such as positive
timeouts, valid addresses, supported database schemes, and allowed page sizes.
This package loads values; it does not impose application constraints.

## API

Create `new Env()`. Each call reads the current process environment using
`os.LookupEnv`; values are not cached or copied into a snapshot.

| Method | Return type | Missing variable |
| --- | --- | --- |
| `String(name, fallback)` | `string` | Returns the typed fallback |
| `Int(name, fallback)` | `int` | Returns the typed fallback |
| `Bool(name, fallback)` | `bool` | Returns the typed fallback |
| `Duration(name, fallback)` | `time.Duration` | Returns the typed fallback |
| `RequiredString(name)` | `string` | Throws `EnvError` with code `missing` |
| `RequiredInt(name)` | `int` | Throws `EnvError` with code `missing` |
| `RequiredBool(name)` | `bool` | Throws `EnvError` with code `missing` |
| `RequiredDuration(name)` | `time.Duration` | Throws `EnvError` with code `missing` |

Fallbacks apply **only when the variable is absent**. A present but malformed
value fails instead of silently using a fallback. `String` preserves an
explicit empty string. The required methods reject empty or Unicode-whitespace-only
values with code `empty`. All nonblank values are preserved without trimming.
Defaults are already typed and are returned unchanged.

- Strings are returned verbatim, including leading/trailing whitespace.
- Integers use decimal `strconv.Atoi`: optional sign and leading zeros are
  accepted. Hexadecimal, fractions, whitespace and native `int` overflow fail.
- Booleans use `strconv.ParseBool`: true spellings are `1`, `t`, `T`, `true`,
  `TRUE`, `True`; false spellings are `0`, `f`, `F`, `false`, `FALSE`, `False`.
  `yes`, `no` and arbitrary mixed case are not accepted.
- Durations use `time.ParseDuration`: examples are `250ms`, `1m30.5s` and `2h`.
  Zero and negative durations are accepted. A unit is required except for `0`.
  Overflow beyond signed 64-bit nanoseconds fails. Import `time "go:time"`
  to provide a typed fallback and use the returned value with Go time APIs.

No `.env` file is read. Populate the process environment through your shell,
container, or deployment system before loading settings. There is no reflection,
struct tag mapping, implicit conversion, global registry, or external dependency.
On Windows, variable names follow the operating system's case-insensitive rules.

## Errors

`EnvError` extends `Exception`. `Name()` returns the variable name, `Code()`
returns a stable string code, and inherited `Error()` returns the name and a
fixed description. Parsing errors are replaced; their original messages and
causes containing input values are not retained.

| Code | Meaning |
| --- | --- |
| `invalid_name` | Name is empty or contains `=` or NUL |
| `missing` | Required variable is absent |
| `empty` | Required variable is empty or Unicode-whitespace-only |
| `invalid_int` | Present value is not a valid native decimal integer |
| `invalid_bool` | Present value is not an accepted boolean spelling |
| `invalid_duration` | Present value is not a valid duration or overflows |

Each method fails immediately by throwing; errors are not aggregated. Keep
variable names as trusted configuration keys. Error names remain visible,
while environment values and fallbacks are never included by this loader.

## Tests

```sh
ghi fmt --check .
ghi test -v .
```

Tests live under `tests/` and use isolated process-environment keys with cleanup.
They cover missing/empty values, required fields, defaults, untouched strings,
typed parsing, overflow, boolean false and numeric zero, fresh reads, and error
redaction. Tests that mutate the process environment must not run in parallel.
