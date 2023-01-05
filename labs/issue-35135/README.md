After more testing I'm able to reproduce this issue when messing with conntrack entries with Docker v20.10.22 on Ubuntu 20.04. I also tested my own fix for #44688, but with no success - it's another bug.

In my initial attempt, I used two `docker run` as the OP mentioned this issue could be reproduced in that way, but I still don't see it failing (although this could be due to not testing with Swarm before testing with `docker run`, or anything else creating an initial condition not reported here by the OP). Here's a viable repro case:

```bash
# Note that the order of following two commands does matter (see below).
$ docker run --rm --name client --log-driver=gelf --log-opt gelf-address=udp://127.0.0.1:12201  ubuntu /bin/sh -c 'COUNTER=1;while true; do date "+%Y-%m-%d %H:%M:%S.%3N" | xargs printf "%s %s | 51c489da-2ba7-466e-abe1-14c236de54c5 | INFO | HostingLoggerExtensions.RequestFinished    | $COUNTER\n"; COUNTER=$((COUNTER+1)); sleep 1; done'
# stack.yml being the OP's stack file.
$ docker stack deploy -c stack.yml test

# At this point, everything is fine.
$ sudo conntrack -L -p udp --dport 12201
udp      17 29 src=127.0.0.1 dst=127.0.0.1 sport=45640 dport=12201 [UNREPLIED] src=127.0.0.1 dst=127.0.0.1 sport=12201 dport=45640 mark=0 use=1
$ sudo journalctl -u docker.service # shows no error log from the gelf driver

# Messing with conntrack. After that point, it starts to break apart.
$ sudo conntrack -D -p udp --dport 12201
$ sudo journalctl -u docker.service
Jan 04 20:24:48 testvm dockerd[4025]: time="2023-01-04T20:24:48.499659258Z" level=error msg="Failed to log msg \"\" for logger gelf: gelf: cannot send GELF message: write udp 127.0.0.1:55026->127.0.0.1:12201: write: connection refused"
# Note that reply-src / reply-dst aren't the same anymore (come from docker_gwbridge):
# * 172.18.0.2 is gateway_ingress-sbox
# * 172.18.0.1 is docker_gwbridge's gateway
# Looks a bit like https://github.com/moby/moby/issues/44015#issuecomment-1257257614.
$ sudo conntrack -L -p udp --dport 12201
udp      17 28 src=127.0.0.1 dst=127.0.0.1 sport=45640 dport=12201 [UNREPLIED] src=172.18.0.2 dst=172.18.0.1 sport=12201 dport=45640 mark=0 use=1

# Stopping all containers, restarting dockerd and trying again.
$ docker service scale test_udpReceiver=0 && \
    docker stop client && \
    sudo systemctl restart docker.service
$ sudo conntrack -L -p udp --dport 12201
conntrack v1.4.5 (conntrack-tools): 0 flow entries have been shown.

$ # Restart the gelf logging container here (see 1st command).
$ docker service scale test_udpReceiver=1
# At this point, everything works as expected again.
$ sudo journalctl -u docker.service # shows no error log from the gelf driver
$ sudo conntrack -L -p udp --dport 12201
udp      17 29 src=127.0.0.1 dst=127.0.0.1 sport=45640 dport=12201 [UNREPLIED] src=127.0.0.1 dst=127.0.0.1 sport=12201 dport=45640 mark=0 use=1

# However, if both the 1st & 2nd commands in the repro or inverted, or if the
# test_udpReceiver service isn't scaled down before restarting the daemon or
# before rebooting, we directly jump to the failing case (= packets are NATed
# through docker_gwbridge). I can't find a workaround for that (ie. no 
# `conntrack -D` tricks or whatever). Actually, the issue probably comes from
# this:
$ Chain DOCKER-INGRESS (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    1   523 DNAT       udp  --  *      *       0.0.0.0/0            0.0.0.0/0            udp dpt:12201 to:172.18.0.2:12201
```
