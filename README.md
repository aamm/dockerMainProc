# dockerMainProc

A Bash script library to simplify writing Docker main process scripts.

## What problem does this library solve?

Each Docker container has a special process whose process ID is 1. The container is alive as long as this main process is alive, and such main process is necessarily the ENTRYPOINT or CMD parameter in the Dockerfile.

But such execution model is not suitable for some server programs. For instance, the default Tomcat startup script is a shell script that creates a JVM process in the background. Not using the default startup script and using a JVM call as the entrypoint is not a realistic option.

A more convenient solution is to use a front shell script to spawn a server process in the background. Such shell script should be the entrypoint and only die when the background process dies.

## Advantages of using dockerMainProc

The dockerMainProc library provides most of the common functions that an entrypoint shell script needs such as signal trapping and subprocess monitoring.

It also allows for shell scripts to pre-overwrite some methods to extend its behavior.

## Getting started with the dockerMainProc lib

Create your own shell script and source the dockerMainProc file. Let's say your script is called daemonScript:

```shell
#!/usr/bin/env bash

function startSubprocess() {
    echo "It works!" | nc -l 8080 &
}

source /dockerMainProc
```

The startSubprocess function overwrites a function defined in the dockerMainProc. In the example above the nc process simulates a web server that dies after the first call on port 8080. The script above should be the entrypoint. Here is an example of Dockerfile:

```
FROM ubuntu:14.04
ADD daemonScript /
ADD dockerMainProc /
RUN chmod u+x /daemonScript
ENTRYPOINT [ "/daemonScript" ]
```

All three files should be on the same directory.

```shell
user@host$ ls

daemonScript  Dockerfile  dockerMainProc
```
First we create an image called test

```shell
user@host$ docker build -t test .

Sending build context to Docker daemon 51.71 kB
Step 1 : FROM ubuntu:14.04
 ---> 89d5d8e8bafb
Step 2 : ADD daemonScript /
 ---> Using cache
 ---> be8f62b6bdd3
Step 3 : ADD dockerMainProc /
 ---> 5c290591cc10
Removing intermediate container f646a2d9cc5f
Step 4 : RUN chmod u+x /daemonScript
 ---> Running in 2dfc95567763
 ---> eb1880b7d3ef
Removing intermediate container 2dfc95567763
Step 5 : ENTRYPOINT /daemonScript
 ---> Running in 9c4b7652ef72
 ---> 0b124eb994fb
Removing intermediate container 9c4b7652ef72
Successfully built 0b124eb994fb
```

Next we create a container form this image:

```shell
user@host$ docker run -d --name test -p 8080:8080 test

243a13d38f6ed52ffbbae309cac12e6a0e1cee8e9241323f95f189bb7ec24307

user@host$ docker ps -a

CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                    NAMES
243a13d38f6e        test                "/daemonScript"     12 seconds ago      Up 11 seconds       0.0.0.0:8080->8080/tcp   test
```

We can see that the library outputs the child process PID on the stdout:

```shell
user@host$ docker logs test

Child PID: 6
```

This is useful for us to kill the container by terminating the child process. The most standard way to terminate a container is by issuing the stop command:

```shell
user@host$ docker stop test

test

user@host$ docker inspect test | grep ExitCode

       "ExitCode": 15,
```

The library issued the exit code 15 to tell us that this was the signal received. 15 is the code of the SIGTERM signal sent by "docker stop".

## Refusing signals

It is possible to refuse signals by overwriting callback functions. See the following example:

```shell
#!/usr/bin/env bash

function startSubprocess() {
    echo "It works!" | nc -l 8080 &
}

function receiveSIGINT() {
    echo "I refuse to die!"
}

source /dockerMainProc
```

The difference from the previous example is that now we are overwriting the receiveSIGINT function. The default implementation of receiveSIGINT calls the teardown function, which kills the subprocess. So any SIGINT signal (code 2) is simply ignored.

```shell
user@host$ docker start test

test

user@host$ docker kill --signal=SIGINT test

test

user@host$ docker logs test

Child PID: 6
Child PID: 6
I refuse to die!

user@host$ docker ps -a

CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                    NAMES
243a13d38f6e        test                "/daemonScript"     30 minutes ago      Up 21 seconds       0.0.0.0:8080->8080/tcp   test
```

As you can see. Now the container refuses all SIGINT signals. SIGINT is the signal that the operarting system sends to a process when the user presses Control+C. So ignoring SIGINT is useful to prevent users from destroying the container in a interactive session (for instance, using "docker run -it").

## Other extension mechanisms

It is possible to change the default behavior by pre-overwriting the functions bellow. For more information, please refer to the source code.

1. setupSubprocess - Set up the child process. The default implementation calls startSubprocess and collects the subprocess PID.
2. startSubprocess - Called by setupSubprocess to simply create a new background process.
2. checkSubprocess - Verifies if the background process is running.
3. teardown - Called whenever a signal arrives.
