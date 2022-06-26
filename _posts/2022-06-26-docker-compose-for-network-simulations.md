---
layout: post
title: Docker Compose for network simulations 
category: tutorial
tags:
  - docker
  - networking
---

I use [Docker Compose](https://github.com/docker/compose) to simulate mobile networks. Here is a tutorial about how to use this tool to simulate networks.

## Docker Compose version
There are 2 main versions of Docker Compose:
- Docker Compose v1, written in Python
- Docker Compose v2, written in Go

The Go version is farly better than the Python, at least for network engineering.
It allows to specify usefull [options](https://github.com/compose-spec/compose-spec/blob/master/spec.md) in `docker-compose.yaml`,
like `dns`, `healthcheck` and `profiles`. Additionnaly, this version is packaged for debian.

In this tutorial, I use the Go version (v2.6.0).

## Docker image creation
To instanciate a container, you must have a Docker image.
It is a good practice to use only [Docker Official Images](https://hub.docker.com/search?image_filter=official),
and images build by yourself (i.e. not using images you randomly found on DockerHub, since they may contain malwares).
For network engineering purpose, I recommend to base your images on [debian:stable-slim](https://hub.docker.com/_/debian).
Personally, I create [Git repositories on Github](https://github.com/louisroyer-docker), and automate the creation of images with Github Actions once a week,
so I can have an [up-to-date image ready in the DockerHub](https://hub.docker.com/u/louisroyer). 
Of course, if you do this, make sure to not put secrets in your image (even if you delete them in the next layer).
I create images that do just one specific thing, with only minimum packages installed and no debug tools.
If possible I use debian packages (I build [some of them](https://github.com/louisroyer-deb) myself),
but if there is no package it is also possible to build a software inside a `builder` image,
and import the binary using `COPY --from=builder …` to reduce the size of final image.
I will show in this tutorial how to use debug tools from another image created specifically for this.

Lets create some images for demonstration purpose:

	``` Docker
	FROM debian:bullseye-slim as router
	RUN  apt-get update -q && DEBIAN_FRONTEND=non-interactive apt-get install -qy iproute2 && rm -rf /var/lib/apt/lists/*
	COPY entrypoint.sh /usr/local/sbin/entrypoint.sh
	ENTRYPOINT ["entrypoint.sh"]
	ENV VOL_FILE=""
	ENV ROUTE6_NET=""
	ENV ROUTE6_GW=""
	ENV ROUTE4_NET=""
	ENV ROUTE4_GW=""
	CMD ["--help"]

	FROM debian:bullseye-slim as router-debug
	RUN  apt-get update -q && DEBIAN_FRONTEND=non-interactive apt-get install -qy iperf3 iputils-ping iproute2 tshark && rm -rf /var/lib/apt/lists/*
	ENTRYPOINT ["sleep"]
	CMD ["infinity"]
	```

The first image `router` will contains only the `iproute2` package.
In a real simulation, it could contains a specific software that will use `iproute2`, `iptables`,
and other network packages for example to create tunnels according to instructions given by the control plane.
The second image `router-debug` contains various tools for testing your network behaviour.

The file `entrypoint.sh` copied in `router` image is an executable file containing the following:

	``` bash
	#!/usr/bin/env bash
	set -e
	if [ "$1" == "--help" ]; then
		echo "Example entrypoint, use --start to run it" > /dev/stderr
		exit 0
	fi

	# Demonstration of volumes
	if [ -n "$VOL_FILE" ]; then
		cat "$VOL_FILE" > /dev/stderr
	fi

	if [ "$1" == "--start" ]; then
		echo "[$(date --iso-8601=s)] Starting router with arguments \"$@\"" > /dev/stderr

		# Demonstration of routes creation at runtime 
		if [[ -n "$ROUTE6_GW" && -n "$ROUTE6_NET" ]]; then
			ip -6 route add "$ROUTE6_NET" via "$ROUTE6_GW" proto static
		fi
		if [[ -n "$ROUTE4_GW" && -n "$ROUTE4_NET" ]]; then
			ip -4 route add "$ROUTE4_NET" via "$ROUTE4_GW" proto static
		fi

		# Remove default route
		ip -6 route delete default
		ip -4 route delete default

		# Display IP Addresses
		ip address list > /dev/stderr

		# Keep the container up 
		exec sleep infinity
	fi
	exit 1
	```



## Implementing the architecture
For this demonstration, I will define the following setup: three containers `R1`, `R2`, and `R3`.
`R1` and `R2` are connected on the same switch `S1`; `R2` and `R3` are connected on the same switch `S2`.

Lets create a `docker-compose.yaml` file:

	``` yaml
	# Debug containers template
	x-router-debug: &router-debug
	  restart: always
	  image: router-debug
	  build:
	    context: .
	    target: router-debug
	  # This capability will allow us to use iproute2
	  cap_add:
	    - NET_ADMIN
	  # Using different profile allow to
	  # not start unrequired debug container
	  profiles:
	    - debug

	# Routers container template
	x-router: &router
	  restart: always
	  image: router
	  build:
	    context: .
	    target: router
	  cap_add:
	    - NET_ADMIN
	  command: --start
	  # Required to enable routing
	  # It is not possible to change these values at runtime
	  sysctls:
	    - net.ipv6.conf.all.disable_ipv6=0
	    - net.ipv6.conf.all.forwarding=1
	    - net.ipv4.ip_forward=1

	services:
	  # Routers containers
	  R1:
	    <<: *router
	    container_name: R1
	    hostname: R1
	    dns:
	      # Change dns for this container
	      - 192.0.2.1
	      - 2001:db8::1
	    volumes:
	      # Our entrypoint will display this file
	      - ./Dockerfile:/Dockerfile:ro
	    environment:
	      # Indicate to the entrypoint which file to display
	      - VOL_FILE=/Dockerfile
	      - ROUTE6_NET=fd83:b442:5c7e::/80
	      - ROUTE6_GW=fd32:f7ff:393f::8000:0:2
	      - ROUTE4_NET=10.0.200.0/24
	      - ROUTE4_GW=10.0.100.130
	    networks:
	      # IP Address is assigned automatically from the pool
	      S1:
	  R2:
	    <<: *router
	    container_name: R2
	    hostname: R2
	    networks:
	      # This container use manually assigned IP Addresses
	      S1:
	          ipv4_address: 10.0.100.130
	          ipv6_address: fd32:f7ff:393f::8000:0:2
	      S2:
	          ipv4_address: 10.0.200.130
	          ipv6_address: fd83:b442:5c7e::8000:0:2
	  R3:
	    <<: *router
	    container_name: R3
	    hostname: R3
	    environment:
	      - ROUTE6_NET=fd32:f7ff:393f::/80
	      - ROUTE6_GW=fd83:b442:5c7e::8000:0:2
	      - ROUTE4_NET=10.0.100.0/24
	      - ROUTE4_GW=10.0.200.130
	    networks:
	      S2:

	  # Debug containers
	  R1-debug:
	   <<: *router-debug
	   container_name: R1-debug
	   network_mode: service:R1
	  R2-debug:
	   <<: *router-debug
	   container_name: R2-debug
	   network_mode: service:R2
	  R3-debug:
	   <<: *router-debug
	   container_name: R3-debug
	   network_mode: service:R3

	networks:
	  S1:
	    name: s1
	    enable_ipv6: true
	    driver: bridge
	    driver_opts:
	      com.docker.network.bridge.name: br-s1
	      com.docker.network.container_iface_prefix: s1-
	    ipam:
	      driver: default
	      config:
	        - subnet: 10.0.100.0/24
	          # ip_range is used for automatic ip allocation,
	          # I reserve half of the subnet for this.
	          # The other half is for manual allocation,
	          # I will use it for R2 and for the host (gateway)
	          # to be sure their IP addresses are available, even
	          # if R1 and R3 are started first
	          ip_range: 10.0.200.0/25
	          gateway: 10.0.100.129
	          # I use a /80 instead of a /48 for my ULA because
	          # of this issue: https://github.com/moby/moby/issues/40275
	        - subnet: fd32:f7ff:393f::/80
	          ip_range: fd32:f7ff:393f::/81
	          gateway: fd32:f7ff:393f::8000:0:1
	  S2:
	    name: s2
	    enable_ipv6: true
	    driver: bridge
	    driver_opts:
	      com.docker.network.bridge.name: br-s2
	      com.docker.network.container_iface_prefix: s2-
	    ipam:
	      driver: default
	      config:
	        - subnet: 10.0.200.0/24
	          ip_range: 10.0.200.0/25
	          gateway: 10.0.200.129
	        - subnet: fd83:b442:5c7e::/80
	          ip_range: fd83:b442:5c7e::/81
	          gateway: fd83:b442:5c7e::8000:0:1
	```

Now, we can run `docker compose --profile debug build` to build images (if you host images on DockerHub, you can run `docker compose --profile debug pull` instead).
The option `--profile debug` allows us to build images associated to the `debug` profile in addition to the `default` one.

We can then launch Docker Compose:

	``` terminal
	$ docker compose up -d
	[+] Running 5/5
	 ⠿ Network s2    Created 0.1s
	 ⠿ Network s1    Created 0.1s
	 ⠿ Container R3  Started 1.2s
	 ⠿ Container R1  Started 1.2s
	 ⠿ Container R2  Started 1.5s
	$ docker compose logs
	R3  | [2022-06-25T22:48:33+00:00] Starting router with arguments "--start"
	R3  | 1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
	R3  |     link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
	R3  |     inet 127.0.0.1/8 scope host lo
	R3  |        valid_lft forever preferred_lft forever
	R3  |     inet6 ::1/128 scope host
	R3  |        valid_lft forever preferred_lft forever
	R3  | 432: s2-0@if433: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
	R3  |     link/ether 02:42:0a:00:c8:01 brd ff:ff:ff:ff:ff:ff link-netnsid 0
	R3  |     inet 10.0.200.1/24 brd 10.0.200.255 scope global s2-0
	R3  |        valid_lft forever preferred_lft forever
	R3  |     inet6 fd83:b442:5c7e::1/80 scope global nodad
	R3  |        valid_lft forever preferred_lft forever
	R3  |     inet6 fe80::42:aff:fe00:c801/64 scope link tentative
	R3  |        valid_lft forever preferred_lft forever
	R1  | FROM debian:bullseye-slim as router
	R1  | RUN  apt-get update -q && DEBIAN_FRONTEND=non-interactive apt-get install -qy iproute2 && rm -rf /var/lib/apt/lists/*
	R1  | COPY entrypoint.sh /usr/local/sbin/entrypoint.sh
	R1  | ENTRYPOINT ["entrypoint.sh"]
	R1  | ENV VOL_FILE=""
	R1  | ENV ROUTE6_NET=""
	R1  | ENV ROUTE6_GW=""
	R1  | ENV ROUTE4_NET=""
	R1  | ENV ROUTE4_GW=""
	R1  | CMD ["--help"]
	R1  |
	R1  | FROM debian:bullseye-slim as router-debug
	R1  | RUN  apt-get update -q && DEBIAN_FRONTEND=non-interactive apt-get install -qy iperf3 iputils-ping iproute2 tshark && rm -rf /var/lib/apt/lists/*
	R1  | ENTRYPOINT ["sleep"]
	R1  | CMD ["infinity"]
	R1  | [2022-06-25T22:48:33+00:00] Starting router with arguments "--start"
	R1  | 1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
	R1  |     link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
	R1  |     inet 127.0.0.1/8 scope host lo
	R2  | [2022-06-25T22:48:33+00:00] Starting router with arguments "--start"
	R2  | 1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
	R2  |     link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
	R2  |     inet 127.0.0.1/8 scope host lo
	R2  |        valid_lft forever preferred_lft forever
	R2  |     inet6 ::1/128 scope host
	R2  |        valid_lft forever preferred_lft forever
	R2  | 430: s1-0@if431: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
	R2  |     link/ether 02:42:0a:00:64:82 brd ff:ff:ff:ff:ff:ff link-netnsid 0
	R2  |     inet 10.0.100.130/24 brd 10.0.100.255 scope global s1-0
	R2  |        valid_lft forever preferred_lft forever
	R2  |     inet6 fd32:f7ff:393f::8000:0:2/80 scope global nodad
	R2  |        valid_lft forever preferred_lft forever
	R2  |     inet6 fe80::42:aff:fe00:6482/64 scope link tentative
	R2  |        valid_lft forever preferred_lft forever
	R2  | 436: s2-0@if437: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
	R2  |     link/ether 02:42:0a:00:c8:82 brd ff:ff:ff:ff:ff:ff link-netnsid 0
	R2  |     inet 10.0.200.130/24 brd 10.0.200.255 scope global s2-0
	R2  |        valid_lft forever preferred_lft forever
	R2  |     inet6 fd83:b442:5c7e::8000:0:2/80 scope global nodad
	R2  |        valid_lft forever preferred_lft forever
	R2  |     inet6 fe80::42:aff:fe00:c882/64 scope link tentative
	R2  |        valid_lft forever preferred_lft forever
	R1  |        valid_lft forever preferred_lft forever
	R1  |     inet6 ::1/128 scope host
	R1  |        valid_lft forever preferred_lft forever
	R1  | 434: s1-0@if435: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
	R1  |     link/ether 02:42:0a:00:64:01 brd ff:ff:ff:ff:ff:ff link-netnsid 0
	R1  |     inet 10.0.100.1/24 brd 10.0.100.255 scope global s1-0
	R1  |        valid_lft forever preferred_lft forever
	R1  |     inet6 fd32:f7ff:393f::1/80 scope global nodad
	R1  |        valid_lft forever preferred_lft forever
	R1  |     inet6 fe80::42:aff:fe00:6401/64 scope link tentative
	R1  |        valid_lft forever preferred_lft forever
	```

To test our routing is right, lets start `R2-debug` and `R3-debug`:

	``` terminal
	$ docker compose up -d R2-debug R3-debug
	[+] Running 2/2
	 ⠿ Container R3        Running 0.0s
	 ⠿ Container R3-debug  Started 0.4s
	 ⠿ Container R2        Running 0.0s
	 ⠿ Container R2-debug  Started 0.4s
	```

So, we will start `tshark` on `R2`, and run a `ping` between `R3` and `R1`:

	``` terminal
	$ docker exec -it R3-debug ping fd32:f7ff:393f::1 -c 1
	PING fd32:f7ff:393f::1(fd32:f7ff:393f::1) 56 data bytes
	64 bytes from fd32:f7ff:393f::1: icmp_seq=1 ttl=63 time=0.083 ms

	--- fd32:f7ff:393f::1 ping statistics ---
	1 packets transmitted, 1 received, 0% packet loss, time 0ms
	rtt min/avg/max/mdev = 0.083/0.083/0.083/0.000 ms
	```

Be sure to not forget the `-it` in `docker exec` command, or you won't be able to kill it using `^C`
and might start multiple `tshark` instances (then you will have strange capture behaviours).
To avoid that, you can also run `tshark` from the host using `tshark -i br-s1 -i br-s2 -Y "icmpv6"`.

	``` terminal
	$ docker exec -it R2-debug tshark -i s1-0 -i s2-0 -Y "icmpv6"
	Running as user "root" and group "root". This could be dangerous.
	Capturing on 's1-0' and 's2-0'
	    1 0.000000000 fd83:b442:5c7e::1 ? fd32:f7ff:393f::1 ICMPv6 118 Echo (ping) request id=0x0043, seq=1, hop limit=63
	    2 0.000023245 fd32:f7ff:393f::1 ? fd83:b442:5c7e::1 ICMPv6 118 Echo (ping) reply id=0x0043, seq=1, hop limit=64 (request in 1)
	    3 -0.000008826 fd83:b442:5c7e::1 ? fd32:f7ff:393f::1 ICMPv6 118 Echo (ping) request id=0x0043, seq=1, hop limit=64
	    4 0.000026059 fd32:f7ff:393f::1 ? fd83:b442:5c7e::1 ICMPv6 118 Echo (ping) reply id=0x0043, seq=1, hop limit=63 (request in 3)
	```

We can see our routes are used because the `R3-debug` container is using the same network stack than `R3`.
Note: packets are not displayed in the right order, but using timestamps we can reorder as follow:

	``` text
	    3 -0.000008826 fd83:b442:5c7e::1 ? fd32:f7ff:393f::1 ICMPv6 118 Echo (ping) request id=0x0043, seq=1, hop limit=64
	    1 0.000000000 fd83:b442:5c7e::1 ? fd32:f7ff:393f::1 ICMPv6 118 Echo (ping) request id=0x0043, seq=1, hop limit=63
	    2 0.000023245 fd32:f7ff:393f::1 ? fd83:b442:5c7e::1 ICMPv6 118 Echo (ping) reply id=0x0043, seq=1, hop limit=64 (request in 1)
	    4 0.000026059 fd32:f7ff:393f::1 ? fd83:b442:5c7e::1 ICMPv6 118 Echo (ping) reply id=0x0043, seq=1, hop limit=63 (request in 3)
	```

We can also run `iperf3`:

	``` terminal
	$ docker up R1-debug -d
	[+] Running 2/2
	 ⠿ Container R1        Running 0.0s
	 ⠿ Container R1-debug  Started 0.2s
	$ docker exec -it R1-debug iperf3 -s
	-----------------------------------------------------------
	Server listening on 5201                       
	-----------------------------------------------------------
	Accepted connection from fd83:b442:5c7e::1, port 46730
	[  5] local fd32:f7ff:393f::1 port 5201 connected to fd83:b442:5c7e::1 port 46732
	[ ID] Interval           Transfer     Bitrate
	[  5]   0.00-1.00   sec  3.40 GBytes  29.2 Gbits/sec                  
	[  5]   1.00-2.00   sec  3.39 GBytes  29.1 Gbits/sec                  
	[  5]   2.00-3.00   sec  3.22 GBytes  27.7 Gbits/sec                  
	[  5]   3.00-4.00   sec  3.39 GBytes  29.1 Gbits/sec                  
	[  5]   4.00-5.00   sec  3.38 GBytes  29.0 Gbits/sec                  
	[  5]   5.00-6.00   sec  3.41 GBytes  29.3 Gbits/sec                  
	[  5]   6.00-7.00   sec  3.41 GBytes  29.3 Gbits/sec                  
	[  5]   7.00-8.00   sec  3.40 GBytes  29.2 Gbits/sec                  
	[  5]   8.00-9.00   sec  3.42 GBytes  29.4 Gbits/sec                  
	[  5]   9.00-10.00  sec  3.41 GBytes  29.3 Gbits/sec                  
	[  5]  10.00-10.00  sec   512 KBytes  13.8 Gbits/sec                  
	- - - - - - - - - - - - - - - - - - - - - - - - -
	[ ID] Interval           Transfer     Bitrate
	[  5]   0.00-10.00  sec  33.8 GBytes  29.1 Gbits/sec                  receiver
	-----------------------------------------------------------
	Server listening on 5201
	-----------------------------------------------------------
	^Ciperf3: interrupt - the server has terminated
	```

	``` terminal
	$ docker -it exec R3-debug iperf3 -c fd32:f7ff:393f::1
	Connecting to host fd32:f7ff:393f::1, port 5201
	[  5] local fd83:b442:5c7e::1 port 46732 connected to fd32:f7ff:393f::1 port 5201
	[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
	[  5]   0.00-1.00   sec  3.40 GBytes  29.2 Gbits/sec    0    897 KBytes
	[  5]   1.00-2.00   sec  3.39 GBytes  29.1 Gbits/sec    0   1.05 MBytes
	[  5]   2.00-3.00   sec  3.22 GBytes  27.7 Gbits/sec  270   1.70 MBytes
	[  5]   3.00-4.00   sec  3.39 GBytes  29.1 Gbits/sec    0   1.70 MBytes
	[  5]   4.00-5.00   sec  3.38 GBytes  29.0 Gbits/sec    0   1.70 MBytes
	[  5]   5.00-6.00   sec  3.41 GBytes  29.3 Gbits/sec  948   1.19 MBytes
	[  5]   6.00-7.00   sec  3.41 GBytes  29.3 Gbits/sec    0   1.19 MBytes
	[  5]   7.00-8.00   sec  3.40 GBytes  29.2 Gbits/sec  245   1.19 MBytes
	[  5]   8.00-9.00   sec  3.42 GBytes  29.4 Gbits/sec  282    855 KBytes
	[  5]   9.00-10.00  sec  3.41 GBytes  29.3 Gbits/sec    0    855 KBytes
	- - - - - - - - - - - - - - - - - - - - - - - - -
	[ ID] Interval           Transfer     Bitrate         Retr
	[  5]   0.00-10.00  sec  33.8 GBytes  29.1 Gbits/sec  1745             sender
	[  5]   0.00-10.00  sec  33.8 GBytes  29.1 Gbits/sec                  receiver

	iperf Done.
	```

For comparison, here is the output of the same test on the same machine but without using Docker:

	``` terminal
	$ iperf3 -c localhost
	Connecting to host localhost, port 5201
	[  5] local ::1 port 58788 connected to ::1 port 5201
	[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
	[  5]   0.00-1.00   sec  5.41 GBytes  46.5 Gbits/sec    0   3.25 MBytes
	[  5]   1.00-2.00   sec  5.87 GBytes  50.4 Gbits/sec    0   3.25 MBytes
	[  5]   2.00-3.00   sec  5.87 GBytes  50.5 Gbits/sec    0   3.25 MBytes
	[  5]   3.00-4.00   sec  5.85 GBytes  50.3 Gbits/sec    0   3.25 MBytes
	[  5]   4.00-5.00   sec  5.87 GBytes  50.5 Gbits/sec    0   3.25 MBytes
	[  5]   5.00-6.00   sec  5.89 GBytes  50.6 Gbits/sec    0   3.25 MBytes
	[  5]   6.00-7.00   sec  5.90 GBytes  50.7 Gbits/sec    0   3.25 MBytes
	[  5]   7.00-8.00   sec  5.90 GBytes  50.7 Gbits/sec    0   3.25 MBytes
	[  5]   8.00-9.00   sec  5.78 GBytes  49.7 Gbits/sec    0   3.25 MBytes
	[  5]   9.00-10.00  sec  5.76 GBytes  49.5 Gbits/sec    0   3.25 MBytes
	- - - - - - - - - - - - - - - - - - - - - - - - -
	[ ID] Interval           Transfer     Bitrate         Retr
	[  5]   0.00-10.00  sec  58.1 GBytes  49.9 Gbits/sec    0             sender
	[  5]   0.00-10.00  sec  58.1 GBytes  49.9 Gbits/sec                  receiver

	iperf Done.
	```

The bitrate is divided by two with the Docker architecture because we are using a router `R2` with two interfaces.
When the client send 1 packet, there are 2 packets for the host to handle (1 on each virtual bridge).
When simulating networks with Docker, you can use this formula: `max_container_bitrate = max_host_bitrate / number_of_virtual_bridges_traversed`
If your bitrate become to low because of the number of bridges traversed, a possible solution could be the use of Docker Swarm to deploy your
architecture using multiple hosts, although I have not tested it yet.

## Setting a DNS Resolver
As we have already seen, network stack is shared when `network_mode:container` is used (`network_mode: service:R3` for example).
This obviously include network interfaces and routing tables, but it also includes some network related files like `/etc/resolv.conf`.

By default, Docker Compose will populate this file with the following:

	``` terminal
	$ docker exec -it R2 cat /etc/resolv.conf
	nameserver 127.0.0.11
	nameserver 2001:4860:4860::8888
	nameserver 2001:4860:4860::8844
	options edns0 trust-ad ndots:0
	```

The IPv4 nameserver is a local resolver, managed by Docker,
that will resolve names using nameserver configured in host `/etc/resolv.conf`,
and if `dns` option is used in `docker-compose.yaml` with an IPv4, this value will be used for resolution.
IPv6 nameservers are Google nameservers by default. Specifing a `dns` option in `docker-compose.yaml` will replace these lines.

For example, here is the content of this file on `R1` (and `R1-debug`), where we have configured `dns` in `docker-compose.yaml`:

	``` terminal
	$ docker exec -it R1 cat /etc/resolv.conf
	nameserver 127.0.0.11
	nameserver 2001:db8::1
	options edns0 trust-ad ndots:0
	```

We can also modify this file manually:
	``` terminal
	$ docker exec -it R3 bash -c "echo nameserver 192.0.2.53 > /etc/resolv.conf"
	$ docker exec -it R3-debug cat /etc/resolv.conf
	nameserver 192.0.2.53
	```

The file is shared with `R3-debug`, even when modified at runtime.

## Modify container MAC addresses
Sometimes you may want to change MAC addresses of a container.
If you use `mac_address` in `docker-compose.yaml`, it is only possible to change it for the first interface
(`priority` can be used to be sure this is always the same interface).

This configuration option become useless if you want to modify more than one MAC address on a container.
It is possible to mount in a volume a script which will edit MAC address using `ip link set` then exec the original entrypoint.
You can run this kind of script by adding an `entrypoint` option in `docker-compose.yaml`.

## Conclusion
I hope this tutorial will help you with your usage of Docker Compose.
Docker Compose is a great tool for network engineering,
even if it might be difficult to use the first times.
