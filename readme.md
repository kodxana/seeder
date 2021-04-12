# seeder

seeder is a crawler for the LBRY network, which exposes a list
of reliable nodes via a built-in DNS server.

Features:
* regularly revisits known nodes to check their availability
* bans nodes after enough failures, or bad behaviour
* accepts nodes down to v0.3.19 to request new IP addresses from,
  but only reports good post-v0.3.24 nodes.
* keeps statistics over (exponential) windows of 2 hours, 8 hours,
  1 day and 1 week, to base decisions on.
* very low memory (a few tens of megabytes) and cpu requirements.
* crawlers run in parallel (by default 24 threads simultaneously).


## Build

```
sudo apt-get install build-essential libboost-dev libssl-dev
make
```

## Use

Assumptions:

- lbrycrd will use the domain `seed.example.com` to find peer nodes
- you will be running this seeder on a server at domain `vps.example.com`

### Configure DNS

You will need two DNS records:

type | name             | value
-----|------------------| ---------------
NS   | seed.example.com | vps.example.com
A    | vps.example.com  | 1.2.3.4


Test your DNS records

```
$ dig -t NS seed.example.com

;; ANSWER SECTION
seed.example.com.   86400    IN      NS     vps.example.com.
```

### Disable systemd resolver (Ubuntu 18.04+)

You only need this if you want to run the seeder on port 53 and it's taken by
Ubuntu's resolved. Run the following to turn the resolver off and prevent
it from starting on reboot

```
sudo systemctl stop systemd-resolved.service
sudo systemctl disable systemd-resolved.service

```

### Open firewall port

For example, if using UFW, run `ufw allow 53`. Some VPS providers also
have their own firewall that you'll need to configure.

### Run the seeder

On the system vps.example.com, you can now run dnsseed:

```
./dnsseed -h seed.example.com -n vps.example.com
```

If you want the DNS server to report SOA records, please provide an
e-mail address (with the @ part replaced by .) using `-m`.


### Running as non-root

Typically, you'll need root privileges to listen to port 53 (name service).

One solution is using an iptables rule (Linux only) to redirect it to
a non-privileged port:

```
iptables -t nat -A PREROUTING -p udp --dport 53 -j REDIRECT --to-port 5353
```

If properly configured, this will allow you to run dnsseed in userspace, using
the -p 5353 option.

Another solution is allowing a binary to bind to ports < 1024 with setcap (IPv6 access-safe)

```
setcap 'cap_net_bind_service=+ep' /path/to/dnsseed
```

## Debugging

### Server-side

On the server, run `sudo tcpdump port 53`. This will show you all traffic on port 53. As
you send DNS queries, you should see `A` requests come in and response IPs go out.

- no incoming responses: DNS or firewall issues, or DNS request is cached client-side
- no responses: seeder is not running, or running on the wrong port, or broken
- empty responses: requested domain doesn't match configured domain in seeder

You can also look at the output of the running seeder. It looks like this

```
[21-04-12 19:30:49] 28/104 available (104 tried in 994s, 1 new, 30 active), 0 banned; 38 DNS requests, 1 db queries
```

- if # of DNS requests is not going up as you send them, then seeder is not getting your requests
- if DNS requests are increasing but db queries are not, then the -h domain doesn't match
- if seeder didn't find any nodes, then it can't contact the nodes it itself is seeded with

### Client-side

Try `dig +short seed.example.com`. If you get node IPs, your setup is working.

Other things to try

- `dig @1.2.3.4 seed.example.com` to bypass local DNS cache or incorrect DNS records
- `dig +trace seed.example.com` for detailed routing info

