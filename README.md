# Iptables cheatsheet
NETFilter is in kernel and is controlled by root user via **Iptables**
IPtables is stateful firewall, means we can keep state information between packets.

`iptables [-t table] -COMMAND chain matches -j target` 


## tables
- `filter`: default
    - built-in chains: INPUT, OUTPUT, FORWARD
- `nat`: 
    - built-in chains: PREROUTING, POSTROUTING, OUTPUT
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

## targets (jumps)
- Terminating
    - `ACCEPT`
    - `DROP`
- Non-Terminating
    - `REJECT`
        - `-j REJECT --reject-with tcp-rst`
    - `LOG --log-prefix`="incomming ssh:"  `--log-level` info : logs can be read with dmesg or from syslogd daemon
    - `SNAT`
    - `DNAT`
    - `MASQUADRE`: It is used with dynamic source ip nat, when the gateway router's ip is dynamic.
    - `LIMIT`
    - `RETURN`
    - `TEE --gateway` 10.0.0.1: traffic mirroring on the local subnet
    - `TOS`
    - `TTL`
    - `REDIRECT -to--ports` 8080: transparent proxy
        - REDIRECT target is only valid within the PREROUTING and OUTPUT chains of the nat table.

Some notes:
- Remember to allow loopback interface traffics.