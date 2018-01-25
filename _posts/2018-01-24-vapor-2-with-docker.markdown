---
layout: post
title:  "Vapor 2 with Docker"
date:   2018-01-25 09:00:00 +1100
---
[Vapor][vapor-home] is a development framework for servers written in Swift (often known as server-side Swift). In this tutorial, I'll be using the latest release of Vapor 2, which is 2.4.3, and Swift 4.0.3.

[Docker][docker-home] is a comprehensive yet simple containerisation system. In addition to the performance benefits from using light-weight containers instead of heavy-weight VMs, a key benefit to containerisation is that you can easily run the same container in development as you will in production. I'll be using Docker Community Edition, version 17.12.0 on MacOS.

There's two particular reasons to use Docker with server-side Swift.

First, Swift on Linux has a (historically deserved, but less-so now) poor reputation for missing features available on Mac. By developing and testing in a Docker container, you'll be able to find any inconsistencies early on.

Second, building a Swift project requires the Swift toolchain to be present, along with several development dependencies, and uses a large amount of memory. Running a Swift binary, however, needs much less. Docker's multi-stage build process allows you to develop and build using one large image, and then deploy to production using a second minimal image.

Here I'll describe my workflow for both development and production using Vapor and Docker. I'm assuming you have Docker installed on your development machine. You don't need to have the Vapor Toolbox installed, nor even Swift for that matter.

## Developing with Docker

The theory here is that you will build and run a Swift container, mounting your project directory, then keep it open during development using command-line Swift to build, test and run your project.

I code in Sublime Text 3, but there should be no difference if you are using Xcode.

### Creating a Vapor project

You'll need, of course, a Vapor project. Here's two ways to do it.

Using Swift Package Manager (creates project in current directory, you'll need to add Vapor to your `Package.swift` manually):

```
docker run -v "$PWD":/app -w /app swift:4.0.3 swift package init --type executable
```

Using Vapor Toolbox (creates project in subdirectory named `project_name`):

```
docker run -v "$PWD":/app -w /app vapor/toolbox:3.1.2 vapor new [project_name]
```

### Setting up your development image

You'll first need a development Dockerfile. I name mine, creatively, `Dockerfile-dev`. This Dockerfile will inherit from the official Swift image and install whatever dependencies you might need (MySQL client headers, for example).

Here's an example file:

{% highlight Docker %}
FROM swift:4.0.3
RUN apt-get -qq update && apt-get -q -y install \
  your-dependencies-here # e.g. libmysqlclient-dev libpq-dev etc
WORKDIR /app
{% endhighlight %}

*No dependencies? You can remove the entire `RUN` section. In fact, you don't strictly need a Dockerfile at all, but since most projects will need to install packages at some point, I think it's worth starting out with one.*

Now you are ready to run your development environment. Here's an example command that hosts your project in the `/app` directory and maps port 8080 to your local computer so you can test your project using your browser.

```
docker build -t myproject:dev -f Dockerfile-dev . && docker run -it -p 8080:8080 -v "$PWD":/app --privileged --rm myproject:dev
```

Let's go over that, because running complicated commands from untrusted websites is a bad idea. It's a two part command (see the `&&`). The first one, `docker build`, builds an image using your development Dockerfile and tags it as `myproject:dev`. The second one, `docker run`, runs a container from that image in interactive mode (`-it`), maps port 8080 (`-p 8080:8080`), mounts the project folder (`-v "$PWD":/app`), enables extra access required by the REPL (`--privileged`) and removes the image when done (`--rm`).

You can remove `--privileged` if you're not going to use the REPL.

### Building, testing and running

After running the previous command, you should see something like:

```
Step 3/3 : WORKDIR /app
 ---> e06c5e3d476e
Successfully built e06c5e3d476e
Successfully tagged myproject:dev
root@87b9684095a6:/app#
```

You're now sitting in a `bash` terminal inside your development container. Congratulations! You can begin working on your project, using `swift build`, `swift test` and `swift run` to build, test and run.

Using Xcode? Then by all means build as you go in Xcode (it's faster than building in Linux) but don't forget to build and test in your container from time to time, and you should always run from it. Xcode and Linux use different build directories, so always `swift build` before you `swift run` to ensure your Linux build is up-to-date.

If you need to add dependencies, press `Ctrl-D` to close your Docker container, edit your `Dockerfile-dev`, and then run the build-and-run command again.

Ready to deploy to production? Read on.

## Vapor and Docker in production

If you've been checking (with `docker image ls`) you'll have seen that your development image is very large (mine is 1.3GB). You don't want that in production. Fortunately, recent versions of Docker can perform multi-stage builds, where one Dockerfile can create a temporary 'build' image, and then use build artifacts from that image in a new 'release' image (typically around 200MB, plus dependencies).

### Setting up the production image

Create a file called `Dockerfile` with contents similar to the below.

{% highlight Docker %}
# Build image
FROM swift:4.0.3 as builder
RUN apt-get -qq update && apt-get -q -y install \
  your-dependencies-here # e.g. libmysqlclient-dev
WORKDIR /app
COPY . .
RUN mkdir -p /build/lib && cp -R /usr/lib/swift/linux/*.so /build/lib
RUN swift build -c release && mv `swift build -c release --show-bin-path` /build/bin

# Production image
FROM ubuntu:16.04
RUN apt-get -qq update && apt-get install -y \
  libicu55 libxml2 libbsd0 libcurl3 libatomic1 \
  your-release-dependencies-here \ # e.g. libmysqlclient20
  && rm -r /var/lib/apt/lists/*
WORKDIR /app
COPY Config/ ./Config/
COPY Resources/ ./Resources/ # if you have Resources
COPY Public/ ./Public/ # if you have Public
COPY --from=builder /build/bin/myappbinary .
COPY --from=builder /build/lib/* /usr/lib/
EXPOSE 8080
CMD ["./myappbinary", "--env=production"]
{% endhighlight %}

Let's break that down. The first block of lines looks very much like your development Dockerfile. It's creating a build image, naming it `builder`, and installing your build dependencies. Then it copies your project folder into the image.

After that, it creates a folder `/build` inside that image, to store files that later need to be copied into your production image. In this case, it copies the Swift libraries (`/usr/lib/swift/linux/*.so`) into `/build/lib`, and then compiles a release build of your app and moves it into `/build/bin`.

The second block of lines is where your production image gets built. First, it pulls a standard Ubuntu 16.04 image, and then installs Swift's standard dependencies plus your own. The `&& rm -r /var/lib/apt/lists/*` line is a standard Docker optimisation that removes apt's install cache to save on space. Note that you'll need to keep the `\` at end of line after adding your dependencies.

Following this, it copies your `Config`, `Resources` and `Public` directories directly into the image. Add or remove directories as required. The last two `COPY` lines pull the Swift libraries and your app binary from the builder image.

Finally, port 8080 is opened and your app is told to run in production mode when the image is launched.

Make sure you change `myappbinary` to the actual name of your binary. If using a Vapor template, it will be called `Run` by default.

### Building your production image

The actual build command is now really easy.

```
docker build -t myproject:1.0.0 .
```

You can launch it locally using:

```
docker run -p 8080:8080 myproject:1.0.0
```

I will not cover deployment here. You will probably want to use `docker push` to send the image to a container registry, and then use `docker-compose`, Docker Swarm or Kubernetes to schedule your containers on your servers. You can even set up CI to build and test your production image on every commit. There is plenty of information out there on how to do this.

## Wrap up

Wasn't that easy now? With one command you can launch a mini Linux server on your laptop, and with another command you can create a minimal production-ready container image. If you've struggled with fabfiles, rsync, or who knows what then you'll appreciate the simplicity. Good luck, and enjoy Vapor.

### Feedback and further help

The very best place for help with Vapor and Docker is in the [Vapor Slack][vapor-slack] workspace, on the `#docker` or `#help` channels. You'll find me there too, under `@bygri`. I'm very happy to receive feedback and improvements to this article.

### Postscript

I found, after a while, that typing out the command to build and launch the development container was *really annoying*, especially if I don't remember what port it uses, so I invented a little cheat method.

At the top of each of my developmental Dockerfiles, I put the launch command in a comment, like so:

{% highlight Docker %}
# docker build -t myproject:dev -f Dockerfile-dev . && docker run -it -p 8080:8080 -v "$PWD":/app --privileged --rm myproject:dev
FROM swift:4.0.3
…
{% endhighlight %}

I then have this command aliased as `dkdev`:

```
eval "`head -n 1 Dockerfile-dev | cut -c 2-`"
```

So all I need to do is go into my project directory, type `dkdev` and it brings up my development container. Tada.


[vapor-home]: https://vapor.codes
[docker-home]: https://www.docker.com
[vapor-slack]: http://vapor.team
