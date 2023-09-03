> Channels of channels
One of the most important properties of Go is that a channel is a first-class value that can be allocated and passed around like any other. A common use of this property is to implement safe, parallel demultiplexing.
> In the example in the previous section, handle was an idealized handler for a request but we didn't define the type it was handling. If that type includes a channel on which to reply, each client can provide its own path for the answer. Here's a schematic definition of type Request.

```
type Request struct {
    args        []int
    f           func([]int) int
    resultChan  chan int
}
```

> The client provides a function and its arguments, as well as a channel inside the request object on which to receive the answer.

```
func sum(a []int) (s int) {
    for _, v := range a {
        s += v
    }
    return
}

request := &Request{[]int{3, 4, 5}, sum, make(chan int)}
// Send request
clientRequests <- request
// Wait for response.
fmt.Printf("answer: %d\n", <-request.resultChan)
```

> On the server side, the handler function is the only thing that changes.

```
func handle(queue chan *Request) {
    for req := range queue {
        req.resultChan <- req.f(req.args)
    }
}
```

> There's clearly a lot more to do to make it realistic, but this code is a framework for a rate-limited, parallel, non-blocking RPC system, and there's not a mutex in sight.
> Explain this in detail with walkthrough and example

---

The concept of "channels of channels" in Go allows for interesting and flexible designs, especially for message passing or remote procedure calls (RPCs). The example you provided demonstrates a basic yet powerful idea: you can include a channel within a struct, which then gets sent over another channel to a receiver, thus allowing the sender to specify exactly where the response should be sent.

Here's the breakdown of each component:

### 1. The `Request` Struct
Firstly, the `Request` struct contains three fields:
- `args`: Arguments to the function `f` that needs to be calculated.
- `f`: The function itself which takes `args` and returns an integer.
- `resultChan`: A channel through which the result of invoking `f` with `args` will be sent back.

```go
type Request struct {
    args        []int
    f           func([]int) int
    resultChan  chan int
}
```

### 2. The Client Side
The client wants to execute the function `sum` on the slice `[3, 4, 5]`. It wraps this request into a `Request` struct and includes a channel (`resultChan`) where the result should be sent. 

```go
func sum(a []int) (s int) {
    for _, v := range a {
        s += v
    }
    return
}

request := &Request{[]int{3, 4, 5}, sum, make(chan int)}
clientRequests <- request
fmt.Printf("answer: %d\n", <-request.resultChan)
```

The client then sends this `Request` struct over the `clientRequests` channel. It then waits on `request.resultChan` for the result to come back.

### 3. The Server Side
On the server side, there's a handler listening on the `clientRequests` channel. When it receives a `Request`, it invokes the function `req.f` with the arguments `req.args` and sends the result back through `req.resultChan`.

```go
func handle(queue chan *Request) {
    for req := range queue {
        req.resultChan <- req.f(req.args)
    }
}
```

This demonstrates the concept of a "channel of channels". You're passing a `Request` struct that includes its own channel (`resultChan`) over another channel (`clientRequests`). When the server receives the `Request`, it knows exactly where to send the response, because the `Request` itself contains that information.

### Rate-Limited, Parallel, Non-blocking RPC
Although the example is minimalistic, you could easily extend it into a rate-limited, parallel, non-blocking RPC system, as the text describes. Rate-limiting can be added by controlling how many instances of `handle()` are running concurrently. Parallelism is inherent because each request can be processed by a different `handle()` goroutine. It's non-blocking because channels are used for both the requests and the responses.

By designing it this way, you avoid the need for mutexes or other complex synchronization primitives, making the system easier to reason about.
