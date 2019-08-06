# What is potr?

Potr is a script for *containerized build* and *development* from *source*. Potr is meant to handle
actions outside the container. Inside the container you should use a normal build tool. Potr only requires bash and docker with version >= 17.05. To use, add the potr script to somewhere in your path. 

## Quickstart

To use potr on a project add a potr.conf file in the root of your project. An example potr.conf file might be:

*potr.conf*
```
name="myproj"
```

Also create a build-container folder and put a Dockerfile in it that will run your build assuming the source is in /src. For example, a golang build container might look like:

*build-container/Dockerfile*
```
FROM golang:1.12.3-stretch
WORKDIR /src
CMD [ "go", "build", "-o", "/src/myproj" ]
```

Finally, create a Dockerfile in the root to generate a deployment container from the built sources.  For our golang example we might use:

*./Dockerfile*
```
FROM alpine:3.10.1
COPY myproj /bin
CMD [ "/bin/myproj" ]
```

Now to build the project, in the project root run:

```
potr
```

This will:

* Build a build container using ./build-container/Dockerfile
* Hash the build container and its config and store the hash in potr.sum. Potr will guarantee all future runs will generate an identical build container, removing the need to check the build container into a repo, so long as the container build is deterministic.
* Run the build container with the current folder mapped into /src in the container (this can be overridden or agumented using build_args in potr.conf). This should build the project and generate the output into the current folder somewhere.

To package the built project into a container run:

```
potr deploy
```

This will:

* Repeat all the steps in the build process. However, because of docker's layer cache the container build process should be a no-op.
* Create a container from ./Dockerfile named with the project name. This should pick up the artifacts from the build step.

To run a custom operation in the build container run:

```
potr build yarn install
```

This will:

* Run the build process (again the docker cache should make it a no-op).
* Run the build container but pass in the arguments passed to yarn install (i.e. docker run ... myproj-build-347ft1 yarn install)

If you update your build container potr will fail because it no longer matches your locked version. To update the locked version run:

```
potr update
```

This will build and hash the new container and store the new hash in potr.sum.

# What is potr?

Potr is a script for *containerized build* and *development* from *source*. It handles 

## Containerized build

The value of containerized builds is pretty well established. By encapsulating the build process
into a container a build can be made reproducable across a wide range of platforms so long as they
support docker. You can run a containerized build with nothing but docker itself, so the rest of the stuff is where potr is of value.

## Development

*Developing* an application in a container is a little trickier. Just putting your build in a 
container definition doesn't easily allow for incremental builds, running from local sources, running extra commands, running debuggers,  etc. Most project get around this by running builds with Makefiles outside of the containers. Potr does that work for you.

## From source

XXX-as-code is a huge trend in development and ops for a good reason. However, most docker build processes are still using the golden-image-we-can't-deterministically-reproduce method for build containers. If your build container is appropriately specified you should be able to generate build containers from your source code. Potr makes that safe by hashing the resultant build container and ensuring it doesn't change unintentionally.