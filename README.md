# httpbackoff
<img hspace="20" align="left" src="https://tools.taskcluster.net/lib/assets/taskcluster-120.png" />
[![Build Status](https://travis-ci.org/taskcluster/httpbackoff.svg?branch=master)](https://travis-ci.org/taskcluster/httpbackoff)
[![GoDoc](https://godoc.org/github.com/taskcluster/httpbackoff?status.svg)](https://godoc.org/github.com/taskcluster/httpbackoff)
[![Coverage Status](https://coveralls.io/repos/taskcluster/httpbackoff/badge.svg?branch=master&service=github)](https://coveralls.io/github/taskcluster/httpbackoff?branch=master)
[![License](https://img.shields.io/badge/license-MPL%202.0-orange.svg)](http://mozilla.org/MPL/2.0)

Automatic http retries for intermittent failures, with exponential backoff,
based on https://github.com/cenkalti/backoff.

The reason for a separate library, is that this library handles http status
codes to know whether to retry or not.  HTTP codes in range 500-599 are
retried. Connection failures are also retried. Status codes 400-499 are
considered permanent errors and are not retried.

The Retry function performs the http request and retries if temporary errors
occur. It takes a single parameter as its input - a function to perform the
http request. This function must return `(resp *http.Response, tempError error,
permError error)` where `tempError` must be non-nil if a temporary error occurs
(e.g.  dropped connection), and `permError` must be non-nil if an error occurs
that does not warrant retrying the request (e.g. badly formed url).

For example, the following code that is not using retries:

```go
res, err := http.Get("http://www.google.com/robots.txt")
```

can be rewritten as:

```go
res, attempts, err := httpbackoff.New().Retry(func() (*http.Response, error, error) {
	resp, err := http.Get("http://www.google.com/robots.txt")
	// assume all errors are temporary
    return resp, err, nil
})
```

Please note the additional return value `attempts` is an `int` specifying how
many http calls were made (i.e. = 1 if no retries, otherwise > 1).

The go http package has 11 functions that return `(*http.Reponse, error)`:

* http://golang.org/pkg/net/http/#Client.Do
* http://golang.org/pkg/net/http/#Client.Get
* http://golang.org/pkg/net/http/#Client.Head
* http://golang.org/pkg/net/http/#Client.Post
* http://golang.org/pkg/net/http/#Client.PostForm
* http://golang.org/pkg/net/http/#Get
* http://golang.org/pkg/net/http/#Head
* http://golang.org/pkg/net/http/#Post
* http://golang.org/pkg/net/http/#PostForm
* http://golang.org/pkg/net/http/#ReadResponse
* http://golang.org/pkg/net/http/#Transport.RoundTrip

To simplify using these functions, 11 utility functions have been written that
wrap these. Therefore you can simplify this example above further with:

```go
res, _, err := httpbackoff.New().Get("http://www.google.com/robots.txt")
```

## Configuring backoff settings

Here is a simple example using custom backoff settings.

```go
package main

import (
	"log"
	"net/http/httputil"
	"time"

	"github.com/cenkalti/backoff"
	"github.com/taskcluster/httpbackoff"
)

func main() {
	customBackOff := &httpbackoff.ExponentialBackOff{
		InitialInterval:     1 * time.Millisecond,
		RandomizationFactor: 0.2,
		Multiplier:          1.2,
		MaxInterval:         5 * time.Millisecond,
		MaxElapsedTime:      20 * time.Millisecond,
		Clock:               backoff.SystemClock,
	}

	res, _, err := customBackOff.Get("http://www.google.com/robots.txt")
	if err != nil {
		log.Fatalf("%s", err)
	}
	data, err := httputil.DumpResponse(res, true)
	if err != nil {
		log.Fatalf("%s", err)
	}
	log.Print(string(data))
}
```

Please note these are global settings to the module, so you can't concurrently
use different backoff settings.

## Testing

The package has tests, which run in travis. See http://travis-ci.org/taskcluster/httpbackoff.

## Concurrency

As far as I am aware, there is nothing in this library that prevents it from
being used concurrently. Please let me know if you find any problems!

## Contributing
Contributions are welcome. Please fork, and issue a Pull Request back with an
explanation of your changes. Also please include tests for any functional
changes.
