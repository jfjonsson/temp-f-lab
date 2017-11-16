# Prep Questions

## iptables

### Chains

- **INPUT:**
> If the packet if ment for this machine the packet gets passed to INPUT chain. If the packet passes though the INPUT chain it gets passed to any process waiting for that packet.

- **FORWARD:**
> If the kernel does not know how to forward packets or has that feature disabled the packet is dropped. But if enabled the packet will be set to the FORWARD chain and if accepted the packet will be sent to the intended interface.

- **OUTPUT:**
> Programs running on the machine that want to send network packets are sent to the OUTPUT chain, if they are accepted they are sent to their intended interface.

### Operators

- **Append:** `-A`
>Appends the iptables rule to the end of the specified chain
- **Insert:** `-I`
>Inserts a rule in a chain at a point specified by a user-defined integer value.
- **Delete:** `-D`
>Deletes a rule in a particular chain by number
- **List:** `-L`
>Lists all of the rules in the chain specified after the command.
- **Flush:** `-F`
> Flushes the selected chain, which effectively deletes every rule in the the chain.

### Filters

- `-p:`
> Sets the IP protocol for the rule
- `-s:`
> Sets the source for a particular packet
- `-d:`
> Sets the destination for a particular packet
- `-i:`
> Sets the incoming network interface
- `-o:`
> Sets the outgoing network interface
- `--sport:`
> Sets the source port of the packet
- `--dport:`
> Sets the destination port for the packet

### Jump Targets

- **ACCEPT:**
> Allows the packet to successfully move on to its destination or another chain.

- **DROP:**
> Drops the packet without responding to the requester. The system that sent the packet is not notified of the failure.

- **REJECT:**
> Sends an error packet back to the remote system and drops the packet.

- **LOG:**
> Logs all packets that match this rule. `/etc/syslog.conf` determines log location.

## nmap

### OSI Layer 3: Network Layer
- `-PO` IP Protocol Ping
> Discover hostsby sending IP packets with specific protocol number set in the IP header.
- `-sL` List Scan
> Scans for hosts on the network without sending any packets.

### OSI Layer 4: Transport Layer
- `-sT` TCP connect scan
> Is the default TCP scan. Ask OS to establish a connection to the target machine, same way browser does.
- `-sU` UDP scan
> Send UDP packet to target ports and see if they respond.

### OSI Layer 7: Application Layer
- `-sO` IP protocol scan
> Protocol allows us to determine which Application Layer protocol is being used.
- `-b` FTP bounce scan
> Check if FTP server supports proxy FTP connections that are abusable.

