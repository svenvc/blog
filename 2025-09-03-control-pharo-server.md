# Controlling a Pharo server process on Linux

September 3, 2025

Continuing from the posts of the last two days, I want to describe 
how I control the Pharo server process on my Linux server.
Most Linux versions use a framework called [systemd](https://systemd.io)
(see also [systemd's Wikipedia entry](https://en.wikipedia.org/wiki/Systemd)) 
to setup, manage and control so called daemons or server processes.
We want to integrate with systemd to be a nice Linux server citizen.
This will help sysadmin recognize us as a something they understand.

You need two things:
- a service specification
- some way to start and stop your server process referencing a PID file

Here is the contents of `zinc-http-server-nxt.service`, our service specification:

```console
[Unit]
Description=Zinc HTTP Components Server NXT
After=network.target

[Service]
Type=forking
User=stfx
WorkingDirectory=/home/stfx/pharo13
ExecStart=/home/stfx/pharo13/pharo-ctl.sh run-zn start zinc
ExecStop=/home/stfx/pharo13/pharo-ctl.sh run-zn stop zinc
PIDFile=/home/stfx/pharo13/run-zn.pid
Restart=always

[Install]
WantedBy=default.target
```

Although there exist many features and options, the basic idea should be clear.
We refer to a working directory where a script called `pharo-ctl.sh` lives
that can start and stop our Pharo server, leaving a PID file behind when started.

The `pharo-ctl.sh` script is a variant of 
[pharo-ctl.sh](https://github.com/svenvc/pharo-server-tools/blob/master/pharo-ctl.sh)
in my (old) [pharo-server-tools](https://github.com/svenvc/pharo-server-tools) project.

Invoked without arguments, the script explains how to use it:
```console
$ ./pharo-ctl.sh 
Executing ./pharo-ctl.sh
Working directory /home/stfx/pharo13
Usage: ./pharo-ctl.sh <script> <command> <image>
    manage a Pharo server
Naming
    script       is used as unique identifier
    script.st    must exist and is the Pharo startup script  
    script.pid   will be used to hold the process id
    image.image  is the Pharo image that will be started
Commands:
    start    start the server in background
    stop     stop the server
    restart  restart the server
    run      run the server in foreground
    pid      print the process id 
```

The script being the one I showed yesterday, the image the one we build with our packages.

In practice, you use a tool called `systemctl` to manage your service, once installed.
Here is the output from the `status` subcommand:
```console
$ sudo systemctl status zinc-http-server-nxt
● zinc-http-server-nxt.service - Zinc HTTP Components Server NXT
     Loaded: loaded (/etc/systemd/system/zinc-http-server-nxt.service; enabled; preset: enabled)
     Active: active (running) since Tue 2025-09-02 15:42:04 CEST; 24h ago
    Process: 820303 ExecStart=/home/stfx/pharo13/pharo-ctl.sh run-zn start zinc (code=exited, status=0/SUCCESS)
   Main PID: 820306 (pharo)
      Tasks: 2 (limit: 1078)
     Memory: 87.6M (peak: 104.0M)
        CPU: 8min 49.796s
     CGroup: /system.slice/zinc-http-server-nxt.service
             └─820306 /home/stfx/pharo13/pharo-vm/lib/pharo --headless /home/stfx/pharo13/zinc.image /home/stfx/pharo13/run-zn.>

Sep 02 15:42:04 audio359 systemd[1]: Starting zinc-http-server-nxt.service - Zinc HTTP Components Server NXT...
Sep 02 15:42:04 audio359 pharo-ctl.sh[820303]: Executing /home/stfx/pharo13/pharo-ctl.sh run-zn start zinc
Sep 02 15:42:04 audio359 pharo-ctl.sh[820303]: Working directory /home/stfx/pharo13
Sep 02 15:42:04 audio359 pharo-ctl.sh[820303]: Starting run-zn in background
Sep 02 15:42:04 audio359 pharo-ctl.sh[820303]: /home/stfx/pharo13/pharo-vm/pharo --headless /home/stfx/pharo13/zinc.image /home>
Sep 02 15:42:04 audio359 systemd[1]: Started zinc-http-server-nxt.service - Zinc HTTP Components Server NXT.
```

The top section details the current state, while the bottom section shows the tail of the log output.

Systemd will make sure our service is started automatically when the system boots,
and keeps running, restarting it when needed.

Note that this is a generic solution for any Pharo Smalltalk server/service process,
it just happens to be running a Zinc HTTP Components's server demo,
but it could really be anything at all.
