# Tapas - A tiny-imaged (8MB) minimal static HTTP file server with Go and Docker

[![Docker Repository on Quay](https://quay.io/repository/leopoldodonnell/tapas/status "Docker Repository on Quay")](https://quay.io/repository/leopoldodonnell/tapas)

Tapas are quick and easy, and so is this recipe for standing up a *quick-and-dirty* static http file server
that is under 10MB.

Another way to look at this project is that it demonstrates how to use `golang` to build Docker images
that are sub 10MB. The recipe supplied can be replicated in other projects as an approach to build up
microservice architectures with tiny Docker images that are easily shipped and started.

This project is less about the actual server code (which you'll find in `main.go`) than it
is about learning how to leverage this approach to tiny images as you get a simple static file
web server.

## Getting Started

1, If you don't already have Docker > 1.10 installed on your development machine, visit the
[Install Docker Engine](https://docs.docker.com/engine/installation/) page and do that now. Then
get your `docker-machine` started. ie `docker-machine start your-machine-name-here`

1. If you are going to run the server on a port other than `80`, edit the `docker-compose.yml` file
and change `80:3000` to something that will work for you like `8080:3000`.

1. Run and verify the tapas server

    > docker-compose up -d
    > docker-compose logs
    tapas-server | + exec app
    tapas-server | 2016/04/12 13:59:53 Listening...
    ^c

Now test that you can serve up a file...

    > curl http://your-docker-ip:your-port/tapas.html
    <div>
      <h1>Tapas are Tasty!</h1>
      <p>
        Ole
      </p>
    </div>

Congratulations! You've got a running static http file server!

## Add your own files

So far, so good. Go ahead and add any files you want to serve up.

    > cp your-files ./static
    > docker-compose build
    > docker-compose stop
    > docker-compose rm
    > docker-compose up -d

Now you can surf on over using `static` as your document root `/`. eg /static/hello-world.html would have
the following url.

    http://your-docker-ip:your-port/hello-world.html

## Make it Tiny

Take a look at your image:

    > docker images
    REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
    tapas               latest              a9affeb76417        18 minutes ago      716.6 MB

**Holey peaella Batman, its huge!**

Not a problem, we can shrink this image.

One of the great things about writing servers in `go` is that `go` is compiled **AND** we can statically
build applications it. For this work I've taken liberal advantage of the following article:
[Building Minimal Docker Containers for Go Applications](https://blog.codeship.com/building-minimal-docker-containers-for-go-applications/)


To get this done, we'll use the standard community golang docker image to build the tapas binary and then `ADD` that binary into
the most minimal of Docker images, using a multistage build and the `FROM scratch` directive.

**Dockerfile.scratch**

Take a look at `Dockerfile.scratch`

```docker
FROM golang as builder
COPY tapas.go /app/tapas.go
WORKDIR /app
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o tapas .

FROM scratch
ADD ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/tapas /
ADD static /static
CMD ["/tapas"]
```

The top half of the Dockerfile is labeled as *builder* and is used to compile the simple `tapas.go` into the statically linked
 `/app/tapas`. Then the second half of the multistage build completes the work to create the final container image. It starts with
 the `FROM scratch` directive that tells docker to use the next Dockerfile command as the first filesystem layer. This doesn't have
to be a complete operating system like `ubuntu`, `alpine`, or `jessie` (to name a few `linux` varieties). Using
a statically linked application will do just fine in many cases. This lets us build tiny images for tiny containers
that take no time to download from a *docker registry*.

Next we'll use two build idioms, one for building things locally, another for when you are building on remote
Docker Engines that don't have access to your filesystem.

### Make it Tiny Locally

To build something locally, we'll just redirect the `docker-compose.yml` to use the new multistage build and then rerun the
build command.

Start by editing the `docker-compose.yml` file and change `Dockerfile` to `Dockerfile.scratch`. Then ...

Build a static version of `tapas`...

    > docker-compose build


Take a look at the size of your new image:

    > docker images
    REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
    tapas               latest              02ad557c10d8        3 seconds ago        6.45 MB

**Nice!** only 6.45MB

So, does it work?

Edit the `docker-compose.yml` file and change `Dockerfile` to `Dockerfile.scratch`. Then ...

    > docker-compose up

Now see that it still works!


## Re-using the Image

Once you've built this image, you could run it and mount files from the docker host's file system without having
to build your own image.

Update the `docker-compose.yml` file to add your volume, or you can run it directly as follows:

    > docker run --rm --name "tapas-server" -v /my-document-root:/static -p 80:3000 tapas

This will pull the tapas image from the registry and run it on port 80 and serve files from  `/my-document-root`

## Recap

Now you've seen how you can build small Docker images and you've got a re-usable static web server that you
can pull very quickly from a registry and host as needed. More importantly, its shown you how you might build
small servers to do all sorts of work in your architecture. This could include standard REST services, or could
include **RabbitMQ** workers; your imagination is all that it takes.

I hope you've enjoyed this project, please Star it, clone it, share it etc.

Thanks,
   Leo O'Donnell


## Copyright

MIT License.
Copyright (c) 2016 Leopold O'Donnell

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated
documentation files (the "Software"), to deal in the Software without restriction, including without limitation
the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software,
and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions
of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED
TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF
CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS
IN THE SOFTWARE.
