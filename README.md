# StatsD Client (Go)

**statsd** is a simple yet powerful, futuristic [StatsD](https://github.com/etsy/statsd) client written in [Go](https://golang.org).

[![build status](https://img.shields.io/travis/netdata/statsd/master.svg?style=flat-square)](https://travis-ci.org/netdata/statsd) [![release](https://img.shields.io/badge/release%20-0.1-0077b3.svg?style=flat-square)](https://github.com/netdata/statsd/releases)

## Features

* Supports *Counting*, *Sampling*, *Timing*, *Gauges* and *Sets* out of the box
* Futuristic and Extendable: Ability to send custom metric values and types, yet unknown to the current client
* It is blazing fast and it does not allocate unnecessary memory. Metrics are sent based on a customized packet size, manual `Flushing` of buffered metrics is also an option
* Beautiful and easy to learn API
* Easy for testing, the `NewClient` function accepts an `io.WriteCloser`, so you use it to test the sent metrics of your application on a development environment 
* Protocol-unawareness.

## Installation

The only requirement is the [Go Programming Language](https://golang.org/dl/)

```sh
$ go get -u github.com/netdata/statsd
```

## Quick start

### API

```go
// NewClient returns a new StatsD client.
// The first input argument, "writeCloser", should be a value which completes the
// `io.WriteCloser` interface
// It can be a UDP connection or a string buffer or even the stdout for testing.
// The second input argument, "prefix" can be empty
// but it is usually the app's name and a single dot.
NewClient(writeCloser io.WriteCloser, prefix string) *Client
```

```go
Client {
    SetMaxPackageSize(maxPacketSize int)
    SetFormatter(fmt func(metricName string) string)
    FlushEvery(dur time.Duration)

    IsClosed() bool
    Close() error

    WriteMetric(metricName, value, typ string, rate float32) error
    Flush(n int) error

    Count(metricName string, value int) error
    Increment(metricName string) error

    Gauge(metricName string, value int) error

    Unique(metricName string, value int) error

    Time(metricName string, value time.Duration) error
    Record(metricName string, rate float32) func() error
}
```

#### Metric Value helpers

```go
Duration(v time.Duration) string
Int(v int) string
Int8(v int8) string
Int16(v int16) string
Int32(v int32) string
Int64(v int64) string
Uint(v uint) string
Uint8(v uint8) string
Uint16(v uint16) string
Uint32(v uint32) string
Uint64(v uint64) string
Float32(v float32) string
Float64(v float64) string
```

#### Metric Type constants

```go
const (
    Count  string = "c"
    Gauge  string = "g"
    Unique string = "s"
    // Set is an alias for "Unique"
    Set         = Unique
    Time string = "ms"
)
```

> Read more at: https://github.com/etsy/statsd/blob/master/docs/metric_types.md

### Example

Assuming that you have a [statsd server](https://github.com/etsy/statsd) running at `:8125` (default port).

```sh
# assume the following codes in example.go file
$ cat example.go
```

```go
package main

import (
	"fmt"
	"net/http"
	"strings"
	"time"

	"github.com/netdata/statsd"
)


// statusCodeReporter is a compatible `http.ResponseWriter`
// which stores the `statusCode` for further reporting.
type statusCodeReporter struct {
    http.ResponseWriter
    written    bool
    statusCode int
}

func (w *statusCodeReporter) WriteHeader(statusCode int) {
    if w.written {
        return
    }

    w.statusCode = statusCode
    w.ResponseWriter.WriteHeader(statusCode)
}

func (w *statusCodeReporter) Write(b []byte) (int, error) {
    w.written = true
    return w.ResponseWriter.Write(b)
}

func main() {
    statsWriter, err := statsd.UDP(":8125")
    if err != nil {
        panic(err)
    }

    statsD := statsd.NewClient(statsWriter, "prefix.")
    statsD.FlushEvery(5 * time.Second)

    statsDMiddleware := func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            path := r.URL.Path
            if len(path) == 1 {
                path = "index" // for root.
            } else if path == "/favicon.ico" {
                next.ServeHTTP(w, r)
                return
            } else {
                path = path[1:]
                path = strings.Replace(path, "/", ".", -1)
            }

            // HERE
            statsD.Increment(fmt.Sprintf("%s.request", path))
            
            newResponseWriter := &statusCodeReporter{ResponseWriter: w, statusCode: http.StatusOK}

            // HERE
            stop := statsD.Record(fmt.Sprintf("%s.time", path), 1)
            next.ServeHTTP(newResponseWriter, r)
            stop()

            // HERE
            statsD.Increment(fmt.Sprintf("%s.response.%d", path, newResponseWriter.statusCode))
        })
    }

    mux := http.DefaultServeMux

    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "Hello from index")
    })

    mux.HandleFunc("/other", func(w http.ResponseWriter, r *http.Request) {
        // HERE
        statsD.Unique("other.unique", 1)

        fmt.Fprintln(w, "Hello from other page")
    })

    http.ListenAndServe(":8080", statsDMiddleware(mux))
}
```

```
# run example.go and visit http://localhost:8080/other
$ go run example.go
```

Navigate [here](_examples) to discover all available examples.

## License

StatsD Go Client is licensed under the GNU GENERAL PUBLIC [License](LICENSE).