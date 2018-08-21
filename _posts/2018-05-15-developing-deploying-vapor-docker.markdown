---
layout: post
title:  "Developing and deploying Vapor 3 with Docker"
date:   2018-05-15 09:00:00 +1100
---
Reading level: Beginner with [Docker][docker-home]. Experienced with [Vapor][vapor-home] and the Swift Package Manager (SPM).

I'll be using the latest currently available versions of all tools: Vapor 3.0.2, Swift 4.1 and Docker Community Edition 18.03.0 on macOS.

## Why develop and deploy Vapor with Docker?

Docker is an easy yet comprehensive implementation of containerisation. Containerisation, simplified, is the practice of encapsulating a single service (such as a Vapor app) inside its own standalone operating system. One container, one service. Once you've put your service in a container, it's portable and predictable; you can deploy it anywhere, and you know it'll respond to the same inputs with the same outputs.

Containerisation has entirely different benefits in development and in production. I'll start with production, because that's where the benefits are most intuitive.

Even a fairly straightforward modern web application generally requires several moving parts. You'll have your API, in this case built in Vapor. It may have any of the following: a front-end website, a database, a memory cache, a message platform, a search provider, logging and analytics. A container scheduler like [Docker's Swarm mode][docker-swarm-overview] allows you to use a single YAML file to define your application as a series of linked containers, and then deploy it to your hosting infrastructure. As there is no one-to-one mapping between physical servers and services, scaling your application is as simple as changing a number and re-deploying.

Now, since you are going to eventually define your production environment in terms of containers and services—why not do the same thing in development, too? Why not develop and test your application with all those other dependent services already in place? Docker lets you essentially replicate your production environment right on your desktop, without messing up your host operating system. Then when you deploy, it's simply a matter of scaling up.

> Taking containerisation a little bit further, you start to get into *microservices* territory. Do you present a different API to web clients than you do to mobile clients? Do you have a v1 and v2 API? Do you have a series of different APIs that share the same authentication mechanism? Would one of the facets of your application really be better off being written in Go? (heresy!) You might benefit from splitting your monolithic Vapor app into several smaller services that each focus on doing a single job well.

## Developing Vapor with Docker

We are going to build one trivial Vapor application with a couple of linked services: MySQL and Redis.

> Note! We will not be using Xcode to build and run. You can code in Xcode to take advantage of auto-complete, but we will be building, running and testing from *inside* a Linux container. This is so that our Vapor application can talk to the other containers as we go. But also, remember what I said about replicating our production environment? We're eventually going to end up on Linux. Why not start there?

You'll be spending much of your time at the command line, so here's a quick primer on the commands we'll use.

* `swift package init` creates a new project. You can also use the [Vapor Toolbox][toolbox-project-new] to do this.
* `swift build`, `swift test` and `swift run` do what you would expect, mirroring Xcode's commands.
* `docker-compose up` and `docker-compose down` will launch or remove a defined stack of linked services.
* `docker build` and `docker run` build and run a single container image, so we won't be using those very much, but `docker exec` performs a command inside an already-running container, so that'll be our primary method of interacting with Docker during development.

### Creating a project folder

Create an empty project folder somewhere, and open up a terminal window at that location. We're not going to build any Swift yet, so there is no need to create a Vapor project or a `Package.swift` at this stage.

### Defining our Vapor container image

In Docker, a *container* is a running instance of an *image*. When building an image, Docker uses a step-by-step recipe called a [Dockerfile][docker-dockerfile-ref]. It takes another image as a starting point, then runs the commands in that file to produce your new image. In development, your Dockerfile will simply define the base image and install required dependencies.

To keep it separate from your production Dockerfile later on, let's call it `Dockerfile-dev`.

```docker
FROM swift:4.1
RUN apt-get -qq update && apt-get -q -y install \
  tzdata \
  && rm -r /var/lib/apt/lists/*
```

When this Dockerfile is executed, the resulting image will be the official Swift image with the MySQL development headers added in.

Note that we are explicitly **not** building our Swift project in this Dockerfile. If you've messed with Docker before, you might expect to do so. We're not. We just want an environment to interactively build in.

> *What's with the long `RUN` command?* It's not strictly necessary in development, but a production Dockerfile ought to be as small in file size as possible. Therefore, we want to remove any unnecessary `apt` cache from our image. It's also a quirk of Docker that removing files in a later step won't actually free up any space in the image, so we have to chain the `rm` on the end of the `apt-get update`. Just roll with it for now.

### Defining our service stack

Now we need to tell Docker that we're going to build up a stack of services: one built on the fly from our Dockerfile, and then a pre-built database and memory cache, as you will recall.

To do this we define a [Compose file][docker-compose-ref], which is in YAML format. I don't tend to store production Compose files within a Vapor project, so I just call mine `docker-compose.yml`.

```yml
version: "3.3"
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile-dev
    image: api:dev
    environment:
      MYSQL_HOST: db
      MYSQL_USER: test
      MYSQL_PASSWORD: test
      MYSQL_DATABASE: test
      REDIS_HOST: redis
    ports:
      - 8080:8080
    volumes:
      - .:/app
    working_dir: /app
    stdin_open: true
    tty: true
    entrypoint: bash
  db:
    image: mysql:5
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_USER: test
      MYSQL_PASSWORD: test
      MYSQL_DATABASE: test
  redis:
    image: redis:alpine
```

This simple Compose file defines the three services, named `api`, `db` and `redis`. Let's read them from the bottom up.

`redis` is a service that simply pulls a pre-built official image named `redis:alpine` (this is Redis running on Alpine Linux).

`db` is a service that pulls the latest version of MySQL 5, and runs it with some environment variables set. The MySQL container knows to look for these environment variables, and sets its primary user, password and database accordingly.

`api` is the only one that contains a `build` key. That means that instead of pulling a pre-built image, Docker will build it using the specified Dockerfile, and name it `api:dev`. It also passes some environment variables, that our Vapor app will use to find MySQL and Redis. You'll note it opens port 8080 so you can interact with it from your browser. Let's leave the other configuration keys for the moment; I'll explain shortly.

Let's do a trial run. From inside your project directory, run:

```bash
docker-compose up --build
```

A whole lot of things should happen. First, Docker will create a virtual network linking the three containers. Then it'll build your `api` container, pulling `swift:4.1` and running our `apt-get` commands. Then it'll pull `mysql:5` and `redis:alpine` images from the public repository. Lastly, your console will get colourful as it boots all three together and prints out the logs from each.

Great! Now what? Well, nothing, because we don't have a Vapor application yet.

> Aside: there's a couple of ways to manage the stack. Either you can use `docker-compose up --build` to launch the stack in foreground mode, and then open a new terminal window to interact with it, or you can run `docker-compose up -d --build` to launch it in background mode. If you do this, use `docker-compose down` to bring the stack down again. If you stay in foreground mode, you can press `Ctrl-C` to bring it down.

### Building our application

Take another look at our Compose file, in particular these parts of the definition of the Vapor `api` service:

```yml
ports:
  - "8080:8080"
volumes:
  - .:/app
working_dir: /app
stdin_open: true
tty: true
entrypoint: bash
```

As explained earlier, `ports` means that port 8080 inside the container is mapped to port 8080 on your local computer, so if your Vapor app listens on port 8080, you'll be able to connect to it from your desktop.

Docker has the ability to mount your local filesystem as a *volume* inside the container. The beauty of a bind-mount is that you can edit your files in a macOS editor like Xcode or Sublime, while running `swift run` from a Bash prompt in your Linux container, operating on the exact same files.

We've defined a volume `.:/app` which means, mount the current directory (`.`) at the path `/app` in the container. We then set the `working_dir` to `/app`. `stdin_open` and `tty` allows us to open an interactive terminal inside our container, and `entrypoint: bash` means that when the container is started, by default it'll be waiting at a `bash` prompt at the working directory `/app`.

So, let's get building and running. I said above that we can edit in macOS, and build in Linux. Let's build in Linux.

### Attaching to the Linux prompt

If you still have your Docker stack running (see previous section), your `api` container will be sitting at a bash prompt right now. You can attach your current terminal to it, much like SSHing into a remote server, and issue commands.

First, let's find out the random identifier that Docker assigned. Enter `docker ps` to see a list of running containers. It should look like this:

```bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND
9629527e3434        api:dev             "bash"
9852d54a5a81        mysql:5             "docker-entrypoint.s…"
868c3c3e0e1a        redis:alpine        "docker-entrypoint.s…"
```

The one named `api:dev` is the one we want. We use the `docker attach` command to attach to it (press return twice after the command).

```bash
$ docker attach 9629
root@9629527e3434:/app#
```

Now we're sitting at that bash prompt. We are the `root` user, and we're in the `/app` directory. Run an `ls` command and you should see the contents of the project folder on your Mac.

### Writing a Vapor app

You'll need, of course, a Vapor project. For the purposes of proceeding with this tutorial, let's put in a placeholder app. Remove everything from your project besides the `Dockerfile` and `docker-compose.yml`, and then create the two files below.

(You can do this from your Mac, or from Linux; remember the project folder is mirrored in both.)

This should be your `Package.swift`:

```swift
// swift-tools-version:4.1
import PackageDescription

let package = Package(
  name: "app",
  dependencies: [
    .package(url: "https://github.com/vapor/vapor.git", from: "3.0.0"),
  ],
  targets: [
    .target(name: "Run", dependencies: ["Vapor"]),
  ]
)
```

And this should be `Sources/Run/main.swift`:

```swift
import Vapor

var services = Services.default()
services.register(MiddlewareConfig())

let router = EngineRouter.default()
router.get() { _ in "Hello Docker" }
services.register(router, as: Router.self)

try Application(
  config: .default(),
  environment: .detect(),
  services: services
).run()
```

> This is a very minimal Vapor 3 example that responds with `Hello Docker` to a `GET` request. No, it doesn't even use MySQL or Redis. A future version of this tutorial will do so.

When you're doing this for real, if you've installed Vapor's Toolbox, you can use [that method][toolbox-project-new] from your Mac, or you can use your new Swift container to do it! Run the below from the bash prompt you just attached to:

```bash
swift package init --type executable
```

Importantly, when you want to connect to your external MySQL and Redis services, you should use the values of the environment vars that you set in the Compose file earlier on: `$MYSQL_HOST`, `$MYSQL_USER`, `$MYSQL_PASSWORD`, `$MYSQL_DATABASE`, `$REDIS_HOST`. By using environment variables in place of hard-coded configuration values, you will be able to make changes to your service stack without needing to rebuild your Vapor app image.

### Building and running your Vapor app

If you're not familiar with using command-line Swift, here's a quick primer.

The equivalent of `Command-B` in Xcode is simply `swift build`. Your first build will, of course, fetch all your Swift package dependencies and then run a long build step. Future builds will make use of incremental debug building and be much faster.

`swift test` runs all unit tests. Note that you'll need to fill out `LinuxMain.swift` properly. Swift will build your test targets, run the tests and output the results.

`swift run` is the equivalent of Xcode's `Command-R`, and will launch your Vapor application. Press `Ctrl-C` to stop it again. Since we bound port 8080 of the container to port 8080 of our local computer, we will be able to reach our running Vapor application at `http://localhost:8080` using a browser or REST client.

> Note: Vapor is automatically configured to accept external connections. So to run your app you will actually need to use the full command below, assuming your binary is named `Run` (if you used the SPM project method, it'll be called `app`):
```
swift run Run serve -b 0.0.0.0
```

`swift package clean` will clean your build folder, and `swift package update` will update your package dependencies just like on macOS.

#### Side note: pre-populating your MySQL database

If you want, you can write a file of SQL commands that will be executed when the MySQL container is launched. Add the following key to the `db` service in your Compose file:

```yml
volumes:
  - ./fixtures.sql:/docker-entrypoint-initdb.d/init.sql
```

This will mount the file `fixtures.sql` in your local directory into the MySQL container in a special location that [MySQL knows to look for][docker-hub-mysql].

### Summary of development stage

By now, you should be all set up to build your Vapor application.

We defined a development environment with one Swift container, a MySQL container and a Redis container, all linked together in a stack which can be set up or torn down on demand.

We mounted our project inside the Swift container, then attached our terminal and used Swift on Linux to build, test and run our project. We used a macOS code editor to write our program and a web browser or [REST client][paw-home] to interact with the running app.

This foundation should be enough for you to go ahead and build your application. Next step: deploy it!

## Deploying Vapor in production with Docker

In development, we had an empty Swift container in which we mounted our project folder for building, testing and running in debug mode. We used pre-built MySQL and Redis images as backing services. Now it's time to package our Vapor application into its very own production-ready image.

> Note! There's a lot of information available online about deploying with Docker, so I will focus on preparing our Vapor app as a container image. Once that's done, you should do some further reading about deploying your stack.

### Building the production Vapor image

A central tenet of containerisation is that your production image should be as small as possible. When we build our Vapor project, we're going to end up with lots of unnecessary files in our `.build` folder. So, we're going to use a Docker feature called [*multi-stage builds*][docker-multi-stage-builds]. Essentially, it lets you create an image to build in, and then pull only a few files from that into a brand new image.

Make a new file called just `Dockerfile`. This is your production image build recipe.

```docker
# Build image
FROM swift:4.1 as builder
RUN apt-get -qq update && apt-get -q -y install \
  tzdata \
  && rm -r /var/lib/apt/lists/*
WORKDIR /app
COPY . .
RUN mkdir -p /build/lib && cp -R /usr/lib/swift/linux/*.so /build/lib
RUN swift build -c release && mv `swift build -c release --show-bin-path` /build/bin

# Production image
FROM ubuntu:16.04
RUN apt-get -qq update && apt-get install -y \
  libicu55 libxml2 libbsd0 libcurl3 libatomic1 \
  tzdata \
  && rm -r /var/lib/apt/lists/*
WORKDIR /app
COPY --from=builder /build/bin/Run .
COPY --from=builder /build/lib/* /usr/lib/
EXPOSE 8080
ENTRYPOINT ./Run serve -e prod -b 0.0.0.0
```

The first section begins just like your development Dockerfile, pulling a Swift 4.1 image and adding in build-time dependencies. Then it sets the working directory to `/app`, and the `COPY . .` line copies everything from your project directory into the image (no volume mounting any more). The first `RUN` command creates a temporary folder `/build` and copies the Swift static libraries in there. The second `RUN` command calls `swift build -c release` to create a release build of your binary and then copies it over to the temporary folder. Your app is now built.

The second section pulls a clean Ubuntu 16.04 image, which is only about 200MB, installs Swift's dependencies and the MySQL client libraries, and sets the working directory. It then uses `COPY --from=builder` to copy the app binary (`Run`, or whatever yours is called), and the Swift static libraries, out of the first section's image and into the production image. It then reports port 8080 as being open for connections, and defines your Vapor binary as the entrypoint, or the command to execute, when your image is run. Your container is all about the entrypoint. Docker will log all the output from your Vapor binary, and when the binary terminates, your container will terminate.

> Do you have additional resources you need to copy over, like Leaf files? Add lines before `EXPOSE` to bring in each folder you need, e.g. `COPY Resources/ ./Resources/`.

We've declared our production Dockerfile, now we just need to tell Docker to build it. Docker images are tagged as `name:version`. For example, the first version of your awesome app could be tagged `myawesomeapp:1.0.0`. Let's build and tag our app:

```bash
docker build -t myawesomeapp:1.0.0 .
```

Done. We now have an entirely self-contained image of a Vapor app that we can run at will. Let's do it:

```bash
docker run -p 8080:8080 myawesomeapp:1.0.0
```

You should now be able to reach your running app at `localhost:8080`. Note that it won't be able to connect to MySQL or Redis, because we are running the container on its own, rather than as part of the environment defined in our Compose file. Press `Ctrl-C` to quit it.

### Making a production Compose file

To give you an indication of how our production image slots into a Compose file, see below. The `api` service no longer has a build step, it simply pulls an image like `db` and `redis` do, opens a port, and configures environment variables.

```yml
version: "3.3"
services:
  api:
    image: myawesomeapp:1.0.0
    ports:
      - 80:8080
    environment:
      MYSQL_HOST: db
      MYSQL_USER: prod_user
      MYSQL_PASSWORD: secretpw
      MYSQL_DATABASE: prod
      REDIS_HOST: redis
  db:
    image: mysql:5
    environment:
      MYSQL_ROOT_PASSWORD: topsecretpw
      MYSQL_USER: prod_user
      MYSQL_PASSWORD: secretpw
      MYSQL_DATABASE: prod
  redis:
    image: redis:alpine
```

If you want to save this file, don't overwrite the development one, save it in a different folder or give it a different name, say `docker-compose-prod.yml`. You can test it out with:

```bash
docker-compose -f docker-compose-prod.yml up --build
```

Docker's built-in multi-node container scheduler, Docker Swarm, uses Compose files with the same syntax we are already used to, with [additional configuration options][docker-compose-deploy] for replication, resource limits and restart policy. I won't go into further detail here, because Docker's Swarm documentation is comprehensive.

The final part I have not covered is using a remote Docker repository. Docker has its own, Docker Hub, but there are other options too, including self-hosting. It's a bit like a GitHub for Docker images. You push a public or private image to a Docker repository, and then you can pull that image from another computer. This is pretty much required for production deployment.

### Summary of production stage

We took our Vapor app and created a Dockerfile to turn it into a containerised image. When we run our image, our Vapor app is immediately responding. We can tie that production image into a Compose file as if it were just another service. And when we're ready, we can deploy our app and its related services to a cluster of physical or virtual servers, scaling up or down as needed.

## Wrap up

That was a fairly comprehensive look at Docker throughout the application development lifecycle: from breaking ground on a new project, to building, testing and releasing to production. If you're feeling overwhelmed, or have any questions to ask or improvements to suggest, drop into the `#docker` channel on the [Vapor Discord server][vapor-discord]. You'll find me there too, under `@bygri`. I'm very happy to receive feedback and improvements to this article.

Thanks for reading!

[docker-compose-ref]: https://docs.docker.com/compose/compose-file/
[docker-compose-deploy]: https://docs.docker.com/compose/compose-file/#deploy
[docker-dockerfile-ref]: https://docs.docker.com/engine/reference/builder/
[docker-home]: https://www.docker.com
[docker-hub-mysql]: https://hub.docker.com/_/mysql/
[docker-multi-stage-builds]: https://docs.docker.com/develop/develop-images/multistage-build/
[docker-swarm-overview]: https://docs.docker.com/engine/swarm/
[paw-home]: https://paw.cloud
[toolbox-project-new]: https://docs.vapor.codes/3.0/getting-started/hello-world/
[vapor-discord]: http://vapor.team
[vapor-home]: https://vapor.codes
