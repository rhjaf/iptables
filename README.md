# Iptables cheatsheet
NETFilter is in kernel and is controlled by root user via **Iptables**
IPtables is stateful firewall, means we can keep state information between packets.

`iptables [-t table] -COMMAND chain matches -j target` 

### table of contents
- [Tables](#tables)
- [Chaines](#chaines)
- [Packet traversal](#packet-traverse)
- [Matches](#matches)
- [Commands](#commands)
- [Targets (Jumps)](#targets)
- [Saving rules](#save-rules-after-reboots)
- [IPSET](#ipset)

## tables
- `filter`: default
    - built-in chains: INPUT, OUTPUT, FORWARD
- `nat`: 
    - built-in chains: PREROUTING, POSTROUTING, OUTPUT
    - First, you should enable the routing process on your machine: `echo "1" > /proc/sys/net/ipv4/ip_forward` 
- `mangle`: packet alteration 
    - PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING
- `raw`:
    - to skip connection tracking, we add rules with `NOTRACK` target to the `raw` table
    - built-in chains: PREROUTING, OUTPUT
## chains : there are chaines in each table
- `INPUT`
- `OUTPUT`
- `FORWARD`
- `PREROUTING`(=DNAT,prot forwarding)
- `POSTROUTING`(=SNAT, MASQUERADE)
- There can be also user defined chains

## packet traverse:
- `Incoming`: network->host : prerouting(raw->mangle->nat) - input (mangle->filter)
- `Routed/Forwarded`: network->host(router)->?: prerouting(raw->mangle->nat) - forward(mangle->filter) - postrouting(mangle->nat)
- `Outgoing`: host->?: prerouting - output(raw->mangle->nat->filter) - postrouting(mangle->nat)

## matches:
- single use (general):
    - `-s` 192.168.0.0/24
    - `-d` dest_ip
    - `-p` protocol
        - `-p icmp --icmp-type echo-request`
    - `--sport` source_port
    - `--dport` dest_port
    - `-i` input_interface
    - `-o` output_interface
    - `-m` time
    - `-m` quota
    - `-m` limit
    
- multi options
    - iprange: `-m iprange --src-range` 10.0.0.10-10.0.0.39
    - `-m addrtype --dst-type` `UNICAST` | `MULTICAST` | `BROADCAST` 
    - multiports: `-m multiport --dports `80,443
    - TCP flags: 
        - for just syn: `-p tcp --syn`
        - for other tcp flags(`SYN`,`ACK`,`FIN`,`RST`,`URG`,`PSH`,`ALL`,`NONE`): `--tcp-flags mask comp`
    - Connection states (also stateless protocols not just TCP): `-m state --state` []
        - `NEW`: the first packet of connection
        - `ESTABLISHED`: packets are part of an existing connection
        - `RELATED`: packets are part of an existing connection and requesting a new connection (e.g: FTP)
        - `INVALID`: packets are not part of any existing connection
        - `UNTRACKED`: packets marked within the raw table with the `NOTRACK` target
    - Source MAC address: `-m mac --mac_source` 08:00:27:55:6f:20
    - DateTime: `-m time` []
        - `--datestart`|`--datestop` (YYYY[-MM[-DD[Thh[:mm[:ss]]]]])
        - `--timestart` | `--timestop` (hh:mm[:ss])
        - `--monthdays` | `--weekdays` [SAN,SAT,MON]
        - `--kerneltz`: by default, timezone is UTC so use  for system time. 
    - limit connections: `-m connlimit` []
        - `--connlimit-upto` 5: match if the number of existing connection is less than 5.
        - `--connlimit-above` 5: match if the number of existing connection is greater than 5.
    - limit: `-m limit`
        - `--limit` 1/minute | 2/s: maximum matches per time-unit.
        - `--limit-burst` 7:
        maximum matches before the above limit kicks in (default 5).
    - Dynamic database of recent black listed source IP address matches. `/proc/net/xt_recent/LIST_NAME`: 
        - add to list: `-m recent --name `blacklist` --set`
        - to update: `-m recent --name `blacklist` --update --seconds 60`
        - options:
            - --name
            - --set
            - --update: checks if the source ip address is in the list and updates the "last seen time".
            - --rcheck: checks if the source ip address is in the list and doesn't update the "last seen time".
            - --seconds 
    - quota: `-m quota --quota` 10000(bytes)
## commands
- `-A`
- `-I` FORWARD 3
    - by default it will be added to position 1
- `-D`
- `-R`
    - replaces a rule in selected chain
- `-F` FORWARD
- `-Z` FORWARD
    - zeroise the counter
- `-L`
- `-S`
- `-N`
    - creates a user defined chain
- `-X`
    - deletes a user defined chain
- `-P` FORWARD ACCEPT
    - set default polciy for chain

## targets
- Terminating
    - `ACCEPT`
    - `DROP`
- Non-Terminating
    - `REJECT`
        - `-j REJECT --reject-with tcp-rst`
    - `LOG --log-prefix`="incomming ssh:"  `--log-level` info : logs can be read with dmesg or from syslogd daemon
    - `SNAT`
        - `iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j SNAT --to-source 80.0.0.1`
    - `MASQUADRE`: It is used with dynamic source ip nat, when the gateway router's ip is dynamic.
        - `iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE`
    - `DNAT`(port-forwarding): permits connections from the internet to servers with private IP addresses inside LAN. The client connects to the public IP address of DNAT router which in turn redirects traffic to the private server and the server with private IP address stays invisible
        - `iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.0.20`
    - `LIMIT`
    - `RETURN`
    - `TEE --gateway` 10.0.0.1: traffic mirroring on the local subnet
    - `TOS`
    - `TTL`
    - `REDIRECT -to--ports` 8080: transparent proxy
        - REDIRECT target is only valid within the PREROUTING and OUTPUT chains of the nat table.

## save rules after reboots
- `iptables-save > ` myfirewall.config
- `iptables-restore` myfirewall.config
- `iptables-persistant` is a tool that will automatically loads ipv4s in /etc/iptables/rules.v4 and ipv6s in /etc/iptables/rules.v6 on system startup

## ipset
Manage more than one IPs
- Creating the set (`-exist` means it won't error if the set is already exist)(Note that another useful type is `nethash` or `hash:net` instead of `hash:ip`): `ipset -N ` myset `iphash -exist`
- Adding IPs to set: `ipset -A myset 3.2.1.4 -exist` (or you can use `add` instead of `-A`)
- Refer to the ipset in a rule: `iptables -A INPUT -m set --match-set myset src -j DROP`
- List set enteries: `ipset -L myset`
- Delete from set: `ipset del myset 4.3.2.1`
- Flush set: `ipset -F myset`
- Set maximum number of elements which can be stored in a set (default value:65535): `ipset create myset1 hash:ip maxelem 2048`
- Destroying the set: `ipset destroy myset`

## Extra notes:
- Remember to allow loopback interface traffics.
- When you create a custom chain, you can jump into it using `-j custom_chain` target. After the user-defined chain is traversed, control returns to the calling built-in chain, and matching continues from the next rule in the calling chain, unless the user-defined chain matched and took a terminating action on the packet. The `RETURN` target in a rule of a custom-chain makes processing resume back in the chain that called the custom chain. It can also be used inside a built-in chain. In this case no other rule will be inspected and packet executes the default `POLICY`. If you want to stop using your custom chain temporarily, you can simply delete the jump from the INPUT chain.

  ## Examples:

  <img src="IPtables.jpg" width="400">
  

  <img src="IPtables 2.jpg" width="400">
