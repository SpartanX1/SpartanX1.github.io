---
tags: Go
---
![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*hJR81BXeqgN0zwkPOxkQVg.png)

Global Exception handling in Gin
================================

Coming from Nodejs background, I am very familiar with the express framework. In express, we used to hook the error middleware to handle and send a custom response. Surprisingly, it is very much the same in Gin !

For those new to gin, it is lightweight framework to write HTTP based APIs in Go.

In order to customize the error response on a runtime error or any other error thrown by the app, we use the [CustomRecovery](https://pkg.go.dev/github.com/gin-gonic/gin#CustomRecovery) middleware provided by gin. It executes whenever there is any error and then passes off the error to a custom function of the type 

```go
RecoveryFunc func(c *Context, err any)
```

So, let’s write our recovery function first,
```go
import (
	"github.com/gin-gonic/gin"
	"github.com/go-errors/errors"
)

type HttpResponse struct {
	Message     string
	Status      int
	Description string
}

func ErrorHandler(c *gin.Context, err any) {
	goErr := errors.Wrap(err, 2)
	httpResponse := HttpResponse{Message: "Internal server error", Status: 500, Description: goErr.Error()}
	c.AbortWithStatusJSON(500, httpResponse)
}
```

Couple of things to note here, first we are using the [go-errors](https://pkg.go.dev/errors) package to wrap the error thrown by the gin recovery middleware using `errors.Wrap(err, 2)` . The 2nd parameter indicates how far up the stack to start the stacktrace. 0 is from the current call, 1 from its caller, etc. For now, we will not use it.

Once we wrap the err object, we can then call the Error() method on it which would give us the error message. So far so good.

Now, in order for this to take effect, we need to explicitly pass this func as a param to gin’s recovery middleware

```go
func main() {
	server := gin.Default()
	...
	server.Use(gin.CustomRecovery(ErrorHandler))
	...
	server.Run()
}
```

and there you go, now any errors will be routed through the error handler !

Where to go from here: You can check for specific type of runtime errors and then send appropriate responses.

Thanks for reading.
