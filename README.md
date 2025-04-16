# go_tips_and_tricks

I code in Go quite a lot, but from time to time I run into things where I start
second guessing myself if I remembered correctly how it works. So in this
document I will collect stuff that might have taken a while to figure out, or
stuff that I just find neat to store for future projects.

I think I will keep everything in a single document so it will be easier to
`Ctrl+F` stuff.

## Channels
Go channels are very nice to work with when you need to send stuff between
different goroutines, but there are situations where it is not entirely
clear what will happen.

### Basics
Before we dive into the more exotic stuff we start with some basic stuff.

#### Creating and Writing
Channels can be both buffered and unbuffered, and it affects what will happen in
case we try to add something to a "full" channel.
<table>
<!--
  The conservative amount of indentation is because GitHub does not properly
  render stuff that is indented more than 3 spaces, because then it becomes a
  codeblock.
-->
 <tr>
  <th align="center">
   <img width="441" height="1">
   <p><small>Unbuffered</small></p>
  </th>
  <th align="center">
   <img width="441" height="1">
   <p><small>Buffered</small></p>
  </th>
 </tr>
 <tr>
  <td>

```go
c := make(chan string)
c <- "hello" // Will block here until someone reads.
```
  </td>
  <td>

```go
c := make(chan string, 1)
c <- "hello" // Will be added and execution continues.
c <- "world" // Will block here until someone reads.
```
  </td>
 </tr>
</table>

#### Reading
Reading from a channel can actually return one or two values. In the first case
we perform a simple read where we just wait for it to return one value.

```go
c := make(chan int, 1)
go func() {
    time.Sleep(1 * time.Second)
    c <- 4
    time.Sleep(1 * time.Second)
    close(c)
}

msg := <- c // Will block here until the message is recieved.
fmt.Println(msg) // "4"

msg = <- c // Will block here until the channel is closed.
fmt.Println(msg) // "0"
```

As you can see we just print the default value of the type defined for the
channel, which in this case is an `int` zero. However, it might sometimes be
of interest to know if it is an actual zero we received or if it is because
the channel was closed. This can be achieved by looking at the second return
value.

```go
c := make(chan int, 1)
c <- 0

msg, ok := <- c
fmt.Printf("%d:%t") // "0:true" => This is a "real" zero received.

c <- 4
close(c)

msg, ok = <- c
fmt.Printf("%d:%t") // "4:true" => This is a "real" four received.
msg, ok = <- c
fmt.Printf("%d:%t") // "0:false" => Channel closed; the 0 is just the default value.
```

### Closed Channel Behavior
When a channel is closed the behavior of it changes, and it is especially
important to know that writing to a closed channel will cause a `panic`. But
we begin with the more safe reading case.

#### Reading
In the example above we observe that we receive an `ok = true` when we read the
value 4, even though the channel is already closed. That is good to know, since
it allows us to perform stuff like this:

```go
select {
case msg, ok := <- c:
    if !ok {
        // Channel closed, time to exit gracefully.
        return nil
    }
    // Handle the message we just recieved.
case <-ctx.Done():
    // Context was canceled before all messages were handled.
    return context.Cause(ctx)
}
```

However, do remember that if the context is canceled and the channel is either
closed or there is a message present, Go will select one of these on random.
This behavior is true for all `case`:es that are true/ready, so the order of
these doesn't matter. A more robust solution to the example above could
thus be:

```go
for {
    if ctx.Err() != nil {
        return context.Cause(ctx)
    }
    select {
    case msg, ok := <- c:
        if !ok {
            return nil
        }
        // ...
    case <-ctx.Done():
        continue // Affects the for-loop.
    }
}
```

#### Writing
Writing to closed channels will cause a `panic`, so it is important to be
careful when coordinating the closing of a channel if there are multiple
writers.

```go
c := make(chan string, 1)
close(c)
c <- "hello" // This is will cause a panic!
```

Unfortunately there is no `ok` equivalent (compared to the reading case) when
writing, so only close channels when you are certain no more writes will be
made to it.

### Non-Blocking Writes
While we usually want to add the thing we have to the channel, there might be
cases where we don't want to hang the entire program because of some slow
consumers. It is therefore possible to just continue execution in case the
channel is full like this:

```go
c := make(chan string, 1)
c <- "hello"
select {
case c <- "world":
    // This will only happen if there is room in the channel.
default:
    // This will only happen if the channel is full.
    fmt.Println("Dropped the message 'world' because the channel was full")
}
// Code continues execution.
```

> This still causes a `panic` if the channel is closed.

Doing this for an unbuffered channel doesn't really make sense since all
messages will be dropped unless there is a reader ready to immediately receive
the message.

### Non-Blocking Reads
Like in the case of non-blocking writes, you can do a similar trick for reading
channels. The usability of this is less obvious, but it is a nice way of
quickly "draining" a channel if that ever becomes relevant.

```go
for {
    select {
    case msg, ok := <-c:
        if !ok {
            // The channel is closed so there are no messages left.
            return
        }
        fmt.Printf("Discarding the follwing message: %s\n", msg)
    default:
        // If we get here there are currently no messages in the channel, but
        // it is still open and new messages can still be written to it.
        return
    }
}
```
