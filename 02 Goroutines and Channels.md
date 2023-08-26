> Goroutines
They're called goroutines because the existing terms—threads, coroutines, processes, and so on—convey inaccurate connotations. A goroutine has a simple model: it is a function executing concurrently with other goroutines in the same address space. It is lightweight, costing little more than the allocation of stack space. And the stacks start small, so they are cheap, and grow by allocating (and freeing) heap storage as required.

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
