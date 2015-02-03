[![Build Status](https://travis-ci.org/travcunn/go-workers.png)](https://travis-ci.org/travcunn/go-workers)
[![GoDoc](https://godoc.org/github.com/travcunn/go-workers?status.png)](https://godoc.org/github.com/travcunn/go-workers)

Queue system independent fork of [github.com/jrallison/go-workers](https://github.com/jrallison/go-workers).
The original fork of this project was tied to Sidekiq (Ruby async tasks). This fork allows you to push tasks to Redis from any programming language (Python, Ruby, GoLang, etc) using JSON.

Background workers in [golang](http://golang.org/).

* reliable queueing for all queues using [BLPOP](http://redis.io/commands/blpop)
* jobs parameters enqueued using JSON
* support custom middleware
* customize concurrency per queue
* responds to Unix signals to safely wait for jobs to finish before exiting.
* provides stats on what jobs are currently running
* well tested

##### Example usage:

```go
package main

import (
	"github.com/travcunn/go-workers"
)

func myJob(message *workers.Msg) {
  // do something with your message
  // message.OriginalJson() is a wrapper around go-simplejson (http://godoc.org/github.com/bitly/go-simplejson)
}

type myMiddleware struct{}

func (r *myMiddleware) Call(queue string, message *workers.Msg, next func() bool) (acknowledge bool) {
  // do something before each message is processed
  acknowledge = next()
  // do something after each message is processed
  return
} 

func main() {
  workers.Configure(map[string]string{
    // location of redis instance
    "server":  "localhost:6379",
    // instance of the database
    "database":  "0",
    // number of connections to keep open with redis
    "pool":    "30",
    // unique process id for this instance of workers (for proper recovery of inprogress jobs on crash)
    "process": "1",
  })

  workers.Middleware.Append(&myMiddleware{})

  // pull messages from "myqueue" with concurrency of 10
  workers.Process("myqueue", myJob, 10)

  // pull messages from "myqueue2" with concurrency of 20
  workers.Process("myqueue2", myJob, 20)

  // stats will be available at http://localhost:8080/stats
  go workers.StatsServer(8080)

  // Blocks until process is told to exit via unix signal
  workers.Run()
}
```

##### Handling Retries
This library does not provide the ability to retry tasks in case of failure. Instead, this responsibility is left up to the programmer. To retry a task, simply push the task onto the appropriate Redis queue. Be careful, since this may cause the task to run forever if the task repeatedly fails.
