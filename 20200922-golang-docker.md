## Do Golang Web Apps Dream of Docker Images? 

Hello there! In my [last blog post](https://evalle.github.io/blog/20200917-ingress-nginx), I used a simple NodeJS application for testing Kuberentes Ingress. I was thinking that it can be useful to write a similar app in Golang. As you can see from the title of this post I chose Golang.
I'm going to deploy the app with docker, so this blog post will be useful for those of you who were wondering how to deploy a Golang web app with Docker.

### Our Golang Application

It's really simple. The goal is to show the hostname to the user. And here is the code: 

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "os"
)


func handler(w http.ResponseWriter, r *http.Request) {
    name, err := os.Hostname()
    if err != nil {
        panic(err)
    }
    fmt.Fprintf(w, "Hi there, my name is %s\n", name)
}

func main() {
    http.HandleFunc("/", handler)
    log.Fatal(http.ListenAndServe(":8080", nil))
```

The `main` function begins with a call to `http.HandleFunc`, which tells the http package to handle all requests to the web root `"/"` with `handler`.
It then calls `http.ListenAndServe`, specifying that it should listen on port `8080` on any interface `":8080"`. 
`ListenAndServe` always returns an error, since it only returns when an unexpected error occurs. In order to log that error, we wrap the function call with `log.Fatal`.
The function handler is of the type `http.HandlerFunc`. It takes an `http.ResponseWriter` and an `http.Request` as its arguments.
An `http.ResponseWriter` value assembles the HTTP server's response; by writing to it, we send data to the HTTP client.
An `http.Request` is a data structure that represents the client HTTP request. 
Function `os.Hostname()` returns the hostname reported by the kernel.
Save this code as `main.go` and let's compile it. 

### Compiling

If you use Linux-based OS, run:

```bash
$ go build main.go
```

in case of MacOS:

```bash
$ go build GOOS=darwin main.go
```

and last, but not least - Windows:

```bash
$ go build GOOS=windows main.go
```

Now, let's test it:

```bash
$ ./main &

$ curl 127.0.0.1:8080
Hi there, my name is evalle.local
```

It Works! Now let's prepare this app for the Docker Image.  
We want to build the app *statically* (with all libraries included) so we don't have to build anything inside of the image. In order to achieve this, we should run the command:

```bash
$ CGO_ENABLED=0 GOOS=linux go build -installsuffix cgo -o main .
```

It will disable `cgo` which gives us a static binary. We’re also setting the OS to Linux (again, if you build this app on Windows or on Mac machine (as I do) ). 
In the end, you should have a statically linked `main` binary in your working directory

```bash
$ ls
main main.go

$ file main
main: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, with debug_info, not stripped
```

Now we're ready to add it to the Docker image. 

### Docker phase

Our Dockerfile should look like this:

```Dockerfile
FROM scratch
ADD main /
CMD ["/main"]
```

`scratch` is the empty Docker image. We `ADD` our `main` golang binary to the image and the start it as the default command (`CMD`). Save it as the `Dockerfile` in your working directory and let's build it:

```bash
$ docker build -t evalle/gordon-golang:v0.1 .
Sending build context to Docker daemon  6.614MB
Step 1/3 : FROM scratch
 --->
Step 2/3 : ADD main /
 ---> 73c266a64776
Step 3/3 : CMD ["/main"]
 ---> Running in 6545a3d83824
Removing intermediate container 6545a3d83824
 ---> 684bcde76982
Successfully built 684bcde76982
Successfully tagged evalle/gordon-golang:v0.1
```

Now, let's run the container from it:

```bash
$ docker run -d -p 8080:8080 evalle/gordon-golang:v0.1
ca7ee636c1aa893e0679f764e417f4eacbe4c196e4b79da513ccfb1c2e46d9de
```

And let's test it:

```bash
$ curl 127.0.0.1:8080
Hi there, my name is ca7ee636c1aa
```

It works! And as you can see it uses container ID as the hostname which is expected. Now you can use this docker image to play with [Kubernetes Ingress](https://evalle.github.io/blog/20200917-ingress-nginx) instead of NodeJS app I've created. Also, you can use the same technique to build and deliver a more complex golang applications.   

## Useful Links

- [Writing Web Application (in Go)](https://golang.org/doc/articles/wiki/)
