Together the WG hooks` implement:

“Keep access to private/LAN networks outside the tunnel”, andca “kill switch” that rejects non-VPN traffic (except LAN + local).

### PostUp (runs when WG comes up)
#### 1) Capture your current default gateway (the non-VPN one)

`DROUTE=$(ip route | grep default | awk '{print $3}')`

Looks at ip route (main table), finds the default ... via <GW>, and stores that gateway IP in DROUTE. This is meant to be your “real” router / uplink gateway. (If there are multiple default routes, this can pick the first match.)

#### 2) Define RFC1918 “private” networks
```
HOMENET=192.168.0.0/16
HOMENET2=10.0.0.0/8
HOMENET3=172.16.0.0/12
```
These cover typical LAN/private address space.

#### 3) Add explicit routes for those private networks via your real gateway
```
ip route add $HOMENET3 via $DROUTE
ip route add $HOMENET2 via $DROUTE
ip route add $HOMENET  via $DROUTE
```
Purpose: if your WG config is doing a “full tunnel” (e.g., `AllowedIPs = 0.0.0.0/0`), WG often installs routes/defaults that would otherwise send everything into the tunnel. These specific routes force traffic to private networks to go via the normal LAN gateway instead (so you can still reach local subnets, printers, NAS, routers, etc.).

#### 4) Allow those private destinations in OUTPUT before the kill switch
```
iptables -I OUTPUT -d $HOMENET  -j ACCEPT
iptables -A OUTPUT -d $HOMENET2 -j ACCEPT
iptables -A OUTPUT -d $HOMENET3 -j ACCEPT
```
Adds `ACCEPT` rules for those private ranges in the `OUTPUT` chain (locally-generated traffic).

`-I` inserts at the top for `192.168.0.0/16`; the others are appended. Since the `REJECT` rule is appended after, all three `ACCEPT` rules end up before the `REJECT` anyway.

#### 5) Kill switch: reject non-local traffic that isn’t going through WG
```
iptables -A OUTPUT ! -o %i \
  -m mark ! --mark $(wg show %i fwmark) \
  -m addrtype ! --dst-type LOCAL \
  -j REJECT
```
`! -o %i` → packet is NOT going out the WireGuard interface (`%`i becomes `wg0`, etc.).

`-m mark ! --mark $(wg show %i fwmark)` → packet is NOT marked with WG’s fwmark (wg-quick uses this mark to identify traffic that should be routed via the WG policy rules).

`-m addrtype ! --dst-type LOCAL` → destination is NOT a local address (so it won’t block loopback/local sockets).

`-j REJECT` → block it.

Net effect: if the VPN is up, any outbound traffic that tries to “leak” out the normal interface instead of the WG tunnel gets rejected — except the RFC1918 private networks (which you explicitly allow) and purely local destinations.

### PreDown (runs when WG goes down)

This undoes what PostUp did.

#### 1) Recreate the same variables
```
HOMENET=192.168.0.0/16
HOMENET2=10.0.0.0/8
HOMENET3=172.16.0.0/12
```
#### 2) Remove the routes
```
ip route delete $HOMENET
ip route delete $HOMENET2
ip route delete $HOMENET3
```

Deletes those explicit private-network routes (from the main table). Note: if the route doesn’t match exactly what was added, deletion can fail. Some people use `ip route del ... via $DROUTE` or `ip route replace` on add to be more robust.

##### 3) Remove the kill switch rule
`iptables -D OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT`

#### 4) Remove the private-network ACCEPT rules
```
iptables -D OUTPUT -d $HOMENET  -j ACCEPT
iptables -D OUTPUT -d $HOMENET2 -j ACCEPT
iptables -D OUTPUT -d $HOMENET3 -j ACCEPT
```
Summary:

PostUp: preserves LAN access (routes + `ACCEPT`) and enforces a VPN kill switch `(REJECT` anything not going through WG).
PreDown: removes those routes and firewall rules.