# grrr

![./grrr](./grrr-img.gif)

### **General Record, Replay, & Report**

A zero-dependency, language-agnostic record/replay/report tester for shell commands.

Record the observable behavior of any command [*exit status, stdout, stderr, raw bytes*] into a single binary snapshot. Later, replay those commands and get a precise report of anything that changed. Works the same whether your program is written in C, C++, Rust, Python, Haskell, Bash, or whatever language, as long as it is presented as a shell command.

```sh
$ ./grrr record
REC   hello  (exit 0, stdout 12 B, stderr 0 B)
REC   binary-nul  (exit 0, stdout 5 B, stderr 0 B)
recorded 14 test(s), snapshot has 14: tests.list.bi

$ ./grrr replay
grrr replay: tests.list (snapshot tests.list.bi)
PASS  hello
PASS  binary-nul
FAIL  exit-one
      status: expected exit 1, got exit 0
...
13 passed, 1 failed, 0 new, 0 stale
```

## Table of contents

1. [Why grrr?](#why-grrr)
1. [Install](#install)
1. [Quick start](#quick-start)
1. [The test list format](#the-test-list-format)
1. [Commands](#commands)
1. [Directives](#directives)
1. [Normalizers](#normalizers)
1. [Example use cases](#example-use-cases)
1. [Reviewing snapshot changes](#reviewing-snapshot-changes)
1. [How it works](#how-it-works)
1. [Exit codes](#exit-codes)
1. [Limitations](#limitations)
1. [FAQ](#faq)
1. [License](#license)
1. [Contributing](#contributing)

## Why grrr?

Most test frameworks are tied to a language, a toolchain, and an install. `grrr` is not. It is:

- **One file.** Drop `grrr` in your repo. It's pure Perl with no CPAN dependencies.
- **Language-agnostic.** It runs commands, not functions. Your program can be anything `sh -c` can run.
- **Binary-safe.** It captures raw bytes, not text. `\x00`, `\xff`, 500KB of zeros, not a problem, all fine. `backticks` and `qx//` cannot do this.
- **Nondeterminism-aware.** Timestamps, PIDs, addresses, and Windows line endings are normalized on both sides at replay, so you only see real changes.
- **Hardware-mocking friendly.** Build a mock and a real binary, or feed captured device data through @stdin. Record once on the real hardware, replay on CI without it.

Use `grrr` when you want golden-file regression tests for a command-line program and you don't want to install a test framework to get them. Do not use it for unit tests, property tests, or anything that needs structured assertions, see [Limitations](#limitations).

## Install

`grrr` is a single Perl script. It needs only Perl (v5.10 or newer) that ships with your OS.

Copy it anywhere on your `PATH`

```sh
$ curl -fsSL https://raw.githubusercontent.com/imdibene/grrr/refs/heads/main/grrr
$ chmod +x ~/bin/grrr
```

Or keep it in your project

```sh
$ cp grrr /path/to/your/project/grrr
$ chmod +x /path/to/your/project/grrr
```

Verify:

```sh
$ ./grrr version
grrr 0.1.0
```

On MacOS and Linux this works out of the box. On Windows, I don't know, you are on your own.
`grrr` uses `fork`, `pipe`, `IO::Select`, and `/bin/sh`.

## Quick start

Three commands. That's the whole workflow.

### Write a test list (one command per line)

```sh
$ cat > tests.list <<'EOF'
echo hello world
printf 'a\000b\000c'
sh -c 'exit 1'
EOF
```

`tests.list` is the input. Each non-comment line is a shell command that `grrr` will run via `/bin/sh -c`.

### Record the behavior of every command

```sh
$ ./grrr record
```

`grrr record` runs each command, captures `exit status`, `stdout`, and `stderr` as ***raw bytes***, and writes `tests.list.bi` next to the list. Commit both files to `git`.

### Later (or in CI), replay and compare

```sh
$ ./grrr replay
```

`grrr replay` runs each command again, compares against the snapshot, and prints a report. Exit code is `0` if everything matches, `1` if anything failed, `2` on error.

You edit your code, run `grrr replay`, and see exactly which behaviors changed. If the change is intended, `grrr record` updates the snapshot. If not, you have introduced a bug with your change, and you just catch it =).

## The test list format

One command per line. Empty lines and `#` comments are ignored. Directives (lines starting with `@`) apply to the next command.

```sh
# comments are fine

# a plain command
echo hello

# a directive gives the test a stable id
@name greeting
echo hello again

# stdin can be redirected from a file
@name piped-input
@stdin /tmp/input.txt
cat

# timestamps are normalized before comparison
@name timestamp-stable
@normalize timestamp
date -u +%Y-%m-%dT%H:%M:%SZ

# timeouts are per-test
@name never-returns
@timeout 0.5
sleep 30
```

Commands run in the current directory, through `/bin/sh -c`, so pipes, redirections, subshells, and environment variables all work:

```sh
printf 'c\nb\na\n' | sort
grrr_TEST_VAR=hello sh -c 'echo "$grrr_TEST_VAR"'
sh -c 'echo out; echo err >&2'
```

## Commands

### `grrr record [pattern]`

Run the tests and write their results to the snapshot.

```sh
$ ./grrr record                    # record all tests
$ ./grrr record 'binary'           # re-record only tests whose id matches /binary/
$ ./grrr record -f my.list         # use a custom test list
$ ./grrr record -s my.list.bi      # use a custom snapshot file
```

If the snapshot already exists, record merges, i.e. it replaces only the re-recorded tests and leaves the rest alone. This is how you update a subset without touching everything else.

### `grrr replay [pattern]`

Run the tests again and compare against the snapshot.

```sh
$ ./grrr replay                    # compare all tests
$ ./grrr replay 'hw'               # compare only tests whose id matches /hw/
$ ./grrr replay --quiet            # only print failures and the summary
$ ./grrr replay -n crlf,timestamp  # apply extra normalizers to both sides
```

Output reports each test as `PASS`, `FAIL`, `NEW`, or `STALE`, and for failures, shows the first differing line (for text) or the first differing byte offset (for binary).

### `grrr inspect [pattern]`

List what is stored in the snapshot without running anything.

```sh
$ ./grrr inspect                   # list all records: id, exit, sizes
$ ./grrr inspect --show            # also dump stdout/stderr
$ ./grrr inspect --show 'binary'   # dump only matching tests
```

Useful for answering "what did I record?" without a full replay.

### `grrr selftest`

Run `grrr`'s own test suite. Useful when you modify `grrr` itself.

```sh
$ ./grrr selftest
 ok 1 - foldl sums
 ok 2 - foldr order
 ...
```
## Options

Options go before or after the command. The first non-option argument is the command, the second is the pattern (a valid regex expression).

Option | Meaning | Default
:-|:-|:-
`-f, --file FILE` | Test list | `tests.list`
`-s, --snapshot FILE` | Snapshot file | `<FILE>.bi`
`-t, --timeout SECS` | Default timeout per test | `10`
`-n, --normalize LIST` | Comma-separated normalizers for all tests | none
`-p, --plugin FILE` | Load extra normalizers, repeatable | none
`-q, --quiet` | Hide `PASS` lines | off
`--show` | `inspect`: dump stdout and stderr | off
`-h, --help`, `-V, --version` | Usage, version |

### Report states

State | Meaning | Exit code
:-|:-|:-
`PASS` | Status, stdout and stderr match | 0
`FAIL` | Something differs (a timeout is noted) | 1
`NEW` | In the list, not in the snapshot | 1
`STALE` | In the snapshot, no longer in the list (not reported when you pass a pattern) | 0

## Directives

Directives appear on a line starting with @ and apply to the next command.

Directive| Meaning | Example
:-|:-|:-
`@name ID` | Stable test id (default: the command itself) | `@name hello`
`@stdin FILE` | Feed `FILE` to the command's stdin (default: `/dev/null`) | `@stdin input.bin`
`@timeout SECS` | Kill the command after `SECS` (default: --timeout, 10s) | `@timeout 0.5`
`@normalize LIST` | Comma-separated normalizers applied at replay | `@normalize crlf,timestamp`

You can use `@name` to make a test's id shorter and stable:

```sh
@name hello
echo hello world

@name hello-again
echo hello world again
```

Without `@name`, the id is the command. That's fine for short commands and annoying for long ones.

## Normalizers

Normalizers are functions that transform bytes before comparison. They run on both the recorded and the current output, at replay time. Use them to strip the parts of your output that legitimately change between runs.

```sh
@name timestamp
@normalize timestamp
date -u +%Y-%m-%dT%H:%M:%SZ
# 2026-11-06T10:11:12Z  ->  <TIMESTAMP>
```

Built-in normalizers:

Name | Transformation
:-|:-
`crlf` | `\r\n` -> `\n`
`trailing-ws` | Strip trailing whitespace from every line
`timestamp` | `2026-11-06T10:11:12Z` -> `<TIMESTAMP>`
`address` | `0xDEADC0DE` (6+ hex digits) -> `0xADDR`

Stack them with commas:

```sh
@name messy-output
@normalize crlf,trailing-ws
printf 'padded   \r\nlines\t\t\r\n'
```

Apply globally with `--normalize`:

```sh
$ ./grrr replay --normalize crlf,timestamp
```

Normalizers apply to **stdout and stderr** on both sides, never to the exit status. The snapshot stores the raw bytes, so adding or changing a normalizer never requires re-recording.

**Order matters.** Global `--normalize` names run first, then the test's `@normalize`, each left to right. For `printf '\033[32mpid=7\033[0m'`, `strip-ansi,pid` works, but `pid,strip-ansi` doesn't: `\bpid=` needs a word boundary, and the escape sequence ends in `m`.

For project-specific needs, write a [plugin](#plugins).

## Plugins

A plugin is a Perl file that returns a hash reference of `name => closure`:

```perl
# grrr.plugins.pl
return {
    'strip-ansi' => sub { (my $s = $_[0]) =~ s/\e\[[0-9;]*m//g;                          return $s },
    'tempdir'    => sub { (my $s = $_[0]) =~ s{/tmp/grrr-[A-Za-z0-9]+}{/tmp/grrr-XXX}g;  return $s },
    'pid'        => sub { (my $s = $_[0]) =~ s/\bpid=\d+/pid=<PID>/g;                   return $s },
};
```

Load it with `--plugin` (or `-p`, repeatable) and use the names like built-ins:

```sh
$ ./grrr --plugin grrr.plugins.pl replay
$ ./grrr -p base.pl -p project.pl replay -n strip-ansi
```

```sh
@name volatile
@normalize strip-ansi,pid,tempdir
./build/tool --verbose
```

Rules:

- **Pure.** A normalizer takes bytes and returns bytes, the same result for the same input. On load, `grrr` calls each one twice on a probe string and rejects it if it dies, returns a non-string, or gives different results.
- **Unique names.** Letters, digits, `_` and `-`. A plugin cannot redefine a built-in or a name from another plugin.
- **Pass `--plugin` on every call that reads the list, `record` included.** The list is validated everywhere, so without the plugin you get `unknown normalizer(s): pid` and exit code 2.
- **No auto-loading.** A plugin is Perl run with `do`, so treat it like any script in your repo. `grrr` never loads one on its own.

Plugin errors (missing file, not a hash, a value that isn't code, impure, duplicate name) all exit with code 2 and name the file and the normalizer.

## Example use cases

### 1. Regression testing a C or C++ CLI

```sh
# tests.list
@name parse-ok
./build/parser fixtures/ok.json

@name parse-error
./build/parser fixtures/bad.json

@name parse-binary
./build/parser fixtures/raw.bin
```

Record once. Every commit, run `./grrr replay` in CI. Any change to the parser's output like error messages, JSON formatting, binary handling will shows up as a diff.

### 2. Testing a compiler's diagnostics

```sh
@name error-undefined-var
./build/mycc -c fixtures/undef.c

@name warning-unused
./build/mycc -Wall -c fixtures/unused.c

@name error-traceback
@normalize address
./build/mycc -c fixtures/crash.c
```

The `@normalize` address is the key: stack traces and heap addresses change every run, but everything else in the diagnostic doesn't.

### 3. Testing a Python CLI without pytest

```sh
@name help
python3 -m mytool --help

@name parse
python3 -m mytool parse fixtures/input.txt

@name failure
python3 -m mytool parse fixtures/missing.txt
```

You get the same coverage as a pytest golden-file test, with no `pip install` and all that python dependency management hell on CI.

### 4. Hardware mocking: two builds

Build the application twice from the same code. The "production/real-build" and the "hardware-mocking-build" is chosen at link time, so the release binary contains no mock code and reads no test-related environment variable.

Three ways to exercise the code, from cheapest to most realistic:

1. **Mock build** (`hw-mock`): tests your logic with no hardware.
2. **Captured data** (`hw-captured`): the real release binary, fed a device reading recorded once and checked in, via `@stdin`. It runs your real parsing code without the device. This only works if the app reads its device as a stream and the device path is configurable.
3. **Real hardware** (`hw-real`): record once on the hardware machine. CI fails it with a distinct exit code (77), and you skip it with `./grrr replay -q '^(?!hw-real)'`.

```sh
make                 # build/mock/hwtest and build/release/hwtest
```

```
@name hw-mock
./build/mock/hwtest --self-check

@name hw-captured
@stdin /path/to/my/fixture/data
./build/mock/hwtest --self-check

@name hw-real
./build/release/hwtest --self-check
```

Record once on the machine with the hardware (`make record`). CI runs `make check`: `hw-mock` passes without hardware, and `hw-real` fails because the device is missing. Have your program exit with a distinct code and message in that case (77 is the automake "skipped" convention), and the diff will show why:

```
FAIL  hw-real
      status: expected exit 0, got exit 77
      ...
      + 1 | hwtest: real sensor not available
```

To skip hardware tests in CI: `./grrr replay -q '^(?!hw-real)'`. To refresh only the hardware record: `./grrr record '^hw-real$'`.

### 5. Testing code that reads stdin

```sh
# create the input file
printf '\x00\x01\x02\xff' > fixtures/input.bin
```

```sh
@name reads-binary-stdin
@stdin fixtures/input.bin
./build/reader
```

This is a case where `grrr`'s `IO::Select`-based capture genuinely does something *shell tools* can't: the input has [NUL](https://en.wikipedia.org/wiki/Null_character) bytes, and the output may too. `cat` and `diff` would mangle it.

### 6. Testing a formatter or linter

```sh
@name format-json
jq . fixtures/pretty.json

@name format-csv
./build/csvfmt fixtures/data.csv

@name lint-clean
./build/lint fixtures/good.c

@name lint-errors
./build/lint fixtures/bad.c
```

### 7. Cross-language integration testing

If you have a C library, a Python binding, and a CLI, `grrr` tests all three with the same snapshot:

```sh
@name c-api
./build/c_example

@name python-binding
python3 -c 'import mylib; print(mylib.run())'

@name cli
./build/cli run
```

One `tests.list`, one snapshot, one report. `pytest` + `GoogleTest` + shell scripts would need three fixtures and three CI steps.

## Reviewing snapshot changes

The snapshot is a binary file (`tests.list.bi`). It is designed to be `xxd`-explorable, i.e. every field is fixed-width or length-prefixed, but it is not `git diff`-friendly. Git will say "binary files differ" and stop.

For debugging the format itself or recovering a partially corrupt snapshot just use `xxd`:

```sh
xxd tests.list.bi | head -20
xxd tests.list.bi | grep -a 'hello'
```

The first four bytes are `GRRR`, followed by a `u32be` record count, followed by length-prefixed records. You can decode it by hand.

If you want a "Human readable" use `grrr inspect`:

```sh
./grrr inspect --show
./grrr inspect --show -s my-testsrun-snapshot.bi
```

You can combine it with output redirection and `diff` to compare what you have commited against your new run:

```sh
git show HEAD:tests.list.bi > /tmp/old.bi
./grrr inspect --show -s /tmp/old.bi > /tmp/old.txt
./grrr inspect --show -s tests.list.bi > /tmp/new.txt
diff -u /tmp/old.txt /tmp/new.txt
```

## How it works

`grrr` is structured as a functional core with an imperative shell:

```text
tests.list --parse--> tests --run--> records --serialize--> snapshot.bi
                            ^
                    (run_command: fork, exec, IO::Select)
```

- **Pure functions**: take data and return data. They are tested by `grrr selftest` without touching the filesystem or the network.
    - `parse_tests`
    - `serialize_results`
    - `deserialize_results`
    - `normalize_record`
    - `compare_records`
    - `format_report`

- **Impure boundary**: is the only part that touches the outside world. It is injected into the commands via the `$io` hash, so tests can substitute fakes.
    - `run_command`
    - `slurp`
    - `spew_atomic`
    - `load_plugin_file`

This is why `grrr` selftest can run 47 assertions in under a second without depending on any external tool: the whole pipeline is exercisable with a fake runner and in-memory files.

### Snapshot format

```text
snapshot := "GRRR" count:u32be record*
record   := id:chunk status:i32be stdout:chunk stderr:chunk
chunk    := length:u32be bytes
```

Simple, tagged (GRRR), and portable across Perl versions. No `Storable`, no `JSON`, no `XML`, no dependency. Length-prefixed so binary payloads (including **NUL** bytes) round-trip byte-for-byte.

### Subprocess capture

`grrr` uses `fork` + `pipe` + `IO::Select` rather than `backticks` or `system`. This has three consequences:

1. Binary-safe. `sysread` returns raw bytes; `backticks` would decode as text and corrupt `\x00` and `\xff`.
1. Deadlock-free. `stdout` and `stderr` are read concurrently, so a child that writes 500KB to `stderr` before touching stdout cannot block.
1. Timeout-killable. Each child is its own process group, so `@timeout` can kill `sh -c 'sleep 30 | cat'` as a tree, not just the top process.

## Exit codes

Code | Meaning
:-|:-
`0` | All tests passed.
`1` | At least one test failed, or a test in the list has no snapshot (run `./grrr record`).
`2` | Usage error, unreadable file, corrupt snapshot.

CI integration:

```sh
./grrr replay || exit 1
```

## Limitations

`grrr` is a golden-file regression tester, not a full test framework. It does not do:

- Unit testing. It runs commands, not functions. Pure functions need their own tests.
- Structured assertions. You cannot say "output must contain X". You can only say "output must be exactly these bytes", with `@normalize` as the escape hatch.
- Test isolation. Tests share the current directory and environment. If test A creates `/tmp/foo` and test B expects it absent, you get order-dependent failures.
- Parallelism. Tests run one at a time.
- Coverage, profiling, fuzzing.

For these, use `pytest`, `GoogleTest`, `tasty`, `cargo test`, `QuickCheck`, `Hypothesis`, whichever fits your language. `grrr` complements them; it doesn't replace them.

Its niche is: "`diff` + a `.expected` file almost works, but I need binary safety, normalization, or hardware mocking."

## FAQ

### Why Perl?

Because I'm a *boomer* at heart, so, use a boomer language. Joking aside, it is because **Perl** it's on every Unix by default, it has no build step, and it ships with `IO::Select` and `fork`. A single-file test tool that works on any Linux or MacOS box with zero install is worth the syntactic cost.

### Why not a shell script?

Perl is basically *sh on steroids*. But, shell cannot safely capture binary output. `$(...)` and `read` decode as text and mangle `\x00`. `grrr` uses `sysread`, which returns raw bytes.

### Why not Python?

Python is a pain in the @$$ to work with in my experience, it is not installed by default on many minimal CI images, it has the dependency management hell (virt env, are also a pain to work with) and `python3` is slow af, it starts in ~30ms, enough to matter when you record 500 or so tests. Perl starts in ~10ms and is on the base image of every Unix-like system.

### Why a binary snapshot?

The payload (`stdout`/`stderr`) is binary. A text format would need escaping, which makes the snapshot larger and slower and still not diff-friendly see [Reviewing snapshot changes](#reviewing-snapshot-changes) for the text projection that is diff-friendly.

### Can I use `grrr` from a `Makefile`?

Yes:

```makefile
test:
	./grrr replay || exit 1

record:
	./grrr record
```

### Can I use `grrr` from CI?

Yes, it's a single file with no dependencies:

```yaml
# .github/workflows/test.yml
- name: Install grrr
  run: |
    curl -fsSL https://raw.githubusercontent.com/imdibene/grrr/refs/heads/main/grrr
    chmod +x /usr/local/bin/grrr

- name: Replay
  run: grrr replay
```

### My test output has a timestamp in it. What do I do?

Add `@normalize timestamp` above the command, or use `--normalize timestamp`. If your format isn't covered, write a [plugin](#plugins).

### My test output has a random PID in it. What do I do?

Add `@normalize address` if it looks like `0x...`. Otherwise write a [plugin](#plugins) with a `pid` normalizer, such as the one above.

### Why does my command fail with "no test matches"?

The first positional argument after the command is a regex pattern, not a file path. Use `-f` for the test list and `-s` for the snapshot:

```sh
./grrr record -f tests.list              # correct
./grrr record tests.list                 # wrong: 'tests.list' is treated as a filter
./grrr replay -s custom.bi               # correct
```

### How do I skip a test?

Comment it out. `grrr` has no skip mechanism, because in a golden-file tool "skip" usually means "the output is nondeterministic", which `@normalize` handles better.

### How do I test a program that needs a network?

Point it at a local mock server (`nc -l`, `python3 -m http.server`, or your own fake), and use `@normalize` to strip ports and timestamps. Or make the program read from a fixture instead of the network.

### How do I update one test after an intentional change?

```sh
./grrr record 'my-test-name'
```

This re-records only matching tests and merges the result into the existing snapshot. Everything else stays untouched.

### Can I have multiple snapshot files?

Yes:

```sh
./grrr record -f unit.list -s unit.bi
./grrr record -f integration.list -s integration.bi
./grrr replay -f unit.list -s unit.bi
```

Use `-f` for the test list, `-s` for the snapshot. If `-s` is omitted, it defaults to `<file>.bi`, also, if `-f` is omitted, it defaults to `tests.list`.
`-s` only selects the snapshot. The list always comes from `-f` (default `tests.list`), and the default snapshot name is `<list>.bi`. When in doubt, pass `-f` and let `-s` default.

## License

MIT. See [LICENSE](./LICENSE)

## Contributing

Bug reports, feature requests, and pull requests are welcome. Run `./grrr selftest` before submitting it must pass with exit code `0`.

If you add a feature, add a test to `cmd_selftest`. The self-test uses the same `make_fake_io` fixture mechanism as the rest of the codebase, so testing a new command is usually 5–10 lines.

