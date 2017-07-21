# unifi-docker

## Supported docker hub tags and respective `Dockerfile` links 
| Tag | Description |
| --- | --- |
| [`latest`, `stable` ](https://github.com/jacobalberty/unifi-docker/blob/master/Dockerfile ) | Tracks UniFi stable version - 5.4.19 as of 2017-07-17 |
| [`testing` ](https://github.com/jacobalberty/unifi-docker/blob/testing/Dockerfile ) | Tracks UniFi Testing version - 5.5.19 as of 2017-07-05 |
| [`oldstable` ](https://github.com/jacobalberty/unifi-docker/blob/oldstable/Dockerfile ) | Tracks UniFi Old Stable version - 5.3.11 as of 2017-06-23 |


These tags generally track the UniFi APT repository. That's why despite 5.5.19 being called stable it is still under the testing tag. We do lead the repository a little when it comes to pushing the latest version. The latest version gets pushed when it moves from `stable candidate` to `stable` instead of waiting for it to hit the repository.

In adition to these tags you may tag specific versions as well, for example `jacobalberty/unifi:5.4.19` will get you unifi 5.4.19 no matter what the current version is.

## Description

This is a containerized version of [Ubiqiti Network](https://www.ubnt.com/)'s Unifi Controller version 5.

The following options may be of use:

- Set the timezone with `TZ`
- Bind mount the `data` and `log` volumes

Example to test with

```bash
mkdir -p unifi/data
mkdir -p unifi/logs
docker run --rm -p 8080:8080 -p 8443:8443 -p 3478:3478 -p 10001:10001 -e TZ='Africa/Johannesburg' -v ~/unifi/data:/var/lib/unifi -v ~/unifi/logs:/var/log/unifi --name unifi jacobalberty/unifi:unifi5
```
## Adopting access points/switches/security gateway
### Layer 3 adoption

The default example requires some l3 adoption method. You have a couple options to adopt.

#### SSH Adoption
The quickest one off method is to ssh into the access point and run the following commands
```
mca-cli
set-inform http://<host_ip>:8080/inform
```
#### Other options

You can see more options on the (UniFi website)[https://help.ubnt.com/hc/en-us/articles/204909754-UniFi-Layer-3-methods-for-UAP-adoption-and-management]


### Layer 2 adoption
You can also enable layer 2 adoption through one of two methods.

#### host networking

If you launch the container using host networking \(With the `--net=host` parameter on `docker run`\) Layer 2 adoption works as if the controller is installed on the host.

#### Bridge networking

It is possible to configure the macvlan driver to bridge your container to the host's networking adapter. Specific instructions for this container are not yet available but you can read a write-up for docker at http://collabnix.com/docker-17-06-swarm-mode-now-with-macvlan-support/


## Beta users

There is now a new `beta` branch on github to support easier building of betas. This branch does not exist on the docker hub at all,
you must check it out from github.
You simply build and pass the build argument `PKGURL` with the url to the .deb file for the appropriate beta you wish to build. I believe
this will keep closest with the letter and spirit of the beta agreement on the unifi forums while still allowing relatively easy access to the betas.
This build method is the method I will be using for my own personal home network to test the betas on so it should remain relatively well tested.

If you would like to submit a new feature for the images the beta branch is probably a good one to apply it against as well.
I will be cleaing up the Dockerfile under beta and gradually pushing out the improvements to the other branches. So any major changes
should apply cleanly against the `beta` branch.

## Volumes:

### `/var/lib/unifi`

Configuration data

### `/var/log/unifi`

Log files

### `/var/run/unifi`

Run information

## Environment Variables:

### `TZ`

TimeZone. (i.e America/Chicago)

### `JVM_MAX_THREAD_STACK_SIZE`

used to set max thread stack size for the JVM

Ex:

```--env JVM_MAX_THREAD_STACK_SIZE=1280k```

as a fix for https://community.ubnt.com/t5/UniFi-Routing-Switching/IMPORTANT-Debian-Ubuntu-users-MUST-READ-Updated-06-21/m-p/1968251#M48264

## Expose:

### 8080/tcp - Device command/control

### 8443/tcp - Web interface + API

### 8843/tcp - HTTPS portal

### 8880/tcp - HTTP portal

### 3478/udp - STUN service

### 6789/tcp - Speed Test (unifi5 only)

### 10001/udp - UBNT Discovery

See [UniFi - Ports Used](https://help.ubnt.com/hc/en-us/articles/218506997-UniFi-Ports-Used)

## Multi-process container

While micro-service patterns try to avoid running multiple processes in a container, the unifi5 container tries to follow the same process execution model intended by the original debian package and it's init script, while trying to avoid needing to run a full init system.

Essentially, `dump-init` runs a simple shell wrapper script placed at `/usr/local/bin/unifi.sh`. `unifi.sh` executes and waits on the jsvc process which orchestrates running the controller as a service. The wrapper script also traps SIGTERM to issue the appropriate stop command to the unifi java `com.ubnt.ace.Launcher` process in the hopes that it helps keep the shutdown graceful.

Example seen within the container after it was started

```bash
$  docker exec -it ef081fcf6440 bash
# ps -e -o pid,ppid,cmd | more
  PID  PPID CMD
    1     0 /usr/bin/dumb-init -- /usr/local/bin/unifi.sh
    7     1 sh /usr/local/bin/unifi.sh
    9     7 unifi -nodetach -home /usr/lib/jvm/java-8-openjdk-amd64 -classpath /usr/share/java/commons-daemon.jar:/usr/lib/unifi/lib/ace.jar -pidfile /var/run/unifi/unifi.pid -procname unifi -outfile /var/log/unifi/unifi.out.log -errfile /var/log/unifi/unifi.err.log -Dunifi.datadir=/var/lib/unifi -Dunifi.rundir=/var/run/unifi -Dunifi.logdir=/var/log/unifi -Djava.awt.headless=true -Dfile.encoding=UTF-8 -Xmx1024M -Xms32M com.ubnt.ace.Launcher start
   10     9 unifi -nodetach -home /usr/lib/jvm/java-8-openjdk-amd64 -classpath /usr/share/java/commons-daemon.jar:/usr/lib/unifi/lib/ace.jar -pidfile /var/run/unifi/unifi.pid -procname unifi -outfile /var/log/unifi/unifi.out.log -errfile /var/log/unifi/unifi.err.log -Dunifi.datadir=/var/lib/unifi -Dunifi.rundir=/var/run/unifi -Dunifi.logdir=/var/log/unifi -Djava.awt.headless=true -Dfile.encoding=UTF-8 -Xmx1024M -Xms32M com.ubnt.ace.Launcher start
   31    10 /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java -Xmx1024M -XX:ErrorFile=/usr/lib/unifi/data/logs/hs_err_pid<pid>.log -Dapple.awt.UIElement=true -jar /usr/lib/unifi/lib/ace.jar start
   58    31 bin/mongod --dbpath /usr/lib/unifi/data/db --port 27117 --logappend --logpath logs/mongod.log --nohttpinterface --bind_ip 127.0.0.1
  108     0 bash
  116   108 ps -e -o pid,ppid,cmd
  117   108 [bash]
```

## Certificate Support

To use custom SSL certs, you must map a volume with the certs to /var/cert/unifi

They should be named:
```
cert.pem  # The Certificate
privkey.pem # Private key for the cert
chain.pem # full cert chain
```
For letsencrypt certs, we'll autodetect that and add the needed Identrust X3 CA Cert automatically.


## TODO

Future work?

- Don't run as root (but Unifi's Debian package does by the way...)
- Possibly use Debian image with systemd init included (but thus far, I don't know of an official Debian systemd image to base off)
