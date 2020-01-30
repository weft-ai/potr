# What is potr?

Potr is a script for *containerized build* and *development from source*. Potr is meant to handle
actions outside the container. Inside the container you should use a normal build tool. Potr only requires bash and docker with version >= 17.05. To use, add the potr script to somewhere in your path. 

The advantages of this approach are:

* To build the project you only need bash, the potr script and docker.
* In development, builds are incremental; unlike normal build containers you don't need to re-install from scratch every time.
* Everything is done from source, there are no dependencies on magical build containers.

## Quickstart

To use potr on a project add a potr.conf file in the root of your project. The simplest potr.conf file looks something like:

*potr.conf*
```
name="myproj"
```

The name specifies the prefix used for containers. By default potr will map the current working dir into /src in the container (this is configurable). Note that the potr idiom is to map the source dir into the container when it is run rather than copying it in at build time, so we can cache build artifacts between runs to speed up build time.

Potr also needs a build-container folder with a Dockerfile in it that will run your build. Potr will use this container as the environment for all of your dev commands. For example, a golang build container definition for a simple web service might look like:

*build-container/Dockerfile*
```
FROM golang:1.12.3-stretch
WORKDIR /src
CMD [ "go", "build", "-o", "/src/myproj" ]
```

Finally, create a Dockerfile in the root to generate a deployment container from the built sources. The container should copy in the build results left after running the build container. This is the final build result, a container with your compiled service that you can run later. For our golang example we might use:

*Dockerfile*
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

* Build a build container using ./build-container/Dockerfile.
* Hash the build container and store the hash in potr.sum. Potr will check the build container against this hash and fail if the hashes don't match. This ensures you have a deterministic build process. You should check potr.sum into your source tree, just as you would check in yarn.lock or go.sum.
* Run the build container with the current working directory mapped into /src in the container. This should build the project and generate the output into the current folder somewhere.

Note that potr prints all of the commands that it runs, as well as their output, so you can always look through the printed output to see what it ran.

To package the project into a container run:

```
potr deploy
```

This will:

* Run the full build process from before. Note that the creation of the build container should be a no-op because of docker caching, and our build process can store object files etc in the working directory to speed up the actual build.
* Create a container from ./Dockerfile named with the project name. This should pick up the artifacts from the build step.

# Extra commands

We probably have other commands we'll need to run while we are developing (yarn upgrade, etc), and many of them will have to run inside our build container. Potr will let us do that as well. For example, to start a bash session in the container we could run:

```
potr run bash
```

This will:

* Build the build container (again the docker cache should make it a no-op).
* Run the build container but execute the command we passed ("bash" in this case).

You could also run any other command:

```
potr run yarn install
potr run go get
etc.
```

# Custom commands

We can also make things easy by including custom scripts into our build container. For example, let's add a handy script to run our project in dev mode: 

*build-container/dev*
```
#!/bin/bash
/src/myproj --dev-mode -port 80
```

Let's update our build container Dockerfile to include it:

*build-container/Dockerfile*
```
FROM golang:1.12.3-stretch
COPY dev /bin
RUN chmod a+rx /bin/dev
WORKDIR /src
CMD [ "go", "build", "-o", "/src/myproj" ]
```

## Updating the build container

If you try to run the script now things will fail. This is because potr has saved the build container hash, but the build container has changed. To tell potr we meant to change the build container we run:

```
potr update
```

This will build update the potr.sum file to match this new container.

## Running our custom command

Now we can run our command inside the build container with:

```
potr run dev
```

Because we put dev on the path this is what should run. However, we still have a problem because our command is listening on port 80, but our container doesn't map port 80 outside the container! We'd really like to pass some extra arguments to docker to export the port. We can do that by adding a line to potr.conf:

*potr.conf*
```
name="myproj"
dev_args="-p 80:80"
```

Now when we run the "dev" command potr will add these extra args to the docker call so our port can be exposed externally.

# Configuration options

Potr.conf contains these configuration options:

 * name - The name used to identify containers
 * common_args - If specified these args are used for all commands. If not specified this defaults to "-v `pwd`:/src"
 * [cmd]_args - Extra arguments passed to command "cmd". Note that this only works for single line commands, today it doesn't handle passing parameters to the cmd (i.e. potr run dev -myarg won't use the docker arguments from dev_args).

# Potr commands

 * [no args] - Run the container with the default command.
 * run [cmd] - Run the given command.
 * update - Update the build container hash.
 * clean - Delete all docker images from this project (build containers, etc.) Note that this will clear all containers with the given name, so if you are running multiple versions of a project all of the build containers will be cleared. However, this should all be regeneratable, so it shouldn't be a big deal.
 * deploy - Run the build, then build the final container.

# Other notes

Some systems require docker to be run as root, and some do not. Potr will initially start by running docker without sudo, but if this fails it will try running it with sudo.

Potr accesses build containers based on their hash, so you can run multiple versions of the same project on the same machine without worrying about build containers getting mixed up.

# Example runs

Here is an example of what potr does if you run this command with our example project. (Note that the output of the commands have been stripped for clarity.)

```
potr
```

```
+ cd build-container
+ sudo docker build --iidfile /tmp/potr.4534 -t potr-temp-4534 .
 (Note, the file /tmp/potr.4534 will include the sha: sha256:5b50ca3a7f1cf13bfead55557bf6f44e185b74323d47308b96e876135578b20d)
+ sudo docker tag sha256:5b50ca3a7f1cf13bfead55557bf6f44e185b74323d47308b96e876135578b20d myproj-build-e876135578b20d
+ sudo docker rmi potr-temp-4534
+ sudo docker run -ti --rm -v `pwd`:/src sha256:5b50ca3a7f1cf13bfead55557bf6f44e185b74323d47308b96e876135578b20d
```

Or if you ran:

```
potr run dev
```

```
+ cd build-container
+ sudo docker build --iidfile /tmp/potr.4534 -t potr-temp-4534 .
 (Note, the file /tmp/potr.4534 will include the sha: sha256:5b50ca3a7f1cf13bfead55557bf6f44e185b74323d47308b96e876135578b20d)
+ sudo docker tag sha256:5b50ca3a7f1cf13bfead55557bf6f44e185b74323d47308b96e876135578b20d myproj-build-e876135578b20d
+ sudo docker rmi potr-temp-4534
+ sudo docker run -ti --rm -v `pwd`:/src -p 80:80 sha256:5b50ca3a7f1cf13bfead55557bf6f44e185b74323d47308b96e876135578b20d dev
```

# What is potr?

This is more details for those interested in the value proposition. Potr is a script for *containerized build* and *development* from *source*. It handles 

## Containerized build

The value of containerized builds is pretty well established. By encapsulating the build process
into a container a build can be made reproducable across a wide range of platforms so long as they
support docker. You can run a containerized build with nothing but docker itself, so the rest of the stuff is where potr is of value.

## Development

*Developing* an application in a container is a little trickier. Just putting your build in a 
container definition doesn't easily allow for incremental builds, running from local sources, running extra commands, running debuggers, etc. Most project get around this by running builds with Makefiles outside of the containers. Potr does that work for you.

## From source

XXX-as-code is a huge trend in development and ops for a good reason. However, most docker build processes are still using the golden-image-we-can't-deterministically-reproduce method for build containers. If your build container is appropriately specified you should be able to generate build containers from your source code. Potr makes that safe by hashing the resultant build container and ensuring it doesn't change unintentionally.