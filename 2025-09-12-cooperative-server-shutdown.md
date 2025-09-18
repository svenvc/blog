# Cooperative Pharo server shutdown

Sept 12, 2025

One of the nice things about blogging is the interaction with readers. 
Norbert Hartl recently described an elegant technique to implement 
cooperative shutdown of a Pharo server using existing Zinc tools. 
I want to describe his idea here.

What is the issue? 
When a Pharo server process is under control of other tools (such as systemd), 
we obviously have control over our startup sequence, 
but when we are stopped, the standard flow is a hard kill of the process.

In many cases, when we are stopped by a hard kill, nothing bad happens. 
The server processes running inside Pharo just stop immediately and that’s it. 

But in some cases, like when we implement some form of persistency ourselves, 
or when we want to give existing client connections some time to finish their in flight requests, 
or when we want to close the network services we are using, we need to take control of the stop sequence.

Pharo and its VM do not provide any services to do this, 
so we’ll have to implement this ourselves. 
Norbert’s idea consists of 2 parts: 
using `ZnReadEvalPrintDelegate` to offer a way to activate a shutdown sequence inside the running image 
and adding a signal handler to our startup script to trap the necessary OS signals 
and invoke the REPL service to execute the actual code.

In our Smalltalk startup script we add the following.

```smalltalk
ZnReadEvalPrintDelegate startInServerOn: 42033.

Smalltalk at: #ShutdownSequence put: [
  'Starting ShutdownSequence...' crTrace.
  ZnServer default stopServer.
  ZnServer default hasOpenConnections
    ifTrue: [ 3 seconds wait ].
  ZnServer default closeConnections.
  'Completed ShutdownSequence successfully' crTrace.
  Transcript newLine; flush.
  Smalltalk snapshot: false andQuit: true ].
```

A special REST service, only accessible on the local network on port 42033, 
with a single endpoint, `/repl`, will accepts POST requests with Smalltalk code, 
execute them and return the result. 
For example:

```console
$ curl -X POST -H'Content-Type:text/plain' -d '42 factorial' http://localhost:42033/repl
1405006117752879898543142606244511569936384000000000
```

Now we can extend our startup shell script with a hook and install it. 
We need an alternative to our `pharo-ctl.sh` script, the simpler `pharo-launcher.sh`, 
that keeps waiting for its Pharo subprocess once it is launched. 
Here it is in its entirety.

```bash
#!/bin/bash

function usage() {
    cat <<END
Usage: $0 <script> <image>
    launch a Pharo server
Naming
    script       is used as unique identifier
    script.st    must exist and is the Pharo startup script  
    script.pid   will be used to hold the process id
    image.image  is the Pharo image that will be started
END
    exit 1
}

script_home=$(dirname $0)
script_home=$(cd $script_home && pwd)

script=$1
image=$2

echo Executing $0 $script $image
echo Working directory $script_home

if [ "$#" -ne 2 ]; then
    usage
fi

image="$script_home/$image.image"

if [ ! -e "$image" ]; then
    echo $image not found
    exit 1
fi

st_file="$script_home/$script.st"

if [ ! -e "$st_file" ]; then
    echo $st_file not found
    exit 1
fi

pid_file="$script_home/$script.pid"

vm=$script_home/pharo-vm/pharo
options="--headless"

pid=

function getpid() {
    if [ -e $pid_file ]; then
        pid=`cat $pid_file`
    else
        echo Pid file not found: $pid_file
        echo Searching in process list for $script
        # assume there is only one
        pid=`ps ax | grep $script | grep -v grep | grep -v $0 | awk '{print $1}'`
    fi
}

repl_url=http://localhost:42033/repl
shutdown_command='ShutdownSequence fork . true'

function shutdown_pharo() {
    echo "Asking Pharo to shut down"
    # send a shutdown command to the running image via REPL.
    # The image starts a shutdown procedure and should exit itself
    curl -s -X POST -H "Content-Type:text/plain" -H "Connection:close" -d "$shutdown_command" $repl_url
}

function signal_handler() {
    echo "Signal handler called"
    getpid
    shutdown_pharo
    # we wait until the image shuts down
    echo Waiting for $pid to exit
    wait $pid
    echo $pid did exit
    # clear the signal for EXIT because now we are exiting and don't want to repeat the handler
    trap - EXIT
    exit 0 
}

function register_signal_handler() {
    echo "Registering signal handler"
    # register the handler for normal EXIT, docker shutdown SIGTERM or SIGINT for abnormal exits
    trap 'signal_handler' EXIT SIGTERM SIGINT
}

function start() {
    echo Launching $script in background
    if [ -e "$pid_file" ]; then
        rm -f $pid_file
    fi
    echo $vm $options $image $st_file
    $vm $options $image $st_file 2>&1 >/dev/null &
    echo $! >$pid_file
}

register_signal_handler

start

echo Waiting for children

wait
```

After defining a couple of helper functions, 3 things happen:
- register our own code when certain signals arrive
- start the Pharo server in the background, noting its pid
- wait until the Pharo server subprocess stops

The `shutdown_pharo` function uses the REPL handler to politely ask Pharo to stop.

We’ll need an alternative systemd service definition as well.

```console
[Unit]
Description=Zinc HTTP Components Server NXT
After=network.target

[Service]
Type=simple
User=stfx
WorkingDirectory=/home/stfx/pharo13
ExecStart=/home/stfx/pharo13/pharo-launcher.sh run-zn zinc
KillMode=mixed
TimeoutStopSec=5
Restart=always

[Install]
WantedBy=default.target
```

We are using a `simple` type, a `mixed` kill mode and a stop timeout of `5` seconds.

Here is the status after a successful start.

```console
$ sudo systemctl status zinc-http-server-nxt
● zinc-http-server-nxt.service - Zinc HTTP Components Server NXT
     Loaded: loaded (/etc/systemd/system/zinc-http-server-nxt.service; alias)
     Active: active (running) since Fri 2025-09-12 15:47:56 CEST; 8min ago
   Main PID: 1117456 (pharo-launcher.)
      Tasks: 3 (limit: 1078)
     Memory: 93.3M (peak: 93.5M)
        CPU: 3.349s
     CGroup: /system.slice/zinc-http-server-nxt.service
             ├─1117456 /bin/bash /home/stfx/pharo13/pharo-launcher.sh run-zn zinc
             └─1117460 /home/stfx/pharo13/pharo-vm/lib/pharo --headless /home/stfx/pharo13/zinc.image /home/stfx/pharo13/run-zn>

Sep 12 15:47:56 audio359 systemd[1]: Started zinc-http-server-nxt.service - Zinc HTTP Components Server NXT.
Sep 12 15:47:56 audio359 pharo-launcher.sh[1117456]: Executing /home/stfx/pharo13/pharo-launcher.sh run-zn zinc
Sep 12 15:47:56 audio359 pharo-launcher.sh[1117456]: Working directory /home/stfx/pharo13
Sep 12 15:47:56 audio359 pharo-launcher.sh[1117456]: Registering signal handler
Sep 12 15:47:56 audio359 pharo-launcher.sh[1117456]: Launching run-zn in background
Sep 12 15:47:56 audio359 pharo-launcher.sh[1117456]: /home/stfx/pharo13/pharo-vm/pharo --headless /home/stfx/pharo13/zinc.image /home/stfx/pharo13/run-zn.st
Sep 12 15:47:56 audio359 pharo-launcher.sh[1117456]: Waiting for children
```

You can see the `pharo-launcher.sh` script reporting on the different steps it is taking, and finally waiting.

Now we can try out our cooperative shutdown.

```console
$ sudo systemctl stop zinc-http-server-nxt
$ sudo systemctl status zinc-http-server-nxt
○ zinc-http-server-nxt.service - Zinc HTTP Components Server NXT
     Loaded: loaded (/etc/systemd/system/zinc-http-server-nxt.service; alias)
     Active: inactive (dead) since Fri 2025-09-12 16:00:02 CEST; 2s ago
   Duration: 13.934s
    Process: 1117716 ExecStart=/home/stfx/pharo13/pharo-launcher.sh run-zn zinc (code=exited, status=0/SUCCESS)
   Main PID: 1117716 (code=exited, status=0/SUCCESS)
        CPU: 567ms

Sep 12 15:59:47 audio359 pharo-launcher.sh[1117716]: /home/stfx/pharo13/pharo-vm/pharo --headless /home/stfx/pharo13/zinc.image>
Sep 12 15:59:47 audio359 pharo-launcher.sh[1117716]: Waiting for children
Sep 12 16:00:01 audio359 systemd[1]: Stopping zinc-http-server-nxt.service - Zinc HTTP Components Server NXT...
Sep 12 16:00:01 audio359 pharo-launcher.sh[1117716]: Signal handler called
Sep 12 16:00:01 audio359 pharo-launcher.sh[1117716]: Asking Pharo to shut down
Sep 12 16:00:01 audio359 pharo-launcher.sh[1117752]: true
Sep 12 16:00:01 audio359 pharo-launcher.sh[1117716]: Waiting for 1117720 to exit
Sep 12 16:00:02 audio359 pharo-launcher.sh[1117716]: 1117720 did exit
Sep 12 16:00:02 audio359 systemd[1]: zinc-http-server-nxt.service: Deactivated successfully.
Sep 12 16:00:02 audio359 systemd[1]: Stopped zinc-http-server-nxt.service - Zinc HTTP Components Server NXT.
```

Again you can see the `pharo-launcher.sh` script reporting on the different steps it is taking, and finally ending.

In the Pharo log file we can see what happened from the image's standpoint.

```
Running startup script run-zn.st
Started a NeoConsoleTelnetServer(running 42031)
Started a ZnManagingMultiThreadedServer(running 42032 127.0.0.1)
Started a ZnManagingMultiThreadedServer(running 42033 127.0.0.1)
2025-09-12 15:59:36 005 943405 Server Socket Bound 0.0.0.0:8180
2025-09-12 15:59:36 006 039688 Started ZnManagingMultiThreadedServer HTTP port 8180
Started a ZnManagingMultiThreadedServer(running 8180)
2025-09-12 15:59:36 007 246440 Server Socket Bound 0.0.0.0:9123
2025-09-12 15:59:36 008 039688 Started ZnManagingMultiThreadedServer HTTP port 9123
Started a ZnManagingMultiThreadedServer(running 9123)

2025-09-12 15:59:39 009 812310 GET /random 200 64B 28ms
2025-09-12 15:59:40 010 613662 GET /random 200 64B 2ms
2025-09-12 15:59:41 011 179495 GET /random 200 64B 14ms

2025-09-12 15:59:47 012 981321 Connection Accepted 127.0.0.1
2025-09-12 15:59:47 013 942193 Request Read a ZnRequest(POST /repl) 3ms
2025-09-12 15:59:47 014 942193 Request Handled a ZnRequest(POST /repl) 4ms
2025-09-12 15:59:47 015 942193 Response Written a ZnResponse(200 OK text/plain;charset=utf-8 5B) 0ms
2025-09-12 15:59:47 016 942193 POST /repl 200 5B 4ms
2025-09-12 15:59:47 017 942193 Server Connection Closed 127.0.0.1

Starting ShutdownSequence...
2025-09-12 15:59:47 018 943405 Server Socket Released 0.0.0.0:8180
2025-09-12 15:59:47 019 028123 Stopped ZnManagingMultiThreadedServer HTTP port 8180
Completed ShutdownSequence successfully
```

After 3 `/random` requests, the REPL server, which is on a higher, more verbose log level, 
receive a POST request invoking the ShutdownSequence code. 
You can see the forked `ShutdownSequence` code stopping the server. 
This then completes the cooperative Pharo server shutdown.

If anything would go wrong, after a timeout of 5 seconds, 
systemd will again send SIGKILL, but to all processes, 
not just the main one as asked by the mixed option.