+++
date = '2023-12-01T11:17:04+08:00'
draft = false
title = 'Common Go channel uses'
+++

Go is known for its native support for concurrency. You can easily spin up a concurrent execution (i.e. goroutine) using keyword `go`.

It is similar to OS thread but managed by Go runtime scheduler instead of OS kernel. The underlying implementation actually employs an M:N model to multiplex M goroutines onto N threads.

With the great power of concurrency comes with the responsibility to handle synchronisation in the right manner.

In Go, we can pass messages between different thread of executions using *channel*.

### Used as signal

Think of sending signals (`kill -s signal pid`) in the OS inter-process communication:

```go
// create an unbuffered channel
done := make(chan int)
// create a parallel execution path to start up a long-running service
go func() {
	server.run()
	// once the server exits, it synchronises the termination to other threads of execution 
	// by passing a signal to the channel
	done <- 1
}()
// the execution blocks here until a signal is received from the channel
<-done
```

### Used as message queue

The typical producer-consumer pattern can be implemented with Channel.

```go
// create a buffered channel with size 10; in this case, producer can keep adding messages to the channel
// without having to wait for receiver to receive (until the channel capacity is reached).
// In the unbuffered case, sender blocks for every message until the receiver has received it.
queue := make(chan string, 10)

// start the producer goroutine
go func() {
	messages := []string{"order-1", "order-2", "order-3"}
    for _, msg := range messages {
		queue <- msg // send message to the channel
    }
	close(queue) // done with producing messages
}()

// the main goroutine acts as the consumer
for msg := range queue {
	fmt.Println("Received: %s", msg)
}
```

There is one more builtin call I didn't mention in the previous post on [Go Builtins](../go-built-in/).
That is the`close()`call

You can think of it as the TCP flag`FIN`to indicate that sender has no more data to send and thus the communication channel should shut down.

If the channel is not closed in the above example, the`range`loop will block and wait for a value, but loop terminates when`close()`is called on the`queue`channel.

The Go doc explains it in more details:

> The close built-in function closes a channel. It should be executed only by the sender, never the receiver, and has the effect of shutting down the channel after the last sent value is received. After the last value has been received from a closed channel c, any receive from c will succeed without blocking, returning the zero value for the channel element.

### Used for timeout

Channel also comes in handy when you want to enforce timeout for some functions.
Think of *timer interrupt* in the OS used to switch context and execute scheduling decision.

`time.After()` already provided a convenient interface for this:

```go
for {
	// the 'select' will choose the first case that unblocks; otherwise it blocks
	select {
	case m := <-orderQueue:
		fmt.Printf("New order: %s\n", m)
	// wait for maximum 1 minute before moving on
	case t := <-time.After(1 * time.Minute):
		fmt.Println(t.Format(time.RFC850)) // Friday, 01-Dec-23 12:16:56 +08
		fmt.Println("Time out: no order received")
	}
}
```
