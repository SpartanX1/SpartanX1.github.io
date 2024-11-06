---
tags: Go
---
![](https://cdn-images-1.medium.com/max/2000/1*KIsHdDLv71tE1dueicb7cw.png)

## Job Queue in Go

One of the strong points of Go, is it’s built in support for concurrency. The way it achieves this is via go routines. Fancy name eh ? Well go routines are supposed to be lighter than threads in terms of both speed and memory consumption, hence the name ?

Go routines communicate via channels. We send and receive values from go routines via these channels. They are also concurrency safe; which means that no two go routines listening on the same channel will pick up the same value. This property makes it incredibly suitable for implementing a queue, and that’s exactly what we will be doing today.

Let’s start off by writing a simple go routine which listens to a channel,

```go
package main

import (
 "fmt"
)

func process(jobChannel chan int, worker int) {
 for job := range jobChannel {
  fmt.Printf("Job %v picked up by %v\n", job, worker)
 }
}

func main() {
 jobChannel := make(chan int, 10)

 go process(jobChannel, 0)

 for i := 0; i < 1000; i++ {
  jobChannel <- i
 }

 close(jobChannel)

}
```

The code creates a channel and then launches a go routine. We only have one routine running right now, hence we just passed a 0 number to identify the routine. The output of the above should be “Job n picked up by 0” where n appears in a non sequential manner.

But wait, we missed something in the above code to make it work…a go routine runs on a separate thread, which means that the main function is not gonna wait around for it to complete ! That is why, we need to add a wait group to signal the main func to wait.

Let’s add it,

```go
package main

import (
 "fmt",
 "sync"
)

var wg sync.WaitGroup

func process(jobChannel chan int, worker int) {
 defer wg.Done()
 for job := range jobChannel {
  fmt.Printf("Job %v picked up by %v\n", job, worker)
 }
}

func main() {
 jobChannel := make(chan int, 10)
 
 wg.Add(1)
 go process(jobChannel, 0)

 for i := 0; i < 1000; i++ {
  jobChannel <- i
 }

 close(jobChannel)
 wg.Wait()
 
}
```
Go provides the built in WaitGroup class from the sync package to achieve the same.

Now that our code is working, we still have only one process (worker) listening in on the channel. But in read world, multiple workers are needed in order to maximize the performance. If we add, say 3 more workers, then the work will be distributed among these and we can truly implement a performant queue.

In order to do that, we can use a simple for loop to launch more than one go routines listening in on the same channel.

```go
package main

import (
 "fmt"
 "sync"
)

var wg sync.WaitGroup

func process(jobChannel chan int, worker int) {
 defer wg.Done()
 for job := range jobChannel {
  fmt.Printf("Job %v picked up by %v\n", job, worker)
 }
}

func main() {
 jobChannel := make(chan int, 10)
 
 wg.Add(1)

 for i := 0; i < 4; i++ {
  go process(jobChannel, i)
 }

 for i := 0; i < 1000; i++ {
  jobChannel <- i
 }

 close(jobChannel)

 wg.Wait()
}
```
>  Note: A very important thing to mention here is that we have used buffered channels. Buffered channels are used to handle back pressure; i.e the channel will block incoming values if it’s full. This way we can make sure not to overwhelm the go routines

No matter how many times you run this, the same job will never be picked up twice ! And there you go, a simple and elegant job queue in Go :)

Thanks for reading.
