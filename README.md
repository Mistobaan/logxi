
![demo](https://github.com/mgutz/logxi/raw/master/images/demo.gif)

# logxi

log XI is a structured [12-factor app](http://12factor.net/logs)
logger built for speed and happy development.

*   Simpler. Sane no-configuration defaults out of the box.
*   Faster. See benchmarks vs logrus and log15.
*   Structured. Key-value pairs are enforced. Logs JSON in production.
*   Configurable. Enable/disalbe Loggers and levels via env vars.
*   Friendlier. Happy, colorful and developer friendly logger in terminal.
*   Helpul. Traces, warnings and errors are emphasized with file, line
    number and callstack.
*   Efficient. Has level guards to avoid cost of building complex arguments.

### Installation

    go get -u github.com/mgutz/logxi/v1

### Getting Started

```go
import "github.com/mgutz/logxi/v1"

// create package variable for Logger interface
var logger log.Logger

func main() {
    // use default logger
    who := "mario"
    log.Info("Hello", "who", who)

    // create a logger with a unique identifier which
    // can be enabled from environment variables
    logger = log.New("pkg")

    // specify a writer
    modelLogger = log.NewLogger(os.Stdout, "models")

    db, err := sql.Open("postgres", "dbname=testdb")
    if err != nil {
        modelLogger.Error("Could not open database", "err", err)
    }

    fruit := "apple"
    languages := []string["go", "javascript"]
    if log.IsDebug() {
        // use key-value pairs after message
        logger.Debug("OK", "fruit", fruit, "languages", languages)
    }
}
```

logxi defaults to showing warnings and above. To view all logs

    LOGXI=* go run main.go

## Higlights

This logger package

*   Is fast in production environment

    A logger should be efficient and minimize performance tax.
    logxi encodes JSON 2X faster than logrus and log15 with primitive types.
    When diagnosing a problem in production, troubleshooting often means
    enabling small trace data in `Debug` and `Info` statements for some
    period of time.

        # primitive types
        BenchmarkLogxi         100000     18139 ns/op    2438 B/op     64 allocs/op
        BenchmarkLogrus         30000     43845 ns/op    8696 B/op    180 allocs/op
        BenchmarkLog15          30000     57332 ns/op    8755 B/op    220 allocs/op

        # nested object
        BenchmarkLogxiComplex   30000     43242 ns/op    6356 B/op    188 allocs/op
        BenchmarkLogrusComplex  30000     59905 ns/op   11878 B/op    262 allocs/op
        BenchmarkLog15Complex   20000     87828 ns/op   12268 B/op    294 allocs/op

*   Is developer friendly in the terminal. The HappyDevFormatter
    is colorful, prints file and line numbers for traces, warnings
    and errors. Prints arguments in the order they are coded.
    Errors print the call stack.

    `HappyDevFormatter` is not too concerned with performance
    and delegates to JSONFormatter internally.

*   Logs machine parsable output in production environments.
    The default formatter for non terminals is `JSONFormatter`.

    `TextFormatter` may also be used which is MUCH faster than
    JSON but there is no guarantee it can be easily parsed.

*   Has level guards to avoid the cost of building arguments. Get in the
    habit of using guards.

        if log.IsDebug() {
            log.Debug("some ", "key1", expensive())
        }

*   Conforms to a logging interface so it can be replaced.

        type Logger interface {
            Trace(msg string, args ...interface{})
            Debug(msg string, args ...interface{})
            Info(msg string, args ...interface{})
            Warn(msg string, args ...interface{})
            Error(msg string, args ...interface{})
            Fatal(msg string, args ...interface{})
            Log(level int, msg string, args []interface{})

            SetLevel(int)
            IsTrace() bool
            IsDebug() bool
            IsInfo() bool
            IsWarn() bool
            // Error, Fatal not needed, those SHOULD always be logged
        }

*   Standardizes on key-value pair argument sequence

    ```go
log.Debug("inside Fn()", "key1", value1, "key2", value2)

// instead of this
log.WithFields(logrus.Fields{"m": "pkg", "key1": value1, "key2": value2}).Debug("inside fn()")
```
    logxi logs `FIX_IMBALANCED_PAIRS =>` if key-value pairs are imbalanced

*   Supports Color Schemes (256 colors)

    `log.New` creates a logger that supports color schemes

        logger := log.New("mylog")

    To customize scheme

        # emphasize errors with white text on red background
        LOGXI_COLORS="ERR=white:red" yourapp

        # emphasize errors with pink = 200 on 256 colors table
        LOGXI_COLORS="ERR=200" yourapp

## Configuration

### Enabling/Disabling Loggers

By default logxi logs entries whose level is `LevelWarn` or above when
using a terminal. For non-terminals, entries with level `LevelError` and
above are logged.

To quickly see all entries use short form

    # enable all, disable log named foo
    LOGXI=*,-foo yourapp

To better control logs in production, use long form which allows
for granular control of levels

    # the above statement is equivalent to this
    LOGXI=*=DBG,foo=OFF yourapp

`DBG` should obviously not be used in production unless for
troubleshooting. See `LevelAtoi` in `logger.go` for values.
For example, there is a problem in the data access layer
in production.

    # Set all to Error and set data related packages to Debug
    LOGXI=*=ERR,models=DBG,dat*=DBG,api=DBG yourapp

### Format

The format may be set via `LOGXI_FORMAT` environment
variable. Valid values are `"happy", "text", "JSON"`

    # Use JSON in production with custom time
    LOGXI_FORMAT=JSON,t=2006-01-02T15:04:05.000000-0700 yourapp

The "happy" formatter has more options

*   pretty - puts each key-value pair indented on its own line

    "happy" default to fitting key-value pair onto the same line. If
    result characters are longer than `maxcol` then the pair will be
    put on the next line and indented

*   maxcol - maximum number of columns before forcing a key to be on its
    own line. If you want everything on a single line, set this to high
    value like 1000. Default is 80.

*   context - the number of context lines to print on source. Set to -1
    to see only file:lineno. Default is 2.

### Color Schemes

The color scheme may be set with `LOGXI_COLORS` environment variable. For
example, the default dark scheme is emulated like this

    # on non-Windows, see Windows support below
    export LOGXI_COLORS=key=cyan+h,value,misc=blue+h,source=magenta,TRC,DBG,WRN=yellow,INF=green,ERR=red+h
    yourapp

    # color only errors
    LOGXI_COLORS=ERR=red yourapp

See [ansi](http://github.com/mgutz/ansi) package for styling. An empty
value, like "value" and "DBG" above means use default foreground and
background on terminal.

Keys

*   \*  - default color
*   TRC - trace color
*   DBG - debug color
*   WRN - warn color
*   INF - info color
*   ERR - error color
*   key - key color
*   value - value color unless WRN or ERR
*   misc - time and log name color
*   source - source context color (excluding error line)

#### Windows

Use [ConEmu-Maximus5](https://github.com/Maximus5/ConEmu).
Read this page about [256 colors](https://code.google.com/p/conemu-maximus5/wiki/Xterm256Colors).

Colors in PowerShell and Command Prompt _work_ but not very pretty.

## Extending

What about hooks? There are least two ways to do this

*   Implement your own `io.Writer` to write to external services. Be sure to set
    the formatter to JSON to faciliate decoding with Go's built-in streaming
    decoder.
*   Create an external filter. See `v1/cmd/filter` as an example.

What about log rotation? 12 factor apps only concern themselves with
STDOUT. Use shell redirection operators to write to a file.

There are many utilities to rotate logs which accept STDIN as input. They can
do many things like send alerts, etc. The two obvious choices are Apache's `rotatelogs`
utility and `lograte`.

```sh
yourapp | rotatelogs yourapp 86400
```

## Testing

```
# install godo task runner
go get -u gopkg.in/godo.v1/cmd/godo

# install dependencies
godo install -v

# run test
godo test

# run bench with allocs (requires manual cleanup)
godo allocs
```

## License

MIT License
