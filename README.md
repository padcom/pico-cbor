# pico-cbor

[CBOR](https://www.rfc-editor.org/rfc/rfc8949) (RFC 8949) for [Pico](https://github.com/padcom/pico-lang): encode and decode
CBOR items, write them in diagnostic notation, and read typed values out of them for schema decoders. All of CBOR by
default (tags, indefinite lengths, simple values, maps with any keys); options restrict it, and `PICO_DESKTOP` is the
profile [Pico Desktop](https://github.com/padcom/pico-desktop)'s protocol uses. Plain Pico, no native code: it runs on
every platform Pico does.

It needs Pico 0.10.0 or later (`DataView`, `std/bytes`).

## Use

```ts
import { Cbor, decode, diagnostic, encode, integer } from "padcom/pico-cbor"

const point = new Map<string, Cbor>()
point.set("x", integer(3))
point.set("y", Cbor.Float(1.5))
const bytes = encode(Cbor.Dict(point))   // a2 6178 03 6179 f93e00
println(diagnostic(decode(bytes)))      // {"x": 3, "y": 1.5}
```

Add it with `pico add https://github.com/padcom/pico-cbor --tag v0.1.0`.

## The data model

`Cbor` is an enum: `Uint(u64)`, `Negative(u64)` (the integer −1 − n, down to −2^64), `Bytes`, `Text`, `List`, `Dict` (a
map whose keys are all text), `Keyed` (all unsigned integers), `Pairs` (any other keys, as `CborEntry { key, value }`),
`Tag` (a `Tagged { tag, value }`; build one with `tagged(tag, value)`), `Bool`, `Null`, `Undefined`, `Simple(int)` and
`Float(f64)`. Maps keep their order. `integer(n)` makes `Uint` or `Negative` of an `int`.

## Encoding

`encode(value, options?)` writes RFC 8949's preferred serialization: the shortest head for every length and integer,
definite lengths, and the shortest float (half, single or double precision) that keeps the value; NaN is `f97e00`.
`floats: "double"` writes every float in 8 bytes instead.

## Decoding

`decode(bytes, options?)` reads exactly one item (trailing bytes are an error) and throws a `CborError` saying what's
wrong for anything else: truncated items, reserved values, invalid UTF-8, duplicate map keys (also when encoded
differently), nesting deeper than `maxDepth`. It never panics, whatever the bytes (`fuzz.test.pi`). Byte strings are
copies, never views of the input.

## Options

| Option | Default | |
|---|---|---|
| `maxDepth` | 64 | the deepest nesting of arrays, maps and tags |
| `tags` | `true` | tags allowed |
| `indefinite` | `true` | indefinite-length strings, arrays and maps decoded |
| `simple` | `true` | `undefined` and simple values other than `false`, `true`, `null` allowed |
| `anyKeys` | `true` | map keys other than text or unsigned integers (and mixed ones) allowed, as `Pairs` |
| `floats` | `"shortest"` | `"double"`: every float in 8 bytes |

`PICO_DESKTOP` is `{ maxDepth: 32, tags: false, indefinite: false, simple: false, anyKeys: false, floats: "double" }`;
`profile.test.pi` checks it against Pico Desktop's shared vectors (`vectors/pico-desktop.json`).

## Diagnostic notation

`diagnostic(value)` writes RFC 8949 §8's notation: `[1, {"a": h'00ff'}]`, `1(1363896240)`, `simple(16)`, floats as
numbers (`1.5`, `-0.0`, `NaN`, `Infinity`). With `{ floatBits: true }` floats are their double bits instead
(`float(0x3ff8000000000000)`), which language-neutral test vectors compare exactly.

## Reading typed values

For decoders of a schema, hand-written or generated: `members(v, path, known)` (a closed text-keyed map), `required(m,
key, path)`, `text`, `bytes`, `boolean`, `isNull`, `uint(v, path, max)`, `int`, `float` (integers count), `choice(v, path,
values)`, `items` and `tagOf(v, path, tag)` (a union's tag member). Each fails with an `InvalidPayload` that names the
path (`$.user.name: expected text`).

## Tests

`pico test .` (and `pico test --interp .`): RFC 8949 Appendix A, Pico Desktop's vectors, the options and a fuzz run.

## License

MIT
