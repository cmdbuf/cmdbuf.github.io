# Command buffers: a field guide

This is a condensed, example-first guide to writing Go automation with
[lesiw.io/command](https://pkg.go.dev/lesiw.io/command) and
[lesiw.io/fs](https://pkg.go.dev/lesiw.io/fs). It is the companion to
[cmdbuf.io](https://cmdbuf.io) and is written to be equally useful to
people and to coding agents. Every example in it compiles and runs
against the current releases.

## The mental model

Command buffers treat automation as what it physically is: bytes moving
between commands and files. Five facts cover the whole design.

1. A `Buffer` is a command's execution as an `io.Reader`. The command
   starts on the first `Read` and is finished at `io.EOF`. There is no
   `Start` and no `Wait`.
2. A `Machine` is anything that can run a command. It has one method:
   `Command(ctx context.Context, arg ...string) Buffer`.
3. Machines wrap machines. `ssh.Machine`, `ctr.Machine`, and
   `sub.Machine` each take a Machine and return a new one, so
   environments nest: a container on a remote host is
   `ctr.Machine(ssh.Machine(sys.Machine(), "ssh", "user@host"), "img")`.
4. Files are buffers on machines too. `command.FS(m)` returns a
   filesystem for any Machine. Copying between machines is `io.Copy`.
   A trailing slash means a directory, which streams as a tar archive.
5. `command.Shell` (type `*command.Sh`) makes automation portable:
   it wraps a Machine with operations that work on any machine plus
   an explicit list of allowed external commands. Everything not on
   the list fails with "command not found" at call time.

There is no shell in this vocabulary. No `sh -c`, no `bash -c`, no
`cmd /c`, no string splitting, no quoting rules. Arguments are passed
as they are written. Anything a shell would have done — piping,
substitution, redirection — has a Go expression instead, listed below.

## Translating from shell

| Shell idiom | Command buffer equivalent |
|---|---|
| `out=$(git describe)` | `out, err := command.Read(ctx, m, "git", "describe")` |
| `cmd >/dev/null 2>&1` | `err := command.Do(ctx, m, "cmd")` |
| run interactively | `err := command.Exec(ctx, m, "cmd")` |
| `a \| b` | `io.Copy(command.NewWriter(ctx, m, "b"), command.NewReader(ctx, m, "a"))` |
| `a \| b \| c` | `command.Copy(dst, src, command.NewFilter(ctx, m, "b"))` |
| `VAR=x cmd` | `command.Do(command.WithEnv(ctx, map[string]string{"VAR": "x"}), m, "cmd")` |
| `echo $VAR` | `command.Env(ctx, m, "VAR")` |
| `cat file` | `sh.ReadFile(ctx, "file")` |
| `echo x > file` | `sh.WriteFile(ctx, "file", []byte("x"))` |
| `mkdir -p dir` | `sh.MkdirAll(ctx, "dir")` |
| `rm -rf dir` | `sh.RemoveAll(ctx, "dir")` |
| `mv a b` | `sh.Rename(ctx, "a", "b")` |
| `chmod 755 f` | `sh.Chmod(ctx, "f", 0o755)` |
| `mktemp` / `mktemp -d` | `sh.Temp(ctx, "prefix")` / `sh.Temp(ctx, "prefix/")` |
| `test -e file` | `_, err := sh.Stat(ctx, "file")` |
| `which cmd` | `command.NotFound(command.Do(ctx, m, "cmd", "--version"))` |
| `uname -s` / `uname -m` | `sh.OS(ctx)` / `sh.Arch(ctx)` |
| `tar -cf- dir` | `sh.Open(ctx, "dir/")` |
| `tar -xf- -C dir` | `sh.Create(ctx, "dir/")` |
| `scp f host:f` | `io.Copy` between two machines' shells (see Files) |
| `ssh host cmd` | `command.Read(ctx, ssh.Machine(m, "ssh", "host"), "cmd")` |
| `docker exec ctr cmd` | `command.Read(ctx, ctr.Machine(m, "image"), "cmd")` |
| `set -x` | `CMDTRACE=on` in the environment |
| `grep`, `awk`, `sed` | Go stdlib: `bufio.Scanner`, `strings`, `regexp` |

## Program skeleton

Standalone automation uses the `run()` pattern, so errors have one exit
path and the code stays testable:

```go
package main

import (
    "context"
    "fmt"
    "os"

    "lesiw.io/command"
    "lesiw.io/command/sys"
)

func main() {
    if err := run(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}

func run() error {
    ctx := context.Background()
    sh := command.Shell(sys.Machine(), "git")

    commit, err := sh.Read(ctx, "git", "rev-parse", "HEAD")
    if err != nil {
        return fmt.Errorf("read commit: %w", err)
    }
    fmt.Println(commit)
    return nil
}
```

## Running commands

Three helpers cover nearly every call. Choose by what you want from the
output:

```go
// Capture output as a string. Trailing newlines are stripped,
// like $(...) in a shell.
version, err := command.Read(ctx, m, "go", "version")

// Discard output; run for the side effect or the error.
err = command.Do(ctx, m, "gofmt", "-l", ".")

// Attach output (and, when possible, the terminal) to the user.
// Use for builds, tests, and anything interactive.
err = command.Exec(ctx, m, "go", "test", "./...")
```

All three exist as methods on `*command.Sh` with the same semantics:
`sh.Read`, `sh.Do`, `sh.Exec`.

Environment variables travel on the context:

```go
ctx = command.WithEnv(ctx, map[string]string{"CGO_ENABLED": "0"})
err := command.Exec(ctx, m, "go", "build", ".")
```

So does the working directory:

```go
ctx = fs.WithWorkDir(ctx, "/opt/app")
out, err := command.Read(ctx, m, "pwd") // "/opt/app"
```

To see every command as it runs (like `set -x`):

```go
command.Trace = os.Stderr
```

### Errors

Failed commands return a `*command.Error` carrying the exit code and
any stderr output:

```go
err := command.Do(ctx, m, "go", "build", ".")
var cerr *command.Error
if errors.As(err, &cerr) {
    fmt.Println("exit code:", cerr.Code)
    fmt.Printf("stderr:\n%s", cerr.Log)
}
```

When a multi-stage `command.Copy` pipeline fails, the returned error
reports every stage and its outcome, so the failing stage is
identifiable:

```go
_, err := command.Copy(
    command.NewWriter(ctx, m, "wc", "-c"),
    command.NewReader(ctx, m, "echo", "not gzip data"),
    command.NewFilter(ctx, m, "gzip", "-d"),
)
fmt.Println(err)
// <*command.reader>
//     <success>
//
// <*command.filter>
//     exit status 1
//         gzip: unknown compression format
```

To check whether a command exists, run it and test the error — there
is no `which`:

```go
err := command.Do(ctx, m, "gotestsum", "--version")
if command.NotFound(err) {
    return sh.Exec(ctx, "go", "install", "gotest.tools/gotestsum@latest")
}
```

## Piping

Two stages are `io.Copy`. The writer's `ReadFrom` optimization closes
stdin automatically when the source hits EOF:

```go
// echo "hello, pipes" | tr a-z A-Z
_, err := io.Copy(
    command.NewWriter(ctx, m, "tr", "a-z", "A-Z"),
    command.NewReader(ctx, m, "echo", "hello, pipes"),
)
```

Three or more stages are `command.Copy`, with `command.NewFilter` for
the middle stages. Commands and files mix freely in one pipeline —
this dumps a database, compresses it in flight, and lands it on a
different machine with no intermediate files:

```go
backup := ssh.Machine(m, "ssh", "backup@vault.example.com")
_, err := command.Copy(
    fs.CreateBuffer(ctx, command.FS(backup), "db.sql.gz"),
    command.NewReader(ctx, m, "pg_dumpall"),
    command.NewFilter(ctx, m, "gzip"),
)
```

Buffers compose with anything that speaks `io`: `strings.NewReader` as
a source, `bytes.Buffer` as a sink, `io.MultiWriter` for tees.

```go
// kubectl apply -f - <<< "$manifest"
_, err := io.Copy(
    command.NewWriter(ctx, m, "kubectl", "apply", "-f", "-"),
    strings.NewReader(manifest),
)
```

## Machines

The provided machines, and what wraps what:

```go
m := sys.Machine()                                  // the local system
m := ssh.Machine(host, "ssh", "user@example.com")   // a remote host, via ssh run on `host`
m := ctr.Machine(host, "alpine:latest")             // a container, via docker/podman/nerdctl on `host`
m := sub.Machine(host, "busybox")                   // `host`, with every command prefixed
m := mem.Machine()                                  // in-memory: echo, cat, tee, tr; Playground-safe
m := new(mock.Machine)                              // programmable responses for tests
```

Notes:

- `ssh.Machine`'s variadic arguments are the **entire ssh command
  line, including the `ssh` command itself**. This is what makes
  ports, identity files, `sshpass`, `autossh`, and jump hosts all
  work with no special API: `ssh.Machine(sys.Machine(), "sshpass", "-p", pw,
  "ssh", "-p", "2222", "user@host")`.
- `ctr.Machine` starts its container lazily on first use and finds
  docker, podman, or nerdctl automatically. Pair it with
  `defer command.Shutdown(ctx, m)` to clean the container up.
  If the image name starts with `/` or `.`, it is treated as a path
  to a Containerfile and built.
- `command.Shutdown` is a no-op for machines with nothing to clean
  up, so it is always safe to call.
- OS and architecture detection: `command.OS(ctx, m)` and
  `command.Arch(ctx, m)` return normalized GOOS/GOARCH values
  ("linux", "darwin", "windows"; "amd64", "arm64") by probing the
  machine. `sh.OS(ctx)` and `sh.Arch(ctx)` cache the result.
- Windows works as the local system and as an ssh target. Commands
  to Windows remotes travel as a base64-encoded PowerShell script, so
  argument quoting survives any intermediate shell — including chains
  like unix to Windows to unix.

### Composition

Machines take machines, so environments nest by construction:

```go
// A container on a remote build host.
host := ssh.Machine(sys.Machine(), "ssh", "admin@build.example.com")
m := ctr.Machine(host, "golang:latest")
defer command.Shutdown(ctx, m)

sh := command.Shell(m, "go")
err := sh.Exec(ctx, "go", "test", "./...")
```

A prefix glued onto a command string — `ssh host ...`,
`docker exec ...`, `env FOO=bar ...` — is a machine that hasn't been
named yet. Once it is one, the code that runs commands no longer
depends on where they run.

A machine can also be a single command, enriched. `command.HandleFunc`
routes one command name through a function; this shim injects an
environment variable into every `go` invocation:

```go
m := command.HandleFunc(sys.Machine(), "go",
    func(ctx context.Context, args ...string) command.Buffer {
        ctx = command.WithEnv(ctx, map[string]string{
            "GOFLAGS": "-trimpath",
        })
        return sys.Machine().Command(ctx, args...)
    })

flags, err := command.Read(ctx, m, "go", "env", "GOFLAGS")
// flags == "-trimpath"
```

`command.MachineFunc` adapts any function into a Machine, the same way
`http.HandlerFunc` adapts functions into handlers.

## Files

`command.FS(m)` returns a `lesiw.io/fs.FS` for any machine. Machines
with native filesystem access (local, in-memory) use it; for the rest,
file operations are implemented with whatever commands the target
system has (`tee` on Unix, `Remove-Item` on Windows). Calling code is
identical either way, and automation written this way is
cross-platform by default: both libraries' CI suites run on Linux,
macOS, Windows, FreeBSD, and Alpine.

```go
fsys := command.FS(m)
err := fs.WriteFile(ctx, fsys, "hello.txt", []byte("Hello!\n"))
data, err := fs.ReadFile(ctx, fsys, "hello.txt")
```

`*command.Sh` exposes the same operations as methods: `ReadFile`,
`WriteFile`, `Open`, `Create`, `Append`, `Stat`, `Remove`, `RemoveAll`,
`Rename`, `MkdirAll`, `Chmod`, `Temp`, `Glob`, `ReadDir`, `Walk`, and
more.

Copying a file between machines is `io.Copy` of a lazy read buffer
into a lazy write buffer:

```go
local := command.Shell(sys.Machine())
remote := command.Shell(ssh.Machine(sys.Machine(), "ssh", "deploy@prod.example.com"))

_, err := io.Copy(
    remote.CreateBuffer(ctx, "/opt/app/server"),
    local.OpenBuffer(ctx, "bin/server"),
)
```

### Directories

A trailing slash means a directory, and directories stream as tar
archives:

```go
// Copy a local directory into a container.
dst, err := remote.Create(ctx, "/app/") // create/empty directory, accept tar
if err != nil {
    return err
}
defer dst.Close()
src, err := local.Open(ctx, "src/") // read directory as tar
if err != nil {
    return err
}
defer src.Close()
_, err = io.Copy(dst, src)
```

`Create("dir/")` replaces the directory's contents; `Append("dir/")`
adds to it; `Open("dir/")` reads it out. The same three verbs you
already use for files.

### File modes

Permissions travel on the context, where they apply to every operation
in the chain (including implicitly created parent directories):

```go
err := sh.WriteFile(
    fs.WithFileMode(ctx, 0o755),
    "hello.sh",
    []byte("#!/bin/sh\necho hello\n"),
)
```

### Temp files

```go
f, err := sh.Temp(ctx, "build")  // temp file; "build/" for a temp directory
if err != nil {
    return err
}
defer sh.RemoveAll(ctx, f.Path())
defer f.Close()
```

## Shell discipline

`command.Shell(machine, commands...)` registers the external commands
the automation is allowed to run. Keep the list to genuine external
tools:

```go
sh := command.Shell(sys.Machine(), "go", "git", "docker")
```

Do not register filesystem or text utilities — `cat`, `ls`, `mkdir`,
`rm`, `mv`, `chmod`, `stat`, `find`, `tar`, `mktemp`, `uname`, `grep`,
`tee`, `echo`. Each has a portable replacement (see the translation
table) that also works on machines where the command doesn't exist,
Windows included. A Shell that declares `"go", "git"` at the top of
the file is documentation: those are the automation's true
dependencies.

To register a command after construction, route it to the underlying
machine:

```go
sh = sh.Handle("kubectl", sh.Unshell())
```

## Testing

`mock.Machine` records calls and returns programmed responses.
Unprogrammed commands succeed with empty output, so tests only
specify what they care about. `mock.Calls` retrieves what ran,
piercing through a Shell:

```go
func Deploy(ctx context.Context, sh *command.Sh) error {
    branch, err := sh.Read(ctx, "git", "branch", "--show-current")
    if err != nil {
        return fmt.Errorf("read branch: %w", err)
    }
    return sh.Exec(ctx, "git", "push", "origin", branch)
}

func TestDeploy(t *testing.T) {
    m := new(mock.Machine)
    m.Return(strings.NewReader("main\n"), "git", "branch", "--show-current")

    sh := command.Shell(m, "git")
    if err := Deploy(t.Context(), sh); err != nil {
        t.Fatal(err)
    }

    got := mock.Calls(sh, "git")
    want := []mock.Call{
        {Args: []string{"git", "branch", "--show-current"}},
        {Args: []string{"git", "push", "origin", "main"}},
    }
    if !cmp.Equal(want, got) {
        t.Errorf("git calls mismatch (-want +got):\n%s", cmp.Diff(want, got))
    }
}
```

Testing notes:

- `command.Read` strips trailing newlines, so mock responses work with
  or without `\n`.
- More specific mocked commands take precedence over less specific
  ones: `m.Return(r, "git", "push")` beats `m.Return(r, "git")` for
  `git push` calls.
- To fail a command: `m.Return(command.Fail(&command.Error{Code: 1}),
  "exit", "1")`.
- Unprogrammed commands succeed with empty output by design (the same
  get-out-of-the-way semantics as Python's MagicMock). Do not stub
  commands just to make them succeed — `m.Return(strings.NewReader(""),
  "cmd")` is never needed. Stub only what the test asserts on.
- For a strict mock, set the default response: `Return` with no
  command arguments applies to every unprogrammed command. A
  zero-code failure composes with `command.NotFound`:
  `m.Return(command.Fail(&command.Error{Err: fmt.Errorf("unexpected")}))`.
- OS-dependent code: `m.SetOS("windows")` and `m.SetArch("amd64")` —
  never mock `uname`.
- `mock.Machine` includes an in-memory filesystem. Pre-populate it
  with `WriteFile` before the code under test runs; inspect it after.
- For package-level shells, swap them out for the test's duration:

```go
func swap[T any](t *testing.T, ptr *T, val T) {
    t.Helper()
    old := *ptr
    *ptr = val
    t.Cleanup(func() { *ptr = old })
}

var sh = command.Shell(sys.Machine(), "go")

func TestBuild(t *testing.T) {
    swap(t, &sh, command.Shell(new(mock.Machine), "go"))
    // ...
}
```

- For quick one-off machines, `command.MachineFunc` adapts a function;
  `mem.Machine()` provides working echo/cat/tee/tr over an in-memory
  filesystem for examples and Playground use.

## Common mistakes

Each of these compiles (or nearly compiles) and then fails at run time.

**Shelling out.** `sh.Read(ctx, "sh", "-c", "a | b")` reintroduces
quoting bugs and fails on any machine without `sh`, including Windows.
Piping is `io.Copy`; substitution is `command.Read`; there is no shell
in the vocabulary and no need for one.

**Probing with `which`.** `which` is absent on Windows and minimal
containers. Run the actual command and check
`command.NotFound(err)`.

**Forgetting `"ssh"` in `ssh.Machine`.** The arguments are the whole
command line: `ssh.Machine(sys.Machine(), "ssh", "user@host")`.
Passing only `"user@host"` tries to execute a program named
`user@host`.

**Registering utilities on a Shell.** If `cat`, `mkdir`, or `tar`
appear in a `command.Shell(...)` call, a portable method was missed.
The exception is when the utility is genuinely the tool under
automation (for example, `gzip` compressing a stream in the middle of
a pipeline).

**Expecting commands to run eagerly.** Creating a buffer runs nothing.
`command.NewReader(ctx, m, "reboot")` is inert until something reads
it. The helpers (`Read`, `Do`, `Exec`) both create and drive the
buffer, which is why most code uses them.

**Reaching for the `os` package.** `os.Getenv` reads the local
process's environment, not the machine's. Use `command.Env(ctx, m,
key)` and `command.WithEnv`. Same for `os.Getwd` versus
`fs.WorkDir(ctx)` / `fs.WithWorkDir`.

**Capturing with the wrong helper.** `Exec` streams to the terminal
and captures nothing. `Read` captures. `Do` discards. The choice
follows from where the output should go.

**Inventing API.** If something seems missing — a `NewStream`, a
`NotFound` field on `Error`, an eager `Run` — check
[pkg.go.dev/lesiw.io/command](https://pkg.go.dev/lesiw.io/command)
first. The vocabulary is small and deliberate; the function that
exists is usually simpler than the one being imagined.

## Quick reference

```go
// Execution
command.Read(ctx, m, args...) (string, error)   // capture output
command.Do(ctx, m, args...) error               // discard output
command.Exec(ctx, m, args...) error             // attach to terminal

// Buffers and piping
command.NewReader(ctx, m, args...) io.ReadCloser    // Close cancels
command.NewWriter(ctx, m, args...) io.WriteCloser   // Close waits
command.NewFilter(ctx, m, args...) io.ReadWriteCloser
command.Copy(dst, src, filters...) (int64, error)

// Environment and context
command.WithEnv(ctx, map[string]string) context.Context
command.Env(ctx, m, key) string
fs.WithWorkDir(ctx, dir) context.Context
fs.WithFileMode(ctx, mode) context.Context
fs.WithDirMode(ctx, mode) context.Context

// Machines
sys.Machine()                     // local
ssh.Machine(m, sshCmdline...)     // remote; args include "ssh" itself
ctr.Machine(m, image, args...)    // container; Shutdown to clean up
ctr.Ctl(m)                        // the container CLI itself
sub.Machine(m, prefix...)         // prefix every command
mem.Machine()                     // in-memory, Playground-safe
new(mock.Machine)                 // tests
command.MachineFunc(fn)           // function as Machine
command.Handle(m, name, handler)  // route one command name
command.HandleFunc(m, name, fn)
command.Shutdown(ctx, m)          // cleanup; safe on any machine
command.OS(ctx, m), command.Arch(ctx, m)

// Shell
command.Shell(m, commands...) *command.Sh
sh.Read / sh.Do / sh.Exec
sh.ReadFile / sh.WriteFile / sh.Open / sh.Create / sh.Append
sh.Stat / sh.Remove / sh.RemoveAll / sh.Rename / sh.MkdirAll
sh.Chmod / sh.Temp / sh.Glob / sh.ReadDir / sh.Walk
sh.OS / sh.Arch / sh.Env / sh.Handle / sh.Unshell

// Errors
command.NotFound(err) bool
*command.Error{Code int, Log []byte, Err error}

// Tracing (set the environment variable; see package docs)
// CMDTRACE=on   argv only    CMDTRACE=full   argv + environment

// Testing
m.Return(reader, args...)
mock.Calls(m, name) []mock.Call
m.SetOS(os), m.SetArch(arch)
command.Fail(err) command.Buffer
```

Full documentation: [pkg.go.dev/lesiw.io/command](https://pkg.go.dev/lesiw.io/command)
and [pkg.go.dev/lesiw.io/fs](https://pkg.go.dev/lesiw.io/fs).
