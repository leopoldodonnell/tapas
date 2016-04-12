# Tapas - A tiny-imaged (8MB) minimal static HTTP file server with Go and Docker

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
    tapas               latest              a9affeb76417        18 minutes ago      759.5 MB

**Holey peaella Batman, its huge!**

Not a problem, we can shrink this image.

One of the great things about writing servers in `go` is that `go` is compiled **AND** we can statically
build applications it. For this work I've taken liberal advantage of the following article: 
[Building Minimal Docker Containers for Go Applications](https://blog.codeship.com/building-minimal-docker-containers-for-go-applications/) 


To get this done, we'll use a separate build Docker image and a runtime image that uses the most minimal of
Docker images, the `FROM scratch` image. These we'll call...

* Dockerfile.build
* Dockerfile.scratch

Take a look at `Dockerfile.scratch`

```docker
FROM scratch
ADD ca-certificates.crt /etc/ssl/certs/
ADD main /
ADD static /static
CMD ["/main"]
```

`FROM scratch` tells docker to use the next Dockerfile command as the first filesystem layer. This doesn't have
to be a complete operating system like `ubuntu`, `alpine`, or `jessie` (to name a few `Linix` varieties). Using
a statically linked application will do just fine in many cases. This lets us build tiny images for tiny containers
that take no time to download from a *docker registry*.

Next we'll use two build idioms, one for building things locally, another for when you are building on remote
Docker Engines that don't have access to your filesystem.

### Make it Tiny Locally

To build something locally, we'll use a mount point to access our current working directory. We'll build the
image, then run it to create a statically linked binary. Finally we'll take the binary, add it to your runtime
`Docker image` and start it up.

Start by editing the `docker-compose.yml` file and change `Dockerfile` to `Dockerfile.scratch`. Then ...

    > docker build --rm -t tapas.build -f Dockerfile.build .
    > docker run --rm -v $(pwd):/app tapas.build go build -a -installsuffix cgo -o main .
    > docker-compose build

Note that `Dockerfile.build` uses the stock `golang` image from [Docker Hub](https://hub.docker.com/) then sets
the environment variables `CGO_ENABLED=0` and `GOOS=linux` before building the source.

Note that we've mounted our current directory and run `go` to build the application. When we're done we should
see `main` in our current directory. Running `docker-compose` to build, creates our final image that we can push
up and deploy.

Take a look at the size of your new image:

    > docker images
    REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
    tapas               latest              02ad557c10d8        3 seconds ago        7.951 MB
    tapas.build         latest              88e6eaa7347b        About a minute ago   759.5 MB

**Nice!** only 7.9MB

Stop, remove the old container and start up the new one.


### Make it Tiny Remotely

Let's say you've got a build machine, or you need to get your image onto a remote machine without
a repository. To do this, we'll need to take a slightly different approach since the local filesystem 
won't be available.

If you haven't setup your remote `docker-machine` and have an **AWS** account, you can use the following
bash commands (sorry Windows devs - it should be similar, but not exactly like this). Make sure you've
setup your AWS credentials in your `~/.awssecret` file ...

    IFS=$'\r\n' GLOBIGNORE='*' :
    AWS_CREDS=($(cat ~/.awssecret))
    
    MACHINE=your-machine-here
    
    docker-machine -D create \
        --driver amazonec2 \
        --amazonec2-access-key ${AWS_CREDS[0]} \
        --amazonec2-secret-key ${AWS_CREDS[1]} \
        --amazonec2-instance-type m3.large \
        --amazonec2-region us-east-1 \
        --amazonec2-root-size 16 \
        --amazonec2-ami $AMI \
        $MACHINE

Make sure that you have your chosen port is open via the host's security group.

Once your machine has been created, you can get it's IP by using the following command:

    > docker-machine ip your-machine-name

And set your docker-machine

    > eval $(docker-machine env your-machine-name)

The trick to building the image remotely is to take advantage of the ability to copy files to
and from containers. This looks something like:

    > docker cp container_name:file_path_in_container file_path_on_local_filesystem
    > docker cp -r some_root_path_on_local_filesystem container_name:file_path_in_container

So, starting with the `Dockefile.scratch` you set up before, the steps to building the container are...

    > docker build --rm -t tapas.build -f Dockerfile.build .
    > docker create --name=tapas.build tapas.build
    > docker cp tapas.build:/app/main .
    > docker rm tapas.build
    > docker-compose build

A lot of the steps are the same as we used when building the image locally. The main difference is
that instead of running a Docker container to build the result, we're copy the binary that was
created during the build process once we've created a container. (Note that we remove the build container, 
since its no longer necessary.)

Take a look at the size of your new image:

    > docker images
    REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
    tapas               latest              02ad557c10d8        3 seconds ago        7.951 MB
    tapas.build         latest              88e6eaa7347b        About a minute ago   759.5 MB

**Nice!** only 7.9MB

Stop, remove the old container and start up the new one.

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
