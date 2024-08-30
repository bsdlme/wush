# wush

[![Go Reference](https://pkg.go.dev/badge/github.com/coder/wush.svg)](https://pkg.go.dev/github.com/coder/wush)

`wush` is a command line tool that lets you easily transfer files and open
shells over a peer-to-peer WireGuard connection. It's similar to
[magic-wormhole](https://github.com/magic-wormhole/magic-wormhole) but:

1. No requirement to set up or trust a relay server for authentication.
1. Powered by WireGuard for secure, fast, and reliable connections.
1. Automatic peer-to-peer connections over UDP.
1. Endless possibilities; rsync, ssh, etc.

> [!NOTE]  
> `wush` uses Tailscale's [tsnet](https://tailscale.com/kb/1244/tsnet) package
> under the hood, managed by an in-memory control server on each CLI. We utilize
> Tailscale's public [DERP relays](https://tailscale.com/kb/1232/derp-servers),
> but no Tailscale account is required.

## Install

```bash
curl -fsSL https://wush.dev/install.sh | sh
```

For a manual installation, see the [latest release](https://github.com/coder/wush/releases/latest).

> [!TIP]
> To increase transfer speeds, `wush` attempts to increase the buffer size of
> its UDP sockets. For best performance, ensure `wush` has `CAP_NET_ADMIN`. When
> using the installer script, this is done automatically for you.
>
> ```bash
> # Linux only
> sudo setcap cap_net_admin=eip $(which wush)
> ```

## Basic Usage

[![asciicast](https://asciinema.org/a/ZrCNiRRkeHUi5Lj3fqC3ovLqi.svg)](https://asciinema.org/a/ZrCNiRRkeHUi5Lj3fqC3ovLqi)

On the host machine:

```bash
$ wush host
Picked DERP region Toronto as overlay home
Your auth key is:
    >  112v1RyL5KPzsbMbhT7fkEGrcfpygxtnvwjR5kMLGxDHGeLTK1BvoPqsUcjo7xyMkFn46KLTdedKuPCG5trP84mz9kx
Use this key to authenticate other wush commands to this instance.
05:18:59 WireGuard is ready
05:18:59 SSH server listening
```

On the client machine:

```bash
# Copy a file to the receiver
$ wush cp 1gb.txt
Auth information:
    > Server overlay STUN address:  Disabled
    > Server overlay DERP home:     Toronto
    > Server overlay public key:    [NFWN0]
    > Server overlay auth key:      [mTbpN]
Bringing WireGuard up..
WireGuard is ready!
Received peer
Peer active with relay  nyc
Peer active over p2p  172.20.0.8:53768
Uploading "1gb.txt" 100% |██████████████████████████████████████████████| (2.1/2.1 GB, 376 MB/s)

# Open a shell to the receiver
$ wush ssh
┃ Enter the receiver's Auth key:
┃ > 112v1RyL5KPzsbMbhT7fkEGrcfpygxtnvwjR5kMLGxDHGeLTK1BvoPqsUcjo7xyMkFn46KLTdedKuPCG5trP84mz9kx
Auth information:
    > Server overlay STUN address:  Disabled
    > Server overlay DERP home:     Toronto
    > Server overlay public key:    [sEIS1]
    > Server overlay auth key:      [w/sYF]
Bringing WireGuard up..
WireGuard is ready!
Received peer
Peer active with relay  nyc
Peer active over p2p  172.20.0.8:44483
coder@colin:~$
```

## Technical Details

`wush` doesn't require you to trust any 3rd party authentication or relay
servers, instead using x25519 keys to authenticate incoming connections. Auth
keys generated by `wush receive` are separated into a couple parts:

```text
112v1RyL5KPzsbMbhT7fkEGrcfpygxtnvwjR5kMLGxDHGeLTK1BvoPqsUcjo7xyMkFn46KLTdedKuPCG5trP84mz9kx

+---------------------+------------------+---------------------------+----------------------------+
| UDP Address (1-19B) | DERP Region (2B) |  Server Public Key (32B)  |  Sender Private Key (32B)  |
+---------------------+------------------+---------------------------+----------------------------+
| 203.128.89.74:57321 |               21 | QPGoX1GY......488YNqsyWM= | o/FXVnOn.....llrKg5bqxlgY= |
+---------------------+------------------+---------------------------+----------------------------+
```

Senders and receivers communicate over what we call an "overlay". An overlay
runs over one of two currently implemented mediums; UDP or DERP. Each message
over the relay is encrypted with the sender's private key.

**UDP**: The receiver creates a NAT holepunch to allow senders to connect
directly. WireGuard nodes are exchanged peer-to-peer. This mode will only work
if the receiver doesn't have hard NAT.

**DERP**: The receiver connects to the closet DERP relay server. WireGuard nodes
are exchanged through the relay.

In both cases auth is handled the same way. The receiver will only accept
messages encrypted from the sender's private key, to the server's public key.

## Why create another file transfer tool?

Lots of great file tranfer tools exist, but they all have some limitations:

1. Slow speeds due to relay servers.
1. Trusting a 3rd party server for authentication.
1. Limited to only file transfers.

We sought to utilize advancements in userspace networking brought about by
Tailscale to create a tool that could solve all of these problems, and provide
way more functionality.

## Acknowledgements

1. [Tailscale](https://tailscale.com)
1. [Headscale](https://github.com/juanfont/headscale)
1. [WireGuard-go](https://github.com/WireGuard/wireguard-go)
