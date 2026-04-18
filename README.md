# sky-uptime

A real-time uptime dashboard built in [Sky](https://github.com/anzellai/sky) to
demonstrate end-to-end typed error handling: every network failure, HTTP error,
and database write error flows as a first-class `Sky.Core.Error` value from the
kernel all the way to a banner in the browser. Nothing is swallowed.

Companion source for **EP06** of the `Anzel | Sky Lang` YouTube channel:
_"I Built a Dashboard Where Every Error Reaches the UI"_.

## What it does

- Register HTTP endpoints to monitor (name + URL).
- Every 30 seconds (SSE subscription) the server re-runs every health check.
- Classifies each result by a real `CheckStatus` ADT -- `Healthy`, `Degraded`,
  `Down` -- and records it in SQLite with the typed `ErrorKind` alongside the
  message.
- The view surfaces errors in two places:
  - A notification banner at the top of the page (info / ok / error).
  - Per-endpoint cards with a badge showing the `ErrorKind` label
    (`Network:`, `HttpStatus:`, `Timeout:` etc.) and the original message.

## Why it exists

Every language I've shipped in lets you make the wrong decision easily:

- Python `try: ... except: pass`
- Go `if err != nil { return nil, err }` (the original context disappears)
- TypeScript `await fetch(url)` with no `.catch()` in sight

In Sky 0.9+, `Sky.Core.Error` is a tagged union with eleven variants shared by
the whole standard library and every generated Go FFI wrapper. You cannot use
the return value of a fallible function without pattern-matching on it, and you
cannot pattern-match without the compiler checking you've covered every
constructor. Missing a case is a build error, not a runtime surprise.

`sky-uptime` is a concrete end-to-end example of what that looks like in a
real-world app -- three files, about 700 lines, including CSS.

## Files

```
sky-uptime/
  sky.toml            -- [live] port + [database] sqlite config
  src/
    Main.sky          -- Sky.Live TEA app; model, update, view, notifications
    Lib/
      Db.sky          -- every query returns Result Error; CAST id AS TEXT
                         for consistent string-typed identifiers
      Monitor.sky     -- the HTTP probe: Task Error Response -> CheckResult
                         with a classifyError that pattern-matches on ErrorKind
```

## Build and run

Requires [Sky](https://github.com/anzellai/sky) v0.9+ on your path.

```bash
git clone https://github.com/anzellai/sky-uptime.git
cd sky-uptime
sky build src/Main.sky
./sky-out/app
```

Open <http://localhost:3000>.

Add a few endpoints. Try one that resolves, one that 500s, and one with a
made-up host name. Click **Check All Now**. Watch the error kinds surface.

## The key piece of the puzzle

The whole ADT propagation story in one function (from `src/Lib/Monitor.sky`):

```elm
checkEndpoint : String -> CheckResult
checkEndpoint url =
    case Task.run (Http.get url) of
        Ok resp ->
            classifyResponse resp

        Err err ->
            classifyError err


classifyError : Error -> CheckResult
classifyError err =
    case err of
        Error kind info ->
            { status = statusForKind kind
            , code = 0
            , errorKind = Error.kindLabel kind
            , errorMessage = info.message
            }
```

`statusForKind` exhaustively matches on every `ErrorKind` variant. Add a new
variant to the stdlib and this function stops compiling until you handle it.

## License

MIT. Build whatever you want with it.
