# A Pharo OCI container experiment

Dec 11, 2025

One way to deploy a Pharo server application is through an 
[OCI container specification](https://opencontainers.org), 
often called a Dockerfile.
This is a format that can be handled by 
[Docker](https://www.docker.com), 
[Podman](https://podman.io) 
and everything built on top of that, 
like [Kubernetes](https://kubernetes.io).
[Containerization](https://en.wikipedia.org/wiki/Containerization_(computing)) 
is a way to package an application in its own isolated computing environment.

A secondary goal is to improve efficiency by reducing the size of the container.
We will be using a two-stage Dockerfile build 
and apply some Pharo level magic to reduce our application to the bare minimum.

This is an example or experiment where we will be deploying 
[Zinc HTTP Components](https://github.com/svenvc/zinc) in its standard form,
much like we did in [Deploying the Zinc server demo](2025-09-02-deploy-zinc-server-demo.md).

The end result is that deploying the demo consists of 3 steps:
- get the specification and its associated files
- build an image
- start a container from the image

```console
$ git clone https://github.com/svenvc/pharo-containers.git
$ cd pharo-containers
$ podman build -t pharo_test .
$ podman run -d --name pharo_test -p 8080:8080 pharo_test
```

We can access the `/up` route to verify the Zn HTTP server works as expected.

```console
$ curl http://localhost:8080/up
Zinc HTTP Components 2.0 (Pharo/13.1) up 1 hour 43 minutes (allocated 99,811,328 bytes - 41.35 % free) [2025-12-11T14:16:24.610428Z]
```

The container/image build specification takes care of everything:
loading and installing Pharo, loading the latest Zinc code in the image,
reducing the image size, setting the start command.

There are 3 Pharo Smalltalk script files that define what happens
during build, code reduction and run time.

```smalltalk
"build.st"

Metacello new
  repository: 'github://svenvc/zinc/repository';
  baseline: 'ZincHTTPComponents';
  onConflict: [ :e | e useIncoming ];
  onUpgrade: [ :e | e useIncoming ];
  onWarning: [ :e | e load ];
  load: #('default' 'WebSocket').

Smalltalk garbageCollect.
```

Here we load one public GitHub repository using a baseline.
You can adapt this to load your own code.

```smalltalk
"remove.st"

TestCase allSubclassesDo: [ :e | e removeFromSystem].

Smalltalk garbageCollect.
```

In the code reduction phase, some additional removals can be specified.
Here we delete all test cases.

```smalltalk
"run.st"

(ZnServer defaultOn: 8080)
  logLevel: 1;
  start.

ZnLogEvent announcer
  when: ZnLogEvent 
  do: [ :event | event traceCr ]
  for: ZnServer default.
```

Finally, the run file defines what happens when running the final image as a container.
Here we start a default `ZnServer` on port 8080 and setup logging to stdout.

Now we can have a look at our `Dockerfile`:

```dockerfile
FROM ubuntu:latest AS builder

RUN apt-get update && apt-get install -y --no-install-recommends curl unzip ca-certificates

WORKDIR /pharo

COPY build.st run.st ./

RUN curl get.pharo.org | bash

RUN ./pharo Pharo.image st --save --quit build.st

RUN ./pharo Pharo.image clean --production
RUN ./pharo Pharo.iamge st --save --quit remove.st
RUN ./pharo Pharo.image eval --save NoPharoFilesOpener install


FROM ubuntu:latest AS final

WORKDIR /pharo

COPY run.st ./
COPY --from=builder /pharo/pharo ./
COPY --from=builder /pharo/pharo-vm ./pharo-vm
COPY --from=builder /pharo/Pharo.image ./

CMD ./pharo Pharo.image st --no-quit run.st
```

The first stage starts from a plain Ubuntu image.
We are using the lastest version, there are many variations possible.
First it installs some necessary tools.
Then it copies our `build.st` and `remove.st` from the host into its `/app` working directory.
Now we are ready to use [Pharo](https://github.com/pharo-project/pharo-zeroconf)'s 
[Zeroconf](https://get.pharo.org) facility to download and install Pharo.
Again, we are using the default version, know that it is possible to specify the version you want.
Invoking the `build.st` script while saving the image loads our own code.

Now we take a couple of steps to reduce the Pharo image size.

First we run the `clean` command line handler with the `--production` option.
This does a reasonable job of reducing memory by doing various cleanups.
See `ImageCleaner cleanUpForProduction` for more details.

Next we run our `remove.st` script for custom removals, thus removing all test cases.
Tests are not needed in production.

The Pharo 13 image size is in the low 50Mb, after these 2 steps, our image size is in the low 40Mb.
So we won about 10Mb, give or take.

Finally, we evaluate `NoPharoFilesOpener install` so that the Pharo image
no longer accesses its changes nor its sources file.
This prevents errors when starting an image without its `.changes` or `.sources` file.
In 99 % of normal production usage outside development this works fine.

Now we start the second stage from a new, clean Ubuntu image.
We use a similar `/pharo` working directory in which we copy the `run.st` from our host.
Now we copy just the bare minimal files that we need, not from the host, 
but from builder image we created in the first stage.

```console
sven@ds9:~/src/pharo-containers$ podman exec -it pharo_test bash
root@b0fc9d98eb46:/pharo# ls -lah
total 42M
drwxr-xr-x 1 root root 4.0K Dec 11 12:32 .
dr-xr-xr-x 1 root root 4.0K Dec 11 12:32 ..
-rw-r--r-- 1 root root  42M Dec 11 12:27 Pharo.image
-rwxr-xr-x 1 root root  365 Dec 11 12:24 pharo
drwxr-xr-x 4 root root 4.0K Dec 11 12:32 pharo-local
drwxr-xr-x 4 root root 4.0K Dec 11 12:27 pharo-vm
-rw-rw-r-- 1 root root  161 Dec 10 14:06 run.st
root@b0fc9d98eb46:/pharo# du -hs          
240M	.
root@b0fc9d98eb46:/pharo# du -hs pharo-vm/
199M	pharo-vm/
```

Our working directory is 240Mb, our Pharo image just 42Mb.
The Pharo VM takes up 200Mb !

Finally, the start command for running the image as a container is set.
For this we invoke our `run.st` script with the default `--no-quit` option.

Here are the sizes of some OCI images.
```console
sven@ds9:~/src/pharo-containers$ podman image ls
REPOSITORY                  TAG         IMAGE ID      CREATED       SIZE
localhost/pharo_test        latest      b221c6064daa  3 hours ago   332 MB
localhost/rpn_calculator    latest      279aae8dc8ea  12 days ago   137 MB
localhost/t3_sso_phx_test   latest      19375feee342  5 weeks ago   137 MB
localhost/sensor_monitor    latest      f06c58935017  5 weeks ago   413 MB
docker.io/library/postgres  latest      a38f9f77ff88  8 weeks ago   463 MB
docker.io/library/ubuntu    latest      97bed23a3497  2 months ago  80.6 MB
```

Our 330Mb image is about equal to 80Mb of Ubuntu + 240Mb of our working directory.

If only the Pharo VM were smaller ... maybe one day.
