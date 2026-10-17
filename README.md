# nstor

Force selected Linux applications through Tor using a dedicated **network namespace + veth pair + transparent proxy**.

Applications launched with `nstor` see only a private network interface. All their TCP and DNS traffic is redirected to a local Tor instance. The processes themselves run as your normal user, not as root.

## Features

- Strong isolation: the application cannot see host interfaces, routes, or the real IP address
- Transparent Torification (no SOCKS configuration required inside the app)
- DNS leak protection via a dedicated dnsmasq stub
- Works with multi-process applications (browsers, Keybase, etc.)
- Privilege dropping: the target command runs as the invoking user
- Idempotent setup / clean teardown

## Prerequisites
- `iproute2` (`ip`)
- `iptables`
- `tor` (with `TransPort` and `DNSPort` enabled)
- `dnsmasq`
- Root privileges for setup (the application itself drops privileges)
  
- run once (and add to rc.local/a boot script)
```bash
sudo ip link add nstor-dummy type dummy 2>/dev/null || true
sudo ip addr add 10.200.1.1/32 dev nstor-dummy
sudo ip link set nstor-dummy up
```

### Example Tor configuration (`/etc/tor/torrc`)

```torrc
TransPort 10.200.1.1:9040
DNSPort 127.0.0.1:5353
VirtualAddrNetworkIPv4 10.192.0.0/10
AutomapHostsOnResolve 1
ClientOnly 1
SocksPolicy accept *
```

Adjust the TransPort address if you change the default subnet in the script.

## Installation
```bash   
sudo install -m 755 nstor /usr/local/bin/nstor
```
## Usage
```bash
# One-time (or after reboot) – create the namespace and firewall rules
sudo nstor setup

# Run any command inside the Tor namespace
sudo nstor firefox
sudo nstor curl https://check.torproject.org/api/ip
sudo nstor keybase

# Remove rules and namespace
sudo nstor clean
```

## Verify isolation
```bash
# Should report "torified"
ip netns identify $(pgrep -n firefox)

# Should return {"IsTor":true,...}
sudo nstor curl -s https://check.torproject.org/api/ip

# Should show only lo + veth (no eth/wlan)
sudo ip netns exec torified ip -4 addr
```
## Under the hood
1. A network namespace torified is created with a veth pair (10.200.1.0/24).
2. iptables REDIRECT rules send all TCP from the namespace to Tor’s TransPort and DNS to a local dnsmasq that forwards to Tor’s DNSPort.
3. The target command is executed inside the namespace as the original user.
4. The application has no route to the host’s real interfaces.

## Config
Edit the variables at the top of the script if you need different addresses or ports:
```
NS_NAME="torified"
HOST_IP="10.200.1.1"
NS_IP="10.200.1.2"
TRANS_PORT=9040
DNS_PORT=5353
```

## Limitations
ICMP (ping) is not carried by Tor.

UDP applications other than DNS are not transparently proxied.

The host’s Tor must be running and reachable on the configured TransPort/DNSPort before nstor setup.


## Distribution notes

### Slackware
The script works out of the box on Slackware.  
Tor is typically started with `/etc/rc.d/rc.tor`. No systemd units are required.

### Other distributions (Debian, Ubuntu, Fedora, Arch, etc.)

The core logic is portable. You only need to adapt service management and package names:

| Item | Slackware | Other distros |
|------|-----------|---------------|
| Start Tor | `/etc/rc.d/rc.tor start` | `systemctl start tor` or `rc-service tor start` |
| Tor package | `tor` | `tor` (sometimes `tor-geoip` extra) |
| dnsmasq | `dnsmasq` | `dnsmasq` |
| iptables | `iptables` | `iptables` or `iptables-nft` |
| iproute2 | `iproute2` | `iproute2` |

#### Required changes
1. **Install the packages**
Debian/Ubuntu  
```bash
sudo apt install tor dnsmasq iptables iproute2
```
Fedora
```bash
sudo dnf install tor dnsmasq iptables iproute
```
Arch
```bash
sudo pacman -S tor dnsmasq iptables iproute2
```
2. Make sure Tor is running with the TransPort / DNSPort lines shown above before calling nstor setup.
3. nftables-only systems
If iptables is not available, either install the iptables compatibility package or translate the few REDIRECT / MASQUERADE rules to nft.
4. SELinux / AppArmor
On enforcing systems you may need a local policy allowing ip netns exec and dnsmasq to bind the veth address. Slackware has neither by default.
