![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*hJR81BXeqgN0zwkPOxkQVg.png)

Global Exception handling in Gin
================================

Coming from Nodejs background, I am very familiar with the express framework. In express, we used to hook the error middleware to handle and send a custom response. Surprisingly, it is very much the same in Gin !

For those new to gin, it is lightweight framework to write HTTP based APIs in Go.

In order to customize the error response on a runtime error or any other error thrown by the app, we use the [**CustomRecovery**](https://pkg.go.dev/github.com/gin-gonic/gin#CustomRecovery) middleware provided by gin. It executes whenever there is any error and then passes off the error to a custom function of the type `RecoveryFunc func(c *[Context](https://pkg.go.dev/github.com/gin-gonic/gin#Context), err [any](https://pkg.go.dev/builtin#any))`

So, let’s write our recovery function first,
https://gist.githubusercontent.com/SpartanX1/9c1a246d39e41712dc4fbc7100175e9e/raw/12566ebedf1561372643d9e969caf2a9832b3b3a/exception.handler.go

Couple of things to note here, first we are using the [go-errors](https://pkg.go.dev/errors) package to wrap the error thrown by the gin recovery middleware using `errors.Wrap(err, 2)` . The 2nd parameter indicates how far up the stack to start the stacktrace. 0 is from the current call, 1 from its caller, etc. For now, we will not use it.

Once we wrap the err object, we can then call the Error() method on it which would give us the error message. So far so good.

Now, in order for this to take effect, we need to explicitly pass this func as a param to gin’s recovery middleware
https://gist.githubusercontent.com/SpartanX1/65202e5358ca70e4107339f5f27e1ae5/raw/3b7e4d80db1b16ab1d90e77d8c1c953e97378cce/main.go

and there you go, now any errors will be routed through the error handler !

Where to go from here: You can check for specific type of runtime errors and then send appropriate responses.

Thanks for reading.
