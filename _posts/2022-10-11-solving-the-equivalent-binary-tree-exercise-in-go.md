---
tags: Go
---
![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*Ifpd_HtDiK9u6h68SZgNuA.png)

Solving the Equivalent Binary Trees exercise in Go
==================================================

Go is a fast and lightweight programming language which I recently started exploring. This article talks about the notorious equivalent binary tree program which you encounter in [https://go.dev/tour/concurrency/8](https://go.dev/tour/concurrency/8)

Solving this program requires paying attention to the previous chapters on Go concurrency. I know it’s a little difficult to follow through it, but don’t worry, I will explain how it works :)

Let’s start off reviewing the problem statement. It says that we consider two binary tress equivalent if and only if they store the same sequence of numbers. In other words, say, that we traverse each tree using in-order traversal, then the result set of numbers should be same in both.

> Binary trees, they are never gonna leave me alone.

Okay so the tree structure is already given. I know GO has pointers, but they are less scary than C ones.

```go
type Tree struct {
    Left  *Tree
    Value int
    Right *Tree
}
```

and let us first implement the walk function

```go
func Walk(t *tree.Tree, ch chan int)  {
 if t != nil {
  Walk(t.Left, ch)
  ch <- t.Value
  Walk(t.Right, ch)
 }
}
```

If you recall doing traversal of binary trees (yes, maybe only in your college :P ) , this is quite the same. We recurse left -> root -> right

The only thing to note here is that instead of storing the value of root in an array, we send it to the channel `ch`. Channels are used to send and receive data by go routines. Let’s add another function to call this `walk`

```go
func Walking(t *tree.Tree, ch chan int) {
 Walk(t, ch)
 defer close(ch)  // close channel after Walk() finishes
}
```

This takes care of closing the channels by using `defer` . When Walk returns, it will close the channels so that we do not fall into a deadlock. You may ask as to why we need to close it ? Well, that’s because we would be continuously reading value from the channel in a for loop and closing the channel would in turn help us terminate the loop. Time to implement the Same function !

**Note**: If we had used defer inside the `Walk` , then any of the channels would have been closed accidently since the function is shared between the routines. The behavior would be unexpected and hence we should call `defer close(ch)` outside of the recursive function.

```go
func Same(t1, t2 *tree.Tree) bool {
 x := make(chan int)    // define separate channels for both trees
 y := make(chan int)
 
 go Walking(t1, x)      // call the Walking func for both trees
 go Walking(t2, y) 
 
 for {
  v1, ok1 := <-x        // receive from channel x and y    
  v2, ok2 := <-y     // the ok param tells us if channel is closed
  
  if ok1 != ok2 || v1 != v2 {
          return false
        }
        if !ok1 {
          break;
        }
 }  return true
}
```

To test whether a channel has closed or not, we use the `v, ok := <- ch` statement as explained in [https://go.dev/tour/concurrency/4](https://go.dev/tour/concurrency/4)

Now, the only thing left is our main() func

```go
func main() {
 if Same(tree.New(1), tree.New(1)) {
  fmt.Print("Yes!")
 } else {
  fmt.Print("No!")
 }
}
```

Again, tree.New() function is already provided by the go program. In this case, the output from tree one and two would be `[1,2,3,4,5,6,7,8,9,10]`

That’s it, now you know how go routines communicate using channels. Thanks for reading !a
