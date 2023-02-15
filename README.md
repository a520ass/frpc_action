# Frp client Action

use frpc to forward any port from/to GitHub Actions Runners machines. ex. fallback to expose ssh server port for debugging

see https://github.com/fatedier/frp

frpc xtcp uses UDP to establish a P2P connection between clients, even behind NAT, which may achieve better throughput than any "proxy" technology such as Cloudflare Tunnel/ngrok/tmate/upterm

## Features

- Support Ubuntu

## Prerequisites

You need a `frps` server running on public ip

## Github Action Inputs

| Variable   | Description                                                                                                             |
| ---------- | ----------------------------------------------------------------------------------------------------------------------- |
| `config`   | frpc config                                                                                                             |
| `users`    | GitHub users who can ssh into the Action Runner machine using there public key(default: ${{ github.triggering_actor }}) |
| `host_key` | set /etc/ssh/sshd_config `HostKey` in Action Runner machine                                                             |

## Example Usage

Actions variables:

```ini
[common]
server_addr = your.frps.com
server_port = 7000

[ssh_stcp]
type = stcp
sk = your_preshared_key_wXjHuAJa
local_ip = 127.0.0.1
local_port = 22
use_encryption = false
use_compression = false

[ssh_xtcp]
type = xtcp
sk = your_preshared_key_wXjHuAJa
local_ip = 127.0.0.1
local_port = 22
use_encryption = false
use_compression = false
```

.github/workflows/example.yml

```yaml
name: Example
permissions:
  contents: read
on:
  workflow_dispatch:
    inputs:
      users:
        type: string
jobs:
  fallback_to_frpc_ssh_for_debugging:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: "echo step 1"
      - run: "echo step 2 failed; false"
      - run: "echo step 3"
      - name: start frpc
        if: failure()
        uses: douniwan5788/frpc_action@main
        with:
          users: ${{ inputs.users || github.actor }}
          host_key: ${{ secrets.HOST_KEY }}
          config: ${{ vars.FRPC_SERVER_CONF }}
```

frps.ini for your fpc server config

```ini
[common]
bind_port = 7000
bind_udp_port = 7000
```

frpc.ini for your local frpc

```ini
[common]
server_addr = your.frps.com
server_port = 7000

[ssh_proxy]
role = visitor
type = stcp
server_name = ssh_stcp
sk = your_preshared_key_wXjHuAJa
bind_addr = 127.0.0.1
bind_port = 8022
use_encryption = false
use_compression = false

[ssh_p2p]
role = visitor
type = xtcp
server_name = ssh_xtcp
sk = your_preshared_key_wXjHuAJa
bind_addr = 127.0.0.1
bind_port = 9022
use_encryption = false
use_compression = false

```

```bash
ssh runner@127.0.0.0 -p 9022 -i ~/.ssh/your_github_private_key
```
