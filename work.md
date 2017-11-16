# Lab F

## Resources
[iptables doc](https://www.centos.org/docs/4/html/rhel-rg-en-4/s1-iptables-options.html#S3-IPTABLES-OPTIONS-MATCH-MODULES)

[iptables-extensions doc](http://ipset.netfilter.org/iptables-extensions.man.html)

## Getting Started


| Group # | Inside subnet   | Outside subnet |
| :-----: | :-------------: | :------------: |
| 16      | 192.168.16.0/24 | 10.16.0.0/20   |

### Connectiong to the Virtual Hosts

> Open three terminals and connect to the three different hosts.

```
sudo lxc-start -F -n inside-host
sudo lxc-start -F -n firewall
sudo lxc-start -F -n outside-host
# Shut down host with
shutdown -h now
```

### Getting Root Access

> Start bash with `sudo`

```
sudo bash
```

### Set up Interfaces

```
# inside
ifconfig eth0 192.168.16.2/24

# firewall
ifconfig eth0 192.168.16.1/24
ifconfig eth1 10.16.0.1/20 

# outside
ifconfig eth0 10.16.0.2/20
```

### Setting up Routing

```
# firewall
# make ip_forward is 1
cat /proc/sys/net/ipv4/ip_forward

# inside
route add default gw 192.168.16.1

# outside
route add default gw 10.16.0.1

# inside test
ping 10.16.0.2

# outside test
ping 192.168.16.2

# check routing table
route
```

## Building a Firewall

* **IMPORTANT!** only run `iptables` command on *firewall* host.
* Use `iptables-save` command to print the rules created so far.

### Blocking ICMP Requests

> First ping inside from outside

```
ping 10.16.0.2
PING 10.16.0.2 (10.16.0.2) 56(84) bytes of data.
64 bytes from 10.16.0.2: icmp_seq=1 ttl=63 time=0.096 ms
64 bytes from 10.16.0.2: icmp_seq=2 ttl=63 time=0.083 ms
64 bytes from 10.16.0.2: icmp_seq=3 ttl=63 time=0.087 ms
^C
--- 10.16.0.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1998ms
rtt min/avg/max/mdev = 0.083/0.088/0.096/0.012 ms
```

> Now block the connection with

```
# firewall
root@firewall:~# iptables -A FORWARD -p icmp --icmp-type echo-request -j DROP
```

> Test with ping again

```
root@inside-host:~# ping 10.16.0.2
PING 10.16.0.2 (10.16.0.2) 56(84) bytes of data.
^C
--- 10.16.0.2 ping statistics ---
168 packets transmitted, 0 received, 100% packet loss, time 168209ms
```

> Since the packet is dropped, we get no response and the command just hangs.

```
# check the current state
root@firewall:~# iptables -vL
Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target prot opt in  out  source    destination   
  168 14112 DROP   icmp --  any any  anywhere  anywhere icmp ech...
```
> We see the firewall dropped all 168 packets we sent with the ping command.

> We can then remove the rule with

```
# first find the line number
root@inside-host:~# iptables --line-numbers -L FORWARD
# then remove the rule
root@inside-host:~# iptables -D FORWARD line_number
```

> Removed from FORWARD

### Rejecting ICMP Requests

> We now create a rule that prevents the outside host from pining the inside one.

```
root@firewall:~# iptables -A FORWARD -d 192.168.16.2 -s 10.16.0.2 -p icmp --icmp-type echo-request -j REJECT
```

> Now we check that inside can ping outside, outside **cannot** ping inside and that firewall can ping both.

```
root@inside-host:~# ping 10.16.0.2
PING 10.16.0.2 (10.16.0.2) 56(84) bytes of data.
64 bytes from 10.16.0.2: icmp_seq=1 ttl=63 time=0.091 ms
64 bytes from 10.16.0.2: icmp_seq=2 ttl=63 time=0.071 ms
64 bytes from 10.16.0.2: icmp_seq=3 ttl=63 time=0.076 ms
^C
--- 10.16.0.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1998ms
rtt min/avg/max/mdev = 0.071/0.079/0.091/0.011 ms
```
```
root@outside-host:~# ping 192.168.16.2
PING 192.168.16.2 (192.168.16.2) 56(84) bytes of data.
From 10.16.0.1 icmp_seq=1 Destination Port Unreachable
From 10.16.0.1 icmp_seq=2 Destination Port Unreachable
From 10.16.0.1 icmp_seq=3 Destination Port Unreachable
^C
--- 192.168.16.2 ping statistics ---
3 packets transmitted, 0 received, +3 errors, 100% packet loss, time 1999ms
```
```
root@firewall:~# ping 10.16.0.2
PING 10.16.0.2 (10.16.0.2) 56(84) bytes of data.
64 bytes from 10.16.0.2: icmp_seq=1 ttl=64 time=0.058 ms
^C
--- 10.16.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.058/0.058/0.058/0.000 ms

root@firewall:~# ping 192.168.16.2
PING 192.168.16.2 (192.168.16.2) 56(84) bytes of data.
64 bytes from 192.168.16.2: icmp_seq=1 ttl=64 time=0.054 ms
^C
--- 192.168.16.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.054/0.054/0.054/0.000 ms
```

> We also try to ping the internal interface of the firewall and succeed since the rule is on the FORWARD chain and not the INPUT chain of the firewall host.

```
root@outside-host:~# ping 192.168.16.1
PING 192.168.16.1 (192.168.16.1) 56(84) bytes of data.
64 bytes from 192.168.16.1: icmp_seq=1 ttl=64 time=0.057 ms
64 bytes from 192.168.16.1: icmp_seq=2 ttl=64 time=0.053 ms
64 bytes from 192.168.16.1: icmp_seq=3 ttl=64 time=0.067 ms
^C
--- 192.168.16.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1998ms
rtt min/avg/max/mdev = 0.053/0.059/0.067/0.005 ms
```

> Since we are only blocking the FORWARD chain of the interface we are not preventing a ping flood attack on the interface itself which at this moment is the only gateway for out inside host.

> REJECT responds to the packet we send while the DROP does not. We can see the difference in the number of packets sent in our tests.

> REJECT takes up time on our machine since we need to send a response. While DROP makes the original sender uncertain of what happened. Without a REJECT the original sender might keep sending requests.

### Logging and Limits

> We create a REJECT rule that logs no more than 5 logs per minute of the rejected connections.

```
root@firewall:~# iptables -A FORWARD -d 192.168.16.2 -s 10.16.0.2 -p icmp --icmp-type echo-request -j NFLOG --nflog-prefix "Ping rejected by Jon:" -m limit --limit 5/minute
root@firewall:~# iptables -A FORWARD -d 192.168.16.2 -s 10.16.0.2 -p icmp --icmp-type echo-request -j REJECT

# run ping from outside then check

root@firewall:~# less +F /var/log/ulog/syslogemu.log
```
> We validate that our logfile only has five entries even though we let the ping command fail more than five times.

```
Nov 14 19:35:40 firewall Ping rejected by Jon: IN=eth1 OUT=eth0 MAC=00:16:3e:f9:02:01:00:16:3e:4a:5c:0a:08:00 SRC=10.16.0.2 DST=192.168.16.2 LEN=84 TOS=00 PREC=0x00 TTL=63 ID=34458 DF PROTO=ICMP TYPE=8 CODE=0 ID=1615 SEQ=2 MARK=0 
Nov 14 19:35:41 firewall Ping rejected by Jon: IN=eth1 OUT=eth0 MAC=00:16:3e:f9:02:01:00:16:3e:4a:5c:0a:08:00 SRC=10.16.0.2 DST=192.168.16.2 LEN=84 TOS=00 PREC=0x00 TTL=63 ID=34465 DF PROTO=ICMP TYPE=8 CODE=0 ID=1615 SEQ=3 MARK=0 
Nov 14 19:35:42 firewall Ping rejected by Jon: IN=eth1 OUT=eth0 MAC=00:16:3e:f9:02:01:00:16:3e:4a:5c:0a:08:00 SRC=10.16.0.2 DST=192.168.16.2 LEN=84 TOS=00 PREC=0x00 TTL=63 ID=34552 DF PROTO=ICMP TYPE=8 CODE=0 ID=1615 SEQ=4 MARK=0 
Nov 14 19:35:43 firewall Ping rejected by Jon: IN=eth1 OUT=eth0 MAC=00:16:3e:f9:02:01:00:16:3e:4a:5c:0a:08:00 SRC=10.16.0.2 DST=192.168.16.2 LEN=84 TOS=00 PREC=0x00 TTL=63 ID=34802 DF PROTO=ICMP TYPE=8 CODE=0 ID=1615 SEQ=5 MARK=0 
Nov 14 19:35:51 firewall Ping rejected by Jon: IN=eth1 OUT=eth0 MAC=00:16:3e:f9:02:01:00:16:3e:4a:5c:0a:08:00 SRC=10.16.0.2 DST=192.168.16.2 LEN=84 TOS=00 PREC=0x00 TTL=63 ID=35197 DF PROTO=ICMP TYPE=8 CODE=0 ID=1615 SEQ=13 MARK=0
```

#### *** MILESTONE ***

### Building a Firewall

> Flush all existing rules
```
root@firewall:~# iptables -F 
```

> **IMPORTANT:** Rule ordering matters!

### Default Policy

> We set the default policy for all chains (FORWARD, INPUT & OUTPUT) to `DROP`

```
root@firewall:~# iptables -P FORWARD DROP
root@firewall:~# iptables -P INPUT DROP
root@firewall:~# iptables -P OUTPUT DROP
root@firewall:~# iptables -L
Chain INPUT (policy DROP)
target     prot opt source               destination         

Chain FORWARD (policy DROP)
target     prot opt source               destination         

Chain OUTPUT (policy DROP)
target     prot opt source               destination         
```

> We cannot set the default to REJECT since it is an *extension* of the standard targets.

### Network Permissions

> Allow all traffic originating from the internal network to 

```
root@firewall:~# iptables -A INPUT -s 192.168.16.0/24 -i eth0 -j ACCEPT
root@firewall:~# iptables -A OUTPUT -d 192.168.16.0/24 -j ACCEPT

```

> Test by seeing if you can reach the internal network from the firewall, and check if you can reach the firewall from the internal.

```
root@firewall:~# ping 192.168.16.2

root@inside-host:~# ping 192.168.16.1
```

### Permitting a Service

> We want host on external to be able to connect with ssh to firewall host.

```
root@firewall:~# iptables -A INPUT -s 10.16.0.0/20 -p TCP --dport 22 -j ACCEPT
root@firewall:~# iptables -A OUTPUT -d 10.16.0.0/20 -p TCP --sport 22 -j ACCEPT
```

> Verify that you **can** ssh from external to firewall. Verify that you can**not** ssh from external to internal unless you ssh to the firewall first.

> We can remotely configure the firewall from basically anywhere.

> One problem with having the option of a remote ssh connection is that intruders may try and abuse the connection.

> Fail2Ban blocks IP that get to many failed password attempts when trying to SSH in.

### Stateful Filtering 

> We need to create a forwarding rule to allow our internal network to establish new connections with the outside, but prevent outside connections from establishing new connections with the inside. We can use the `state` module to decide which `NEW` and which `ESTABLISHED` connections are allowed.

```
root@firewall:~# iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT -m state --state NEW,ESTABLISHED
root@firewall:~# iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT -m state --state ESTABLISHED
``` 

> We test this by sending a ping from inside to outside and getting response and by sending ping from outside to inside and **no**t getting response.

### Opening Ports

> We want to allow TCP and UDP traffic to port 7 on the inside.

```
root@firewall:~# iptables -A FORWARD -i eth1 -o eth0 -p TCP --dport 7 -j ACCEPT
root@firewall:~# iptables -A FORWARD -i eth1 -o eth0 -p UDP --dport 7 -j ACCEPT
```

> Test using `telnet`

```
root@outside-host:~# telnet 192.168.16.2 7
Trying 192.168.16.2...
Connected to 192.168.16.2.
Escape character is '^]'.
Hello
Hello
```

### Blocking Ports

> We need to block all traffic from internal machines going to port 135 on the outside. Make sure this is first rule others might jump it to ACCEPT.

```
iptables -I FORWARD -i eth0 -o eth1 -p TCP --dport 135 -j REJECT
iptables -I FORWARD -i eth0 -o eth1 -p UDP --dport 135 -j REJECT
```

### Testing Setup

```
root@inside-host:~# telnet 10.16.0.2 135
Trying 10.16.0.2...
telnet: Unable to connect to remote host: Connection refused
```

```
root@outside-host:~# ssh 10.16.0.1
root@10.16.0.1's password: 
```

```
root@outside-host:~# telnet 192.168.16.2 7
Trying 192.168.16.2...
Connected to 192.168.16.2.
Escape character is '^]'.
Hello Milestone
Hello Milestone
^]
telnet> quit
Connection closed.
```
**Wat Explain Please ???**
> last three are not clear

### Dealing with SSH Brute-force Attack

> Create new chain called `SSH`
```
root@firewall:~# iptables -N SSH
```

> Redirect all ssh traffic on `INPUT` to `SSH`
```
root@firewall:~# iptables -A INPUT -s 10.16.0.0/20 -p TCP --dport 22 -j SSH
```

> Add rule that accepts packets from `ESTABLISHED` connections using `state` module.
```
root@firewall:~# iptables -A SSH -m state --state ESTABLISHED ACCEPT
```

> Add rules using `recent` module that block and log suspicious activity. That is more than 3 failed logins within 30 seconds. Only log the fourth connection and block all other connection within the 30 second window.

```
root@firewall:~# iptables -A SSH -m recent --name SSH_COUNTER --set 
root@firewall:~# iptables -A SSH -m recent --name SSH_COUNTER --rcheck --hitcount 5 --seconds 30 -j DROP
root@firewall:~# iptables -A SSH -m recent --name SSH_COUNTER --rcheck --hitcount 4 --seconds 30 -j NFLOG --nflog-prefix "SSH brute force attacker: "
root@firewall:~# iptables -A SSH -m recent --name SSH_COUNTER --rcheck --hitcount 4 --seconds 30 -j DROP
root@firewall:~# iptables -A SSH -j ACCEPT
```

> Default policies do not work on custom chains so the request never gets accepted.

```
Nov 14 22:23:21 firewall SSH brute force attacker:  IN=eth1 OUT= MAC=00:16:3e:f9:02:01:00:16:3e:4a:5c:0a:08:00 SRC=10.16.0.2 DST=10.16.0.1 LEN=60 TOS=00 PREC=0x00 TTL=64 ID=21484 DF PROTO=TCP SPT=42663 DPT=22 SEQ=441154379 ACK=0 WINDOW=29200 SYN URGP=0 MARK=0
```
> The attacker can spoof their source address. Which makes our counting useless since the new requests will arrive with new ips every time.

### Building Your Own Firewall

#### Rules to keep:
- The default policies that drop all unmatched packets.
- The rules that allow the internal network to start new connections to the outside but only allow incoming connections that are already established. 
#### Rules not to keep:
- Close port 7 for echo no reason to have this open to the world.
- Close the SSH ports from the outside to the firewall. Only want to access the firewall from within the network.
#### Additional Rules
- I use Skype at home so I would need to open the relevant ports
- I would also like to open the standard ports used for bittorrent.

## nmap: Detecting Server Capabilities

### Enumeration

```
root@outside-host:~# nmap -sn 10.16.0.0/20

Starting Nmap 6.40 ( http://nmap.org ) at 2017-11-16 09:41 UTC
mass_dns: warning: Unable to determine any DNS servers. Reverse DNS is disabled. Try using --system-dns or specify valid servers with --dns-servers
Nmap scan report for 10.16.0.1
Host is up (0.000044s latency).
MAC Address: 00:16:3E:F9:02:01 (Xensource)
Nmap scan report for 10.16.3.14
Host is up (0.000034s latency).
MAC Address: 00:16:3E:8E:DC:2A (Xensource)
Nmap scan report for 10.16.0.2
Host is up.
Nmap done: 4096 IP addresses (3 hosts up) scanned in 39.98 seconds
root@outside-host:~#
```

> We discover three hosts. .0.2 the outside machine running the command. .0.1 the firewall machine and .3.14 a new machine. We scanned **4096** addresses in **39.98** seconds.

### Service Discovery

```
root@outside-host:~# nmap -sV -T4 10.16.3.14

Starting Nmap 6.40 ( http://nmap.org ) at 2017-11-16 11:29 UTC
Nmap scan report for 10.16.3.14
Host is up (0.000014s latency).
Not shown: 993 closed ports
PORT     STATE SERVICE    VERSION
7/tcp    open  echo
22/tcp   open  ssh        (protocol 2.0)
53/tcp   open  domain     dnsmasq 2.68
80/tcp   open  http       Apache httpd 2.4.7 ((Ubuntu))
5222/tcp open  jabber     ejabberd (Protocol 1.0)
5269/tcp open  jabber     ejabberd
5280/tcp open  xmpp-bosh?
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at http://www.insecure.org/cgi-bin/servicefp-submit.cgi :
SF-Port22-TCP:V=6.40%I=7%D=11/16%Time=5A0D76B4%P=i686-pc-linux-gnu%r(NULL,
SF:2B,"SSH-2\.0-OpenSSH_6\.6\.1p1\x20Ubuntu-2ubuntu2\.8\r\n");
MAC Address: 00:16:3E:8E:DC:2A (Xensource)
Service Info: Host: localhost

Service detection performed. Please report any incorrect results at http://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 108.02 seconds
```
```
Starting Nmap 6.40 ( http://nmap.org ) at 2017-11-16 12:12 UTC
Nmap scan report for 10.16.3.14
Host is up (0.00013s latency).
Not shown: 96 closed ports
PORT    STATE         SERVICE VERSION
7/udp   open          echo
53/udp  open          domain  dnsmasq 2.68
68/udp  open|filtered dhcpc
123/udp open          ntp     NTP v4 (unsynchronized)
MAC Address: 00:16:3E:8E:DC:2A (Xensource)

Service detection performed. Please report any incorrect results at http://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 212.78 seconds
```

> `UDP` scanning is generally slower than the `TCP` scan because it can't determine the port state as easily as with the 3-way of `TCP`. `namap` cannot directly tell the difference between open port that eats the packets or one where a firewall drops all the incoming packets.

- `open` : Indicates that an application is actively accepting packets on the port.
- `close` : A closed port is visible to the probe but there is no application actively listening on it.
- `filtered` : Indicates that namp cannot determine if the port is open due to packets being filtered by firewall or routing before they reach the port.
- `unfiltered` : The port is accessible but nmap cannot determine if the port is open or closed.

| TCP / UDP | Port  | Service    | Version                       |
| :-------: | :---: | ---------- | ----------------------------- |
| TCP       | 7     | echo       | N/A                           |
| TCP       | 22    | ssh        | (protocol 2.0)                |
| TCP       | 53    | domain     | dnsmasq 2.68                  |
| TCP       | 80    | http       | Apache httpd 2.4.7 ((Ubuntu)) |
| TCP       | 5222  | jabber     | ejabberd (Protocol 1.0)       |
| TCP       | 5269  | jabber     | ejabberd                      |
| TCP       | 5280  | xmpp-bosh? | N/A                           |
| UDP       | 7     | echo       | N/A                           |
| UDP       | 53    | domain     | dnsmasq 2.68                  |
| UDP       | 68    | dhcp       | N/A                           |
| UDP       | 123   | ntp        | NTP v4 (unsynchronized)       |

### OS Discovery

> `nmap` could not answer with certainty what OS was running. But we can get its estimates using the `--fuzzy` flag.

```
nmap -O --fuzzy 10.16.3.14
Aggressive OS guesses: Netgear DG834G WAP or Western Digital WD TV media player (96%), Linux 2.6.32 (95%), Linux 2.6.32 - 3.9 (95%), Linux 3.8 (93%), Linux 3.1 (93%), Linux 3.2 (93%), AXIS 210A or 211 Network Camera (Linux 2.6) (92%), Linux 2.6.26 - 2.6.35 (92%), Linux 2.6.32 - 2.6.35 (92%), Linux 2.6.32 - 3.2 (92%)
```

done!