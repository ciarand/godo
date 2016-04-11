This project is deprecated and unmaintained. Proceed with caution!

# godo

[godoc](https://godoc.org/gopkg.in/godo.v1)

godo is a task runner and file watcher for golang in the spirit of
rake, gulp.

To install

    go get -u gopkg.in/godo.v1/cmd/godo

## Godofile

**godo** runs either `Gododir/Godofile.go` or `tasks/Godofile.go`.

As an example, create a file **Gododir/Godofile.go** with this content

```go
package main

import (
	"fmt"
    . "gopkg.in/godo.v1"
)

func tasks(p *Project) {
    Env = "GOPATH=.vendor::$GOPATH PG_PASSWORD=dev"

    p.Task("default", D{"hello", "build"})

    p.Task("hello", func(c *Context) {
        name := c.Args.ZeroString("name", "n")
        if name == "" {
            Bash("echo Hello $USER!")
        } else {
            fmt.Println("Hello", name)
        }
    })

    p.Task("build", func() {
        Run("GOOS=linux GOARCH=amd64 go build", In{"cmd/server"})
    }).Watch("**/*.go")

    p.Task("views", func() {
        Run("razor templates")
    }).Watch("templates/**/*.go.html")

    p.Task("server", D{"views"}, func() {
        // rebuilds and restarts the process when a watched file changes
        Start("main.go", In{"cmd/server"})
    }).Watch("server/**/*.go", "cmd/server/*.{go,json}").
       Debounce(3000)
}

func main() {
    Godo(tasks)
}
```

To run "server" task from parent dir of `tasks/`

    godo server

To rerun "server" and its dependencies whenever any `*.go.html`,  `*.go` or `*.json` file changes

    godo server --watch

To run the "default" task which runs "hello" and "views"

    godo

Task names may add a "?" suffix to execute only once even when watching

```go
// build once regardless of number of dependents
p.Task("build?", func() {})
```

Task options

    D{} or Dependencies{} - dependent tasks which run before task
    W{} or Watches{}      - array of glob file patterns to watch

        /**/   - match zero or more directories
        {a,b}  - match a or b, no spaces
        *      - match any non-separator char
        ?      - match a single non-separator char
        **/    - match any directory, start of pattern only
        /**    - match any in this directory, end of pattern only
        !      - removes files from result set, start of pattern only

Task handlers

    func() {}           - Simple function handler
    func(c *Context) {} - Handler which accepts the current context

### Task Arguments

Task arguments follow POSIX style flag convention
(unlike go's built-in flag package). Any command line arguments
succeeding `--` are passed to each task. Note, arguments before `--`
are reserved for `godo`.

As an example,

```go
p.Task("hello", func(c *Context) {
    // "(none)" is the default value
    name := c.Args.MayString("(none)", "name", "n")
    fmt.Println("Hello", name)
})
```

running

```sh
# prints "Hello (none)"
godo hello

# prints "Hello dude" using POSIX style flags
godo hello -- --name=dude
godo hello -- -n dude
```

Args functions are categorized as

*  `Must*` - Argument must be set by user or panic.

    ```go
c.Args.MustInt("number", "n")
```

*  `May*` - If argument is not set, default to first value.

    ```go
    // defaults to 100
    c.Args.MayInt(100, "number", "n")
```

*  `Zero*` - If argument is not set, default to zero value.

    ```go
// defaults to 0
c.Args.ZeroInt("number", "n")
```

## Exec functions

### Bash

Bash functions uses the bash executable and may not run on all OS.

Run a bash script string. The script can be multine line with continutation.

```go
Bash(`
    echo -n $USER
    echo some really long \
        command
`)
```

Run a bash script and capture STDOUT and STDERR.

```go
output, err := BashOutput(`echo -n $USER`)
```

### Run

Run `go build` inside of cmd/app and set environment variables.

```go
Run(`GOOS=linux GOARCH=amd64 go build`, In{"cmd/app"})
```

Run and capture STDOUT and STDERR

```go
output, err := RunOutput("whoami")
```

### Start

Start an async command. If the executable has suffix ".go" then it will be "go install"ed then executed.
Use this for watching a server task.

```go
Start("main.go", In{"cmd/app"})
```

Godo tracks the process ID of started processes to restart the app gracefully.

### Inside

To run many commands inside a directory, use `Inside` instead of the `In` option.
`Inside` changes the working directory.

```go
Inside("somedir", func() {
    Run("...")
    Bash("...")
})
```

## Godofile Run-Time Environment

To specify whether to inherit from parent's process environment,
set `InheritParentEnv`. This setting defaults to true

```go
InheritParentEnv = false
```

To specify the base environment for your tasks, set `Env`.
Separate with whitespace or newlines.

```go
Env = `
    GOPATH=.vendor::$GOPATH
    PG_USER=mario
`
```

TIP: Set the `Env` when using a dependency manager like `godep`

```go
wd, _ := os.Getwd()
ws := path.Join(wd, "Godeps/_workspace")
Env = fmt.Sprintf("GOPATH=%s::$GOPATH", ws)
```
Functions can add or override environment variables as part of the command string.
Note that environment variables are set before the executable similar to a shell;
however, the `Run` and `Start` functions do not use a shell.

```go
p.Task("build", func() {
    Run("GOOS=linux GOARCH=amd64 go build" )
})
```

The effective environment for exec functions is: `parent (if inherited) <- Env <- func parsed env`

Paths should use `::` as a cross-platform path list separator. On Windows `::` is replaced with `;`.
On Mac and linux `::` is replaced with `:`.
