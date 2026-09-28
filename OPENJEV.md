# OpenJEV support in Jevlin

This fork adds optional [OpenJEV](https://openjev.sh) support alongside the
original TypeSafe integration. OpenJEV is a free community gateway to the same
Jev model built by [TypeSafe](https://typesafe.ai). TypeSafe remains the default;
anyone with a TypeSafe key sees zero behaviour change.

## What was added

- `src/http.zig` — new `endpoint` field on `Http` (defaults to the TypeSafe
  endpoint), `Provider` enum, `endpointFor` helper, and `typesafe_endpoint` /
  `openjev_endpoint` / `typesafe_model` / `openjev_model` constants. The
  `exchange` callback now uses `self.endpoint` instead of a hardcoded TypeSafe
  URL, so the transport can route to either provider.
- `examples/triage.zig` — provider selection: `JEV_PROVIDER=openjev` forces
  OpenJEV; otherwise TypeSafe is used when `TYPESAFE_API_KEY` is set, falling
  back to OpenJEV when only `OPENJEV_API_KEY` is set. The model id switches
  automatically (`jev-latest` for TypeSafe, `openjev` for OpenJEV).
- `README.md` and `examples/README.md` — documentation of the new option.

No TypeSafe code, defaults, or documentation were removed or renamed.

## Provider selection rule

1. `JEV_PROVIDER=openjev` (or `typesafe`) — explicit choice wins.
2. Otherwise, if `TYPESAFE_API_KEY` is set → TypeSafe (unchanged default).
3. Otherwise, if only `OPENJEV_API_KEY` is set → OpenJEV.

The endpoint and model are the only differences between providers:

| | TypeSafe direct | OpenJEV |
|---|---|---|
| Endpoint | `https://api.typesafe.ai/v1/systemone` | `https://api.openjev.sh/v1/systemone` |
| Model | `jev-latest` | `openjev` |
| Key env | `TYPESAFE_API_KEY` | `OPENJEV_API_KEY` |

HTTP 503 (OpenJEV overload) is already retryable alongside 429 and the rest of
the 5xx range in `src/engine.zig`.

## How to configure

```sh
# TypeSafe (default, unchanged)
TYPESAFE_API_KEY=... zig build live

# OpenJEV (explicit)
OPENJEV_API_KEY=... zig build live

# OpenJEV (force when both keys are present)
JEV_PROVIDER=openjev OPENJEV_API_KEY=... zig build live
```

In library code, set `http.endpoint` after `Http.init` and pass the matching
model string to `client.evaluate`:

```zig
var http = try jevlin.Http.init(gpa, io, openjev_key);
http.endpoint = jevlin.Http.openjev_endpoint;
// ...
const result = try client.evaluate(state, questions, jevlin.Http.openjev_model, workspace, &diagnostics);
```

## Verification

- No hardcoded `api.typesafe.ai` default remains in source (grep confirmed; the
  TypeSafe URL lives only in the `typesafe_endpoint` constant, which is the
  unchanged default).
- A live `POST https://api.openjev.sh/v1/systemone` request with model `openjev`,
  state `ping`, and one noul question returned HTTP 200.

## Upstream

Original project: https://github.com/copyleftdev/jevlin by @copyleftdev.
