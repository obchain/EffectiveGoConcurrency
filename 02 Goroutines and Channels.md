> Goroutines - 
> They're called goroutines because the existing terms—threads, coroutines, processes, and so on—convey inaccurate connotations. A goroutine has a simple model: it is a function executing concurrently with other goroutines in the same address space. It is lightweight, costing little more than the allocation of stack space. And the stacks start small, so they are cheap, and grow by allocating (and freeing) heap storage as required.

> Goroutines are multiplexed onto multiple OS threads so if one should block, such as while waiting for I/O, others continue to run. Their design hides many of the complexities of thread creation and management.

> Prefix a function or method call with the go keyword to run the call in a new goroutine. When the call completes, the goroutine exits, silently. (The effect is similar to the Unix shell's & notation for running a command in the background.)

### Goroutines: Overview and Multiplexing

In Go, concurrency is primarily achieved through goroutines, which are lightweight threads of execution managed by the Go runtime rather than the underlying operating system. Goroutines are "multiplexed" onto multiple OS threads, which means that multiple goroutines can run concurrently on a smaller number of operating system threads.

### Why is this Beneficial?

1. **Lower Overhead**: Goroutines consume less memory and resources compared to traditional OS threads, making it possible to spawn thousands or even millions of them without running into issues.
  
2. **Blocking Isn't Problematic**: In languages with traditional threading models, a blocked thread (e.g., waiting for I/O) can be a significant issue that reduces overall system performance. In contrast, Go's runtime can shift other goroutines onto available threads when one goroutine gets blocked. This ensures that CPU time is used efficiently.

3. **Simpler for Developers**: The Go runtime takes care of managing these details, so you don't have to explicitly create and manage threads, which often involves dealing with complex synchronization primitives.

### Example: I/O Blocking with Goroutines

To illustrate how this works, consider an example where you have three tasks to accomplish:

1. Download a file from the internet (an I/O-bound operation).
2. Compute the factorial of a large number (a CPU-bound operation).
3. Write data to a disk (another I/O-bound operation).

Here's some simple pseudo-Go code to demonstrate how you might handle these with goroutines:

```go
func downloadFile(url string) {
    // Simulate a file download
}

func computeFactorial(n int) int {
    // Compute factorial
    return result
}

func writeDataToDisk(data []byte) {
    // Simulate writing data to disk
}

func main() {
    go downloadFile("http://example.com/large-file")
    go computeFactorial(100000)
    go writeDataToDisk([]byte("some data"))

    // Wait for all goroutines to complete
    // (In real code, you'd probably use a sync.WaitGroup or some other
    // synchronization primitive)
    time.Sleep(time.Second * 10)
}
```

In this example, each of the functions runs in its own goroutine. This means that they can run concurrently, but more importantly, if `downloadFile` is waiting for data to arrive over the network, that won't stop `computeFactorial` or `writeDataToDisk` from making progress.

Because the Go runtime can move goroutines around between underlying OS threads, it can ensure that all the CPU cores are kept busy, even if some goroutines are blocked waiting for I/O.

### Summary

In traditional languages, you'd have to manage threads explicitly and deal with the complexities of different threads blocking on I/O. In Go, goroutines and the runtime simplify this considerably: you launch the concurrent tasks, and if one blocks, the Go runtime rearranges the others to keep the system busy. This makes it easier to write efficient, concurrent programs.

---

> Channels

> Like maps, channels are allocated with make, and the resulting value acts as a reference to an underlying data structure. If an optional integer parameter is provided, it sets the buffer size for the channel. The default is zero, for an unbuffered or synchronous channel.

```
ci := make(chan int)            // unbuffered channel of integers
cj := make(chan int, 0)         // unbuffered channel of integers
cs := make(chan *os.File, 100)  // buffered channel of pointers to Files
```

> Unbuffered channels combine communication—the exchange of a value—with synchronization—guaranteeing that two calculations (goroutines) are in a known state.

> There are lots of nice idioms using channels. Here's one to get us started. In the previous section we launched a sort in the background. A channel can allow the launching goroutine to wait for the sort to complete.

```
c := make(chan int)  // Allocate a channel.
// Start the sort in a goroutine; when it completes, signal on the channel.
go func() {
    list.Sort()
    c <- 1  // Send a signal; value does not matter.
}()
doSomethingForAWhile()
<-c   // Wait for sort to finish; discard sent value.
```

> Receivers always block until there is data to receive. If the channel is unbuffered, the sender blocks until the receiver has received the value. If the channel has a buffer, the sender blocks only until the value has been copied to the buffer; if the buffer is full, this means waiting until some receiver has retrieved a value.

> A buffered channel can be used like a semaphore, for instance to limit throughput. In this example, incoming requests are passed to handle, which sends a value into the channel, processes the request, and then receives a value from the channel to ready the “semaphore” for the next consumer. The capacity of the channel buffer limits the number of simultaneous calls to process.

```
var sem = make(chan int, MaxOutstanding)

func handle(r *Request) {
    sem <- 1    // Wait for active queue to drain.
    process(r)  // May take a long time.
    <-sem       // Done; enable next request to run.
}

func Serve(queue chan *Request) {
    for {
        req := <-queue
        go handle(req)  // Don't wait for handle to finish.
    }
}
```

> Once MaxOutstanding handlers are executing process, any more will block trying to send into the filled channel buffer, until one of the existing handlers finishes and receives from the buffer.

> This design has a problem, though: Serve creates a new goroutine for every incoming request, even though only MaxOutstanding of them can run at any moment. As a result, the program can consume unlimited resources if the requests come in too fast. We can address that deficiency by changing Serve to gate the creation of the goroutines. Here's an obvious solution, but beware it has a bug we'll fix subsequently:

```
func Serve(queue chan *Request) {
    for req := range queue {
        sem <- 1
        go func() {
            process(req) // Buggy; see explanation below.
            <-sem
        }()
    }
}
```

> The bug is that in a Go for loop, the loop variable is reused for each iteration, so the req variable is shared across all goroutines. That's not what we want. We need to make sure that req is unique for each goroutine. Here's one way to do that, passing the value of req as an argument to the closure in the goroutine:

```
func Serve(queue chan *Request) {
    for req := range queue {
        sem <- 1
        go func(req *Request) {
            process(req)
            <-sem
        }(req)
    }
}
```

> Compare this version with the previous to see the difference in how the closure is declared and run. Another solution is just to create a new variable with the same name, as in this example:

```
func Serve(queue chan *Request) {
    for req := range queue {
        req := req // Create new instance of req for the goroutine.
        sem <- 1
        go func() {
            process(req)
            <-sem
        }()
    }
}
```

> It may seem odd to write

`req := req`

> but it's legal and idiomatic in Go to do this. You get a fresh version of the variable with the same name, deliberately shadowing the loop variable locally but unique to each goroutine.

> Going back to the general problem of writing the server, another approach that manages resources well is to start a fixed number of handle goroutines all reading from the request channel. The number of goroutines limits the number of simultaneous calls to process. This Serve function also accepts a channel on which it will be told to exit; after launching the goroutines it blocks receiving from that channel.

```
func handle(queue chan *Request) {
    for r := range queue {
        process(r)
    }
}

func Serve(clientRequests chan *Request, quit chan bool) {
    // Start handlers
    for i := 0; i < MaxOutstanding; i++ {
        go handle(clientRequests)
    }
    <-quit  // Wait to be told to exit.
}
```

## Explainer
Let's break it down:

### Using Buffered Channels as Semaphores

In this example, a buffered channel named `sem` is being used as a semaphore to limit the number of goroutines executing the `process(r)` function concurrently. The size of the buffer, `MaxOutstanding`, defines the maximum number of goroutines that can be executing `process(r)` at any given time.

When `sem <- 1` is executed, it tries to send a value into the channel. If the channel is full, this will block until there's room, effectively limiting the number of concurrent executions of the following code (`process(r)` in this case).

After `process(r)` is done, `<-sem` is executed, which receives a value from the channel, making room for another value to be sent into the channel. This is equivalent to releasing the semaphore so that another goroutine can proceed.

### Problem with Creating Unbounded Number of Goroutines

The initial `Serve` function starts a new goroutine for every incoming request with `go handle(req)`. This isn't efficient because you can potentially create an infinite number of goroutines if requests come in too quickly, each taking up system resources.

### Solving the Problem with Gating Goroutine Creation

The revised version of the `Serve` function gates the creation of new goroutines. Before starting a new goroutine, it waits until there is room in the semaphore by sending a value into the `sem` channel with `sem <- 1`.

### Bug in Loop Variable Closure

The initial "buggy" version demonstrates a common pitfall in Go where the loop variable (`req` in this case) is captured by the goroutine, and because goroutines execute concurrently, they might all end up using the last value of `req`.

The "loop variable capture" problem occurs when you use the loop variable inside a goroutine that you launch within the loop. Because of the way Go's closures work, all the goroutines will capture the same variable, which will end up being set to its final value.

Let's use a simple example to demonstrate this issue. Imagine you want to start 3 goroutines that print the numbers 0, 1, and 2.

### Incorrect Code (with Bug)

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup

	for i := 0; i < 3; i++ {
		wg.Add(1)
		go func() {
			fmt.Println(i)  // This prints the value of 'i'
			wg.Done()
		}()
	}
	
	wg.Wait()
}
```

You might expect the code to print:

```
0
1
2
```

But it could actually print something like:

```
3
3
3
```

All three goroutines might end up printing "3" (or sometimes "2" or other numbers based on the timing of the goroutines) because they all reference the same `i` variable, which reaches the value of 3 when the loop terminates.

### Corrected Code

#### Method 1: Passing as an argument

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup

	for i := 0; i < 3; i++ {
		wg.Add(1)
		go func(i int) {
			fmt.Println(i)
			wg.Done()
		}(i)
	}
	
	wg.Wait()
}
```

#### Method 2: Shadowing the loop variable

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup

	for i := 0; i < 3; i++ {
		i := i // Shadow the loop variable
		wg.Add(1)
		go func() {
			fmt.Println(i)
			wg.Done()
		}()
	}
	
	wg.Wait()
}
```

Both of these methods ensure that each goroutine receives its own copy of the loop variable, printing the expected "0, 1, 2" (though the order may vary).

1. The first method passes `i` as a parameter to the goroutine's anonymous function, ensuring each closure has its own copy.
  
2. The second method shadows the loop variable, essentially creating a new variable with the same name but a different memory location, again ensuring each closure has its own copy.

Understanding this behavior is crucial for avoiding subtle bugs when using goroutines in loops.

### Fixes for the Bug

1. Passing `req` as a parameter to the goroutine: This ensures that each goroutine gets its own copy of `req`.
2. Declaring a new variable inside the loop (`req := req`): This shadows the loop variable, giving each goroutine its own copy.

### Another Approach: Fixed Number of Goroutines

The last code block shows yet another strategy: creating a fixed number of handler goroutines (`MaxOutstanding` of them) upfront. All these goroutines listen to the same request channel, ensuring that only `MaxOutstanding` requests are processed at any given time.

This is a more efficient strategy because it avoids the overhead of continuously creating and destroying goroutines. It also makes the semaphore unnecessary, as the number of goroutines is fixed.

In the fixed number of goroutines approach, a pre-defined number of goroutines are launched, and each of them is running the `handle` function. These goroutines concurrently pull requests from the same request channel and process them. Because a channel in Go is a synchronized queue, each request will be processed by exactly one of the goroutines.

### What is `<-quit` used for?

The `<-quit` line is a blocking receive operation on a channel named `quit`. The `Serve` function will not terminate until it receives a message on this channel. Essentially, this provides a way to signal the `Serve` function to stop its operation. You would send a message into the `quit` channel from another part of your program when you want to stop the server.

Here's a basic example to illustrate:

```go
package main

import (
	"fmt"
	"time"
)

const MaxOutstanding = 3

func process(r int) {
	time.Sleep(time.Second)  // Simulating work
	fmt.Println("Processed", r)
}

func handle(queue chan int) {
	for r := range queue {
		process(r)
	}
}

func main() {
	clientRequests := make(chan int)
	quit := make(chan bool)

	// Start Serve function in its own goroutine
	go func() {
		// Start handlers
		for i := 0; i < MaxOutstanding; i++ {
			go handle(clientRequests)
		}
		<-quit  // Wait to be told to exit.
		fmt.Println("Server exiting.")
	}()

	// Simulate some requests
	go func() {
		for i := 0; i < 10; i++ {
			clientRequests <- i
		}
	}()

	// Wait for a while and then signal the server to stop
	time.Sleep(5 * time.Second)
	quit <- true  // Send signal to quit

	// Wait to make sure everything is printed before the program exits
	time.Sleep(1 * time.Second)
}
```

In this example, the `Serve` function (which is simulated by the anonymous goroutine in the `main` function) starts `MaxOutstanding` number of handler goroutines and then waits for a message on the `quit` channel. When it receives that message, it prints "Server exiting" and terminates.

By sending a message into the `quit` channel after some time (`quit <- true`), you're signaling the server to stop running.

### Summary

1. Buffered channels can be used as semaphores to limit concurrency.
2. Be careful when capturing loop variables in goroutines.
3. Consider different strategies for managing resources effectively, like gating the creation of goroutines or using a fixed number of goroutines.
