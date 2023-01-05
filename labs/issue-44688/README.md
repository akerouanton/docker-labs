## Repro

TODO:

* Try using `iptables -j TRACE`
* Try using nftables tracing
* Try using nf-ct-* tools

#### On `test_vm1`

```bash
$ sudo hping3 --udp --fast --keep --destport 9555 --data 1 10.8.4.3
```

#### On `test_vm2`

###### UDP

```bash
# Check connection tracking state
$ sudo conntrack -L -p udp --dport 9555
# Add a rule for tracing packets
$ sudo iptables -t raw -A PREROUTING -p udp --dport 9555 -j TRACE
$ sudo iptables -t nat -I PREROUTING -p udp --dport 9555 -j LOG --log-prefix "Debug UDP 9555 "

# Run the UDP server
$ docker run --rm -d --name server -p 9555:9555/udp raesene/ncat -nul 0.0.0.0 9555
# Run the UDP server with Swarm
$ docker service create --name server -p 9555:9555/udp raesene/ncat -nul 0.0.0.0 9555

# Check that the container receives packets with the source address of the Docker bridge
$ sudo nsenter -n -t $(docker inspect -f '{{ .State.Pid }}' $(docker ps | awk '{print $1}' | tail -n1)) tcpdump -nvi eth0 udp | head
# Check that docker-proxy is involved
$ sudo ss -aunp | grep docker-proxy
$ sudo iptables-trace 'port 9555'
```

Workaround:

```bash
$ sudo conntrack -D -p udp --dport 9555
$ sudo nsenter -n -t $(docker inspect -f '{{ .State.Pid }}' server) tcpdump -nvi eth0 udp | head
```

###### TCP

Client script:

```python
#!/usr/bin/env python3

from socket import *
import signal
import sys
import time

def signal_handler(signum, frame):
	global do_send, signals, sock
	do_send = not do_send
	signals += 1

	if signals >= 3:
		sys.exit(0)

	if do_send:
		print("Resuming http reqs...")
		sock = create_sock()
	else:
		print("Temporarily stopping http reqs...")
		sock.close()

def create_sock():
	sock = socket(family=AF_INET, type=SOCK_STREAM)
	sock.setsockopt(SOL_SOCKET, SO_REUSEADDR, 1)
	sock.bind(("0.0.0.0", 50000))
	sock.connect(("10.8.4.3", 80))
	return sock

do_send = True
signals = 0
sock = create_sock()

signal.signal(signal.SIGINT, signal_handler)

while True:
	if do_send:
		sock.send(b"GET / HTTP/1.0\nHost: 10.8.4.3\nConnection: keep-alive\n\n")
		print(sock.recv(1024).decode("utf-8"))
	time.sleep(1)
```

```bash
# Set up test_vm2 and start whoami server:
vagrant@testvm2$ curl --fail -Lo whoami.tgz https://github.com/traefik/whoami/releases/download/v1.8.7/whoami_v1.8.7_linux_amd64.tar.gz
vagrant@testvm2$ tar zxvf whoami.tgz
vagrant@testvm2$ sudo ./whoami -verbose

# Send a curl request every second to testvm2
vagrant@testvm1$ chmod +x testreq.py
vagrant@testvm1$ ./testreq.py

# Start traefik/whoami container on testvm2
vagrant@testvm2$ docker run --rm -d --name server -p 80:80/tcp traefik/whoami
```

## Test results (repro)

#### Before starting container

iptables-trace:

```
$ sudo iptables-trace --limit 'port 9555'
IN=eth1 OUT= SRC=10.8.4.2 DST=10.8.4.3 LEN=29 TOS=0x00 PREC=0x00 TTL=64 ID=26905 PROTO=UDP SPT=2151 DPT=9555 LEN=9 
	raw PREROUTING NFMARK=0x0 (0x52297e00)
		ACCEPT
	mangle PREROUTING NFMARK=0x0 (0x52297e00)
		ACCEPT
	mangle INPUT NFMARK=0x0 (0x52297e00)
		ACCEPT
	filter INPUT NFMARK=0x0 (0x52297e00)
		ACCEPT
	security INPUT NFMARK=0x0 (0x52297e00)
		ACCEPT
```

#### After starting container

```
$ sudo conntrack -L -p 17 | grep 9555
conntrack v1.4.5 (conntrack-tools): 8 flow entries have been shown.
udp      17 6 src=172.17.0.1 dst=172.17.0.2 sport=44256 dport=9555 [UNREPLIED] src=172.17.0.2 dst=172.17.0.1 sport=9555 dport=44256 mark=0 use=1
udp      17 15 src=172.17.0.1 dst=172.17.0.2 sport=48214 dport=9555 [UNREPLIED] src=172.17.0.2 dst=172.17.0.1 sport=9555 dport=48214 mark=0 use=1
udp      17 11 src=172.17.0.1 dst=172.17.0.2 sport=58970 dport=9555 [UNREPLIED] src=172.17.0.2 dst=172.17.0.1 sport=9555 dport=58970 mark=0 use=1
udp      17 30 src=10.8.4.2 dst=10.8.4.3 sport=2151 dport=9555 [UNREPLIED] src=10.8.4.3 dst=10.8.4.2 sport=9555 dport=2151 mark=0 use=1
udp      17 16 src=172.17.0.1 dst=172.17.0.2 sport=54062 dport=9555 [UNREPLIED] src=172.17.0.2 dst=172.17.0.1 sport=9555 dport=54062 mark=0 use=1
udp      17 29 src=172.17.0.1 dst=172.17.0.2 sport=49807 dport=9555 [UNREPLIED] src=172.17.0.2 dst=172.17.0.1 sport=9555 dport=49807 mark=0 use=1
```

iptables-trace:

```
$ sudo iptables-trace --limit 'port 9555'
IN= OUT=docker0 SRC=172.17.0.1 DST=172.17.0.2 LEN=29 TOS=0x00 PREC=0x00 TTL=64 ID=24402 PROTO=UDP SPT=42972 DPT=9555 LEN=9 
	raw OUTPUT NFMARK=0x0 (0x197aaa00)
		ACCEPT
	mangle OUTPUT NFMARK=0x0 (0x197aaa00)
		ACCEPT
	filter OUTPUT NFMARK=0x0 (0x197aaa00)
		ACCEPT
	security OUTPUT NFMARK=0x0 (0x197aaa00)
		ACCEPT
	mangle POSTROUTING NFMARK=0x0 (0x197aaa00)
		ACCEPT
```

#### After applying workaround

```
$ sudo iptables-trace --limit 'port 9555'
IN=eth1 OUT= SRC=10.8.4.2 DST=10.8.4.3 LEN=29 TOS=0x00 PREC=0x00 TTL=64 ID=57687 PROTO=UDP SPT=2151 DPT=9555 LEN=9 
	raw PREROUTING NFMARK=0x0 (0x791aae00)
		ACCEPT
	mangle PREROUTING NFMARK=0x0 (0x791aae00)
		ACCEPT
	mangle FORWARD NFMARK=0x0 (0x791aae00)
		ACCEPT
	filter FORWARD (#1) NFMARK=0x0 (0x791aae00)
		ip 0.0.0.0/0.0.0.0 -> 0.0.0.0/0.0.0.0 
		=> DOCKER-USER 
	filter DOCKER-USER (#1) NFMARK=0x0 (0x791aae00)
		=> RETURN
	filter FORWARD (#2) NFMARK=0x0 (0x791aae00)
		ip 0.0.0.0/0.0.0.0 -> 0.0.0.0/0.0.0.0 
		=> DOCKER-ISOLATION-STAGE-1 
	filter DOCKER-ISOLATION-STAGE-1 (#2) NFMARK=0x0 (0x791aae00)
		=> RETURN
	filter FORWARD (#4) NFMARK=0x0 (0x791aae00)
		ip 0.0.0.0/0.0.0.0 -> 0.0.0.0/0.0.0.0 
		=> DOCKER 
	filter DOCKER (#1) NFMARK=0x0 (0x791aae00)
		udp 0.0.0.0/0.0.0.0 -> 172.17.0.2/255.255.255.255 udp:{'dport': '9555'} 
		=> ACCEPT 
	security FORWARD NFMARK=0x0 (0x791aae00)
		ACCEPT
	mangle POSTROUTING NFMARK=0x0 (0x791aae00)
		ACCEPT
```

## Code analysis

* `pm.AppendForwardingTableEntry()` in mapper.go, line 155

Related issues/PRs:

* Issue #8795
* Issue #35135

Already fixed in: https://github.com/moby/moby/pull/43409

## Test results (fix)
