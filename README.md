


# liteq

> Lightweight Portable Message Queue Using SQLite

<!-- badges: start -->
[![R-CMD-check](https://github.com/r-lib/liteq/actions/workflows/R-CMD-check.yaml/badge.svg)](https://github.com/r-lib/liteq/actions/workflows/R-CMD-check.yaml)
[![](http://www.r-pkg.org/badges/version/liteq)](http://www.r-pkg.org/pkg/liteq)
[![CRAN RStudio mirror downloads](http://cranlogs.r-pkg.org/badges/liteq)](http://www.r-pkg.org/pkg/liteq)
[![Codecov test coverage](https://codecov.io/gh/r-lib/liteq/branch/main/graph/badge.svg)](https://app.codecov.io/gh/r-lib/liteq?branch=main)
<!-- badges: end -->

Temporary and permanent message queues for R. Built on top of SQLite
databases. 'SQLite' provides locking, and makes it possible to detect
crashed consumers. Crashed jobs can be automatically marked as "failed",
or put back in the queue again, potentially a limited number of times.

## Installation

Stable version:

```r
install.packages("liteq")
```

Development versiot:

```r
pak::pak("r-lib/liteq")
```

## Introduction

`liteq` implements a serverless message queue system in R.
It can handle multiple databases, and each database can contain
multiple queues.

`liteq` uses SQLite to store a database of queues, and uses other,
temporary SQLites databases for locking, and finding crashed workers
(see below).

## Usage

### Basic usage


```r
library(liteq)
```

In the following we create a queue in a temporary queue database.
The database will be removed if the R session quits.


```r
db <- tempfile()
q <- ensure_queue("jobs", db = db)
q
```

```
#> liteq queue 'jobs'
```

```r
list_queues(db)
```

```
#> [[1]]
#> liteq queue 'jobs'
```

Note that `ensure_queue()` is idempotent, if you call it again on the same
database, it will return the queue that was created previously. So it is
safe to call it multiple times, even from multiple processes. In case of
multiple processes, the locking mechanism eliminates race conditions.

To publish a message in the queue, call `publish()` on the queue object:


```r
publish(q, title = "First message", message = "Hello world!")
publish(q, title = "Second message", message = "Hello again!")
list_messages(q)
```

```
#>   id          title status
#> 1  1  First message  READY
#> 2  2 Second message  READY
```

A `liteq` message has a title, which is a string scalar, and the message
body itself is a string scalar as well. To use more complex data types in
messages, you need to serialize them using the `serialize()` function (set
`ascii` to `TRUE`!), or convert them to JSON with the `jsonlite` package.

Two functions are available to consume a message from a queue.
`try_consume()` returns immediately, either with a message (`liteq_message`
object), or `NULL` if the queue is empty. The `consume()` function blocks
if the queue is empty, and waits until a message appears in it.


```r
msg <- try_consume(q)
msg
```

```
#> liteq message from queue 'jobs':
#>   First message (12 B)
```

The title and the message body are available as fields of the message
object:


```r
msg$title
```

```
#> [1] "First message"
```

```r
msg$message
```

```
#> [1] "Hello world!"
```

When a consumer is done processing a message it must call `ack()` on the
message object, to notify the queue that it is safe to remove the message.
If the consumer fails to process a message, it can call `nack()` (negative
ackowledgement) on the message object. Then the status of the message will
be set to `"FAILED"`. Failed messages can be removed from the queue, or
put back in the queue again, depending on the application.


```r
ack(msg)
list_messages(q)
```

```
#>   id          title status
#> 1  2 Second message  READY
```

```r
msg2 <- try_consume(q)
nack(msg2)
list_messages(q)
```

```
#>   id          title status
#> 1  2 Second message FAILED
```

The queue is empty now, so `try_consume()` returns `NULL`:


```r
try_consume(q)
```

```
#> NULL
```

### Crashed workers

If a worker crashes without calling either `ack()` or `nack()` on a message,
then this messages will be put back in the queue the next time a message is
requested from the queue.

To make this possible, each delivered message keeps an open connection to
a lock file, and crashed workers are found by the absense of this open
connection. In R basically means that the worker is considered as crashed
if the R process has no reference to the message object.

Note, that this also means that having many workers at the same time means
that it is possible to reach the maximum number of open connections by
R or the operating system.

## License

MIT © Gábor Csárdi
