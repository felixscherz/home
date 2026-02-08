# WireGuard MTU Configuration

## Problem

Services running directly on the home server (e.g., OpenCode on port 4096) are
unreachable from the proxy server via the WireGuard VPN tunnel. Connections
establish successfully (TCP handshake completes), but the server's HTTP response
never arrives. `curl` hangs indefinitely.

Smaller packets (pings, TCP ACKs, small HTTP requests) work fine. Only larger
response packets are silently dropped.

## Symptoms

A `tcpdump` capture on the home server's `wg0` interface shows:

1. TCP three-way handshake completes normally (SYN, SYN-ACK, ACK)
2. The client sends its HTTP request (small packet, ~76 bytes) -- arrives fine
3. The server sends back a response (~1760 bytes, split into two TCP segments)
4. The **first segment (~1368 bytes payload)** is retransmitted repeatedly but
   never acknowledged by the proxy
5. The **second segment (~392 bytes payload)** arrives fine
6. The proxy sends SACKs for the second segment, proving it received it but not
   the first

This is the signature of an MTU / packet size issue: large packets are dropped,
small packets pass through.

## Root Cause

WireGuard encapsulates every inner IP packet with additional headers:

```
[Outer IP (20B)] [UDP (8B)] [WireGuard (32B)] [Inner IP packet]
```

This adds ~60 bytes of overhead to every packet. The maximum inner packet size
that can traverse the tunnel without fragmentation is:

```
Path MTU - WireGuard overhead = Max inner packet size
1500     - 60                 = 1440
```

When the WireGuard interface MTU is set too high (or left at the default), the
TCP MSS negotiation allows segments that, after WireGuard encapsulation, exceed
the physical path MTU. These oversized outer packets are silently dropped by
intermediate routers that cannot fragment them (DF bit is set).

With `MTU = 1420` on the WireGuard interface:
- TCP negotiates MSS = 1380 (MTU minus 40 bytes IP+TCP headers)
- With TCP timestamp options, actual segment + headers = ~1420 bytes
- After encapsulation: ~1480 bytes
- This is under 1500 but leaves very little margin. Any additional overhead
  (VLAN tags, PPPoE, intermediate links with lower MTU) causes drops.

With `MTU = 1380` on the WireGuard interface:
- TCP negotiates MSS = 1340
- After encapsulation: ~1440 bytes
- Comfortably under 1500 with room for variance

## Why ping works but curl doesn't

ICMP ping packets are small (default 64 bytes). They fit easily within any MTU.
The TCP handshake packets (SYN, ACK) are also small. Only when the application
sends a response larger than ~1340 bytes does the packet exceed the effective
path MTU after encapsulation.

## Why Docker-based services were unaffected

Docker containers use their own network bridge and iptables NAT rules. Docker
typically handles MSS clamping automatically via the `TCPMSS` iptables target,
which adjusts the MSS in TCP SYN packets to match the outgoing interface MTU.
This prevents oversized packets from being generated in the first place.

Native services (like OpenCode running as a systemd service) bind directly to
the WireGuard interface address and rely entirely on the `wg0` interface MTU
for correct MSS negotiation.

## Fix

Set `MTU = 1380` in the `[Interface]` section of both WireGuard configs:

- `templates/client_wg0.conf.j2` (home server)
- `templates/server_wg0.conf.j2` (proxy server)

Both sides should use the same MTU value for consistent behavior.

## Diagnostic commands

```bash
# Check interface MTU values
ip link show wg0

# Test path MTU to remote host (don't fragment)
ping -M do -s 1400 -c 3 <remote-ip>

# Capture traffic on WireGuard interface to inspect packet sizes
sudo tcpdump -i wg0 port <port> -n -c 20

# Check TCP connection state
nc -zv -w 5 <remote-ip> <port>
```

## References

- [WireGuard MTU considerations](https://www.wireguard.com/quickstart/#nat-and-firewall-traversal-persistence)
- WireGuard overhead: 32 bytes (WireGuard header) + 8 bytes (UDP) + 20 bytes (IPv4) = 60 bytes
- Standard Ethernet MTU: 1500 bytes
