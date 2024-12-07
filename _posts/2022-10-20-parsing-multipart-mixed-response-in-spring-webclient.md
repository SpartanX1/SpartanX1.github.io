Parsing multipart/mixed response in Spring Webclient
====================================================

Uploading multipart data is very common nowadays. It is a http format which lets you attach multiple files to the request body of an REST API. What is uncommon, however, is receiving such data as multipart. So, if you are stuck with receiving a multipart response from which you need to extract images or files, then this guide is for you :)

Format
------

Specifically, I am gonna talk about the `multipart/mixed` data format. If we take a look at the [w3](https://www.w3.org/Protocols/rfc1341/7_2_Multipart.html) specification, the format looks something like this

```txt
--ABu8YWMq2QCE2oFsfjUYozp83ssXunG5pRGU9Eb 
Content-Disposition: form-data; name="responseMessage"
Content-Type: application/json

{"status":"Success","pageCount":"1"}
--ABu8YWMq2QCE2oFsfjUYozp83ssXunG5pRGU9Eb
Content-Disposition: attachment; filename=Page1.jpeg
Content-Type: image/jpeg

...GIF89a� 
--ABu8YWMq2QCE2oFsfjUYozp83ssXunG5pRGU9Eb--
```

Quoting from w3,

> Each part starts with an encapsulation boundary, and then contains a body part consisting of header area, a blank line, and a body area

We can have multiple responses with each one following the same `boundary-header-body` format.

Parsing
-------

Although we can parse this response ourselves since we already know the format, it would be tedious and time consuming. Fortunately Spring has exactly the thing we are looking for !

Introducing the `[DefaultPartHttpMessageReader](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/http/codec/multipart/DefaultPartHttpMessageReader.html)` . It reads the `"multipart/form-data"` requests to a stream of `[Part](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/http/codec/multipart/Part.html)`s. Now let’s see how we can use this to get what we want.

First, we need to initialize the reader

```java
final var partReader = new DefaultPartHttpMessageReader();
partReader.setStreaming(false);
```

I have set the streaming to false and going to be using blocking code here after, but please feel free to make it non-blocking.

Next up, let’s build our webclient to make a request which would be receiving the multipart data

```java
WebClient webClient = WebClient.builder().build();
ResponseEntity<Flux<Part>> request = webClient
                .get()
                .uri("...")
                .accept(MediaType.MULTIPART_MIXED)
                .retrieve()
                .toEntityFlux((inputMessage, context) ->
                        partReader
                                .read(ResolvableType.forType(byte[].class), inputMessage, Map.of()))
                .block();
```

We are using the `toEntityFlux` method which gives us access to the response which we will be reading via the `read` function provided by the reader class. At the time of writing, I didn’t find any other method apart from toEntityFlux which would provide a reactive response stream supported by the read function.

Also, a very important thing to note is that, is, whenever we handle multipart data, we always do so in binary format. In Java `byte[]` and `DataBuffer` class come in handy to do that.

Once we execute the request, we will get `Flux<Part>` as response with each part being the individual parts of the multipart response.

For eg, if you want to extract an image from the response and send it in bytes, you can do this

```java
byte[] image = null;
    
List<Part> parts = request.getBody().collectList().block();

for (Part part : parts) {
    // access individual parts here
    System.out.println(part.headers());
    if (part.headers().get("Content-Type").get(0).equals("image/jpeg")) {
        image = DataBufferUtils.join(part.content())
                .map(dataBuffer -> {
                    byte[] bytes = new byte[dataBuffer.readableByteCount()];
                    dataBuffer.read(bytes);
                    DataBufferUtils.release(dataBuffer);
                    return bytes;
                }).block();
    }
}

return image;
```

`part.headers()` will give you the headers and `part.content()` will give you the actual content in [DataBuffer](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/io/buffer/DataBuffer.html) format.

We iterate the list of parts, and determine which one is an image by peeking into the headers. Once we get that, we use the `DataBufferUtils` , a helper class to convert the buffer into byte array format.

And that’s it, extracting images/files from multipart data is now a walk in the park thanks to Spring Flux !

Thank you for reading.
