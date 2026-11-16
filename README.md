# lua

[Lua](https://www.lua.org) 5.4 for [sysl](https://github.com/sysl-lang/sysl): run a script, read what
it left behind, and hand it sysl functions to call.

*An independent sysl package for Lua. It is not affiliated with or endorsed by Lua's authors or trademark holders.*

```sysl
import sh.sysl.lua.{state, Error}

run() -> Result[unit, Error]
    val lua = state()?

    lua.run("function greet(name) return 'hello, ' .. name end")?
    print(lua.global("greet")?.call([lua.text("sysl")])?.text()?)     // hello, sysl

    val scale = 3

    lua.define("scale", (call) -> Ok(call.state.integer(call.integer(0)? * scale)))?
    print(lua.eval("scale(14)")?.integer()?)                          // 42

    Ok(())

main()
    run() match
        Ok(_) -> ()
        Err(e) -> print(e)
```

```hocon
dependencies {
  lua { git = "github.com/sysl-lang/lua", version = "0.1.0" }
}
```

**Nothing has to be installed.** Lua is vendored: `c/lua/` is upstream's `src/` directory from
`lua-5.4.9.tar.gz`, byte for byte, and sysl compiles it as part of the build.

| | |
|---|---|
| upstream | <https://www.lua.org/ftp/lua-5.4.9.tar.gz> (2026-08-10) |
| SHA-256 | `2335b6c582a52654f94612bf10d2f4672805d05329aa6568b1d8cd9e5c6fb8e6` (matches lua.org's published sum) |
| test suite | <https://www.lua.org/tests/lua-5.4.9-tests.tar.gz>, SHA-256 `7d971845f545ffc09fbb3128a86b2c6524161c70d0fdf0154a16e8c00c343fca` |
| licence | MIT, Copyright © 1994–2026 Lua.org, PUC-Rio (the notice is at the end of `c/lua/lua.h`) |

**Why vendored rather than Homebrew's.** Lua is written to be embedded in anything with a C
library, and every file compiles for a Cortex-M33 against newlib, so it can run on a board with a
libc (the Pico SDK, Zephyr, FreeRTOS). It does not build for a bare target with no libc —
`string.h`, `setjmp.h` and `math.h` are where it stops. Homebrew's `lua` is also 5.5 now, with
`lua@5.4` keg-only and deprecated.

## The two layers

```
sh/sysl/lua/
    lua.sysl          sh.sysl.lua — the embedding: State, Value, Call, Error
    tests.sysl        the embedding, against the manual's examples and Lua's own messages
    suite_tests.sysl  Lua's official test suite, run through the embedding
    c/
        c.sysl        sh.sysl.lua.c — lua.h, lauxlib.h and lualib.h, under their C names
        macros.sysl   the header's function-like macros, as sysl functions
        tests.sysl    every group of the C API, and the struct layouts against the header
c/lua/                Lua 5.4.9's src/, unchanged
testdata/             a script for the file tests, and the suite files
```

**`sh.sysl.lua` is what a program names.** A `State` is one Lua universe; a `&Value` holds one Lua
value alive (a registry slot) and lets it go in its destructor; a `Call` is what a registered sysl
function is handed. Nothing in it touches Lua's stack from the outside.

- `state()` opens every standard library; `bare()` opens none and `open(Library.X)` adds them one at
  a time.
- `run`, `eval`, `eval_all`, `load`, `run_file`, `load_file`. **`eval` tries the source as an
  expression first**, as the stand-alone interpreter does, so `eval("1 + 2")` is 3.
- `global`, `set_global`, and `nil`/`boolean`/`integer`/`number`/`text`/`table` to make values.
- A value: `kind`, `truthy`, `integer`, `number`, `text`, `bytes`, `describe` (Lua's `tostring`),
  `get`/`at`/`index`, `set`/`set_at`/`set_index`, `len`, `pairs`, `call`, `call_all`. Every
  operation that can run a metamethod runs inside a protected call, so a metamethod that errors is an
  `Err`.
- `define(name, f)` / `function(name, f)` register a sysl closure; `define_many` / `function_many`
  for one answering several values. `Call.integer(i)` and its neighbours fail with **the message
  Lua's own library gives** — `bad argument #1 to 'scale' (number expected, got string)` — which the
  tests check against Lua's.
- `limit_memory(bytes)`, `memory_used()`, `limit_steps(n)`: the allocator counts every byte, and a
  count hook stops a script that runs too long, past any `pcall` inside it.
- An `Error` has a kind (`Runtime`, `Syntax`, `Memory`, `Handler`, `File`, `Type`), a message and a
  traceback taken where the error was raised.

**`sh.sysl.lua.c` is the C API, all of it** — §4 and §5 of the reference manual, each function under
the name the manual prints (`lua_pushinteger`, `luaL_checkoption`, `luaopen_math`), the constants
measured from the header, `lua_Debug`/`luaL_Buffer`/`luaL_Reg`/`luaL_Stream` laid out field for field
and checked against `sizeof`/`offsetof`, and the function-like macros as sysl functions.

## What is not here, and why

- **`lua_pushliteral`, `luaL_newlib`, `luaL_newlibtable`, `luaL_argcheck`, `luaL_argexpected`,
  `luaL_opt`, `luaL_intop`** — macros with no function form: one pastes a string literal, two take a C
  array's element count, the rest are an `if` or an expression better written at the call. The note
  in `c/macros.sysl` says what to write.
- **`LUA_VERSION`, `LUA_COPYRIGHT` and the other string macros** are not measured, since a `c const`
  reads numbers; the names Lua looks up (`LUA_GNAME`, `LUA_LOADED_TABLE`, the library names) are
  transcribed, and a test checks each against what Lua registered.
- **`LUA_MININTEGER`** is not measured: a `c const` cannot read the most negative `i64` (filed
  against the compiler). `LUA_MAXINTEGER` is.
- **Yielding from a sysl function** (`lua_yieldk` with a continuation) is in the C layer and not in
  the embedding. A sysl function can be *called* from a coroutine, and a script's own coroutines work.
- **Loading C modules from shared libraries** (`LUA_USE_DLOPEN`) is off: an embedding hands Lua its
  sysl functions instead.

## Three rules worth knowing

**A Lua error is a `longjmp`.** It unwinds every frame between the raise and the nearest `pcall`
without running their cleanup, so a sysl frame Lua can unwind through must hold nothing counted. The
embedding is arranged so: the function Lua calls for every registered closure holds a pointer and an
integer, and every API call that may raise runs inside a protected call. Code using the C layer
directly carries the same rule.

**An error crossing a sysl function becomes its message.** A sysl function answering `Err(e)` raises
`e.message` as a Lua string; a script's `pcall` sees that string, not the original error object.

**A registered sysl function is a C closure with one upvalue** (the closure's address), which
`debug.getinfo(f).nups` reports. It is freed with its state, together with whatever it captured; a
closure that captured a `&Value` of its own state keeps that state alive in a cycle.

## Tests

`sysl test .` — 84 tests, 83 run and 1 ignored: the C layer group by group against a running interpreter, the embedding
against the reference manual's worked examples (§4.6's `lua_call`, §6.4.1's `string.gsub`) and against
the messages Lua's own library functions give for the same mistakes, and **twenty files of Lua's
official test suite** (`strings`, `math`, `sort`, `tpack`, `utf8`, `vararg`, `closure`, `goto`,
`bitwise`, `events`, `nextvar`, `pm`, `literals`, `constructs`, `locals`, `coroutine`, `errors`,
`calls`, `db`, `gc`) run through `run_file`, with `_port` and `_soft` set as `all.lua` sets them for a
portable run. `api.lua` and the rest need Lua's internal test library and are not run. `gc.lua` is
the ignored one: every assertion in it holds but line 487, a one-kilobyte tolerance on memory left
after freeing two 4 MB strings, which measures the depth of the caller's stack — the stock `lua`
built from these sources fails the same block run alone.

Under AddressSanitizer (`SYSL_EXTRA_CFLAGS="-fsanitize=address -g" sysl test .`) the run reports
`instrumented: address (1 binary)` and passes, and because Lua is vendored its C is compiled with the
flag too: the kept test binary's `luaV_execute` alone carries 819 `__asan_report` calls. The run
covers the interpreter, not only the sysl half.
