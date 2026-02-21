# Run a Node Behind a Proxy

{% hint style="warning" %}
Running a publicly accessible node exposes your infrastructure to the open internet. The configurations below are starting points, not complete security solutions. **Do this at your own risk.** You are responsible for securing and maintaining your own infrastructure.
{% endhint %}

If you plan to run a Stacks node with publicly accessible RPC endpoints, it is strongly recommended to place the node behind a reverse proxy with rate limiting. Without rate limiting, a public node can be overwhelmed by excessive requests, leading to degraded performance or denial of service.

This guide provides minimal, production-tested configurations for [HAProxy](https://www.haproxy.org/) and [Nginx](https://nginx.org/) that you can adapt to your environment.

### Ports overview

A Stacks node deployment typically exposes the following services:

| Service     | Default Port | Protocol | Proxy?          |
| ----------- | ------------ | -------- | --------------- |
| Stacks RPC  | 20443        | HTTP     | Yes             |
| Stacks P2P  | 20444        | TCP      | Optional        |
| Stacks API  | 3999         | HTTP     | Yes, if running |
| Bitcoin RPC | 8332         | HTTP     | Yes, if exposed |
| Bitcoin P2P | 8333         | TCP      | No              |

{% hint style="info" %}
The **P2P ports** (20444, 8333) use custom binary protocols for peer-to-peer communication, not HTTP. You can leave them open directly to the network. The proxy configurations below focus on the **RPC/API ports** which serve HTTP traffic and are the primary target for abuse.
{% endhint %}

## Configure the Stacks node

Before setting up the proxy, configure your Stacks node so its RPC endpoint is not directly reachable from the public internet. The proxy will be the only public-facing service.

### Bare metal

In your node's configuration file (e.g. `Stacks.toml`), set `rpc_bind` to a localhost address:

{% code title="Stacks.toml" %}

```toml
[node]
rpc_bind = "127.0.0.1:20443"    # Only accessible from localhost
p2p_bind = "0.0.0.0:20444"      # Open to peers on the network
# data_url = "http://<your-public-ip>:20443"  # Uncomment if peers need to reach your RPC
```

{% endcode %}

{% hint style="info" %}
If you change the RPC port (e.g. to `30443`), update the proxy backend to match.
{% endhint %}

### Docker (stacks-blockchain-docker)

When running with [stacks-blockchain-docker](https://github.com/stacks-network/stacks-blockchain-docker), the node's ports are controlled by the Docker Compose configuration. By default, ports are exposed on all interfaces (`0.0.0.0`). To restrict them to localhost, edit `compose-files/common.yaml` and change the port mappings to bind to `127.0.0.1` with internal host ports:

{% code title="compose-files/common.yaml (port changes)" %}

```yaml
services:
  stacks-blockchain:
    ports:
      - 127.0.0.1:30443:20443   # RPC: only localhost, internal port 30443
      - 127.0.0.1:30444:20444   # P2P: only localhost, internal port 30444
      - 127.0.0.1:9153:9153     # Metrics: only localhost
  stacks-blockchain-api:
    ports:
      - 127.0.0.1:33999:3999    # API: only localhost, internal port 33999
```

{% endcode %}

The node inside the container still listens on its default ports. Docker maps the host-side ports (`30443`, `30444`, `33999`) to the container ports. HAProxy then listens on the standard public ports (`20443`, `20444`, `3999`) and forwards to these internal host ports.

## HAProxy

HAProxy provides fine-grained connection tracking and abuse detection via [stick tables](https://www.haproxy.com/blog/introduction-to-haproxy-stick-tables). The configuration below proxies Stacks RPC, P2P, and API traffic, automatically rejecting clients that exceed request rate thresholds.

{% hint style="info" %}
Adjust `maxconn`, rate thresholds (`ge 25`, `ge 10`), stick-table sizes, and expiry times to suit your traffic patterns. The values below are conservative defaults.
{% endhint %}

### Linux

{% code title="/etc/haproxy/haproxy.cfg" %}

```
global
    log /dev/log    local0
    log /dev/log    local1 notice
    maxconn 512
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log     global
    mode    tcp
    option  tcplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000

# -------------------------------------------
# Abuse tracking table
# Keeps 100k entries, each expiring after 30m
# -------------------------------------------
backend Abuse
    stick-table type ip size 100K expire 30m store gpc0,http_req_rate(10s)

# -------------------------------------------
# Stacks RPC (public: 20443 -> node: 20443)
# For Docker setups, point to the internal
# host port (e.g. 127.0.0.1:30443)
# -------------------------------------------
frontend stacks_rpc
    bind *:20443
    maxconn 512
    acl is_abuse src_http_req_rate(Abuse) ge 25
    acl inc_abuse_cnt src_inc_gpc0(Abuse) gt 0
    acl abuse_cnt src_get_gpc0(Abuse) gt 0
    tcp-request connection track-sc0 src table Abuse
    tcp-request connection reject if abuse_cnt
    default_backend stacks_rpc_back

backend stacks_rpc_back
    server stacks-node 127.0.0.1:20443 maxconn 100 check inter 10s

# -------------------------------------------
# Stacks P2P (public: 20444 -> node: 20444)
# -------------------------------------------
frontend stacks_p2p
    bind *:20444
    maxconn 512
    acl is_abuse src_http_req_rate(Abuse) ge 10
    acl inc_abuse_cnt src_inc_gpc0(Abuse) gt 0
    acl abuse_cnt src_get_gpc0(Abuse) gt 0
    tcp-request connection track-sc0 src table Abuse
    tcp-request connection reject if abuse_cnt
    default_backend stacks_p2p_back

backend stacks_p2p_back
    server stacks-node 127.0.0.1:20444 maxconn 100 check inter 10s

# -------------------------------------------
# Stacks API (public: 3999 -> node: 3999)
# -------------------------------------------
frontend stacks_api
    bind *:3999
    maxconn 512
    acl is_abuse src_http_req_rate(Abuse) ge 25
    acl inc_abuse_cnt src_inc_gpc0(Abuse) gt 0
    acl abuse_cnt src_get_gpc0(Abuse) gt 0
    tcp-request connection track-sc0 src table Abuse
    tcp-request connection reject if abuse_cnt
    default_backend stacks_api_back

backend stacks_api_back
    server stacks-api 127.0.0.1:3999 maxconn 100 check inter 10s

# -------------------------------------------
# Bitcoin RPC (optional, if you expose it)
# -------------------------------------------
frontend btc_rpc
    bind *:18332
    maxconn 512
    acl is_abuse src_http_req_rate(Abuse) ge 25
    acl inc_abuse_cnt src_inc_gpc0(Abuse) gt 0
    acl abuse_cnt src_get_gpc0(Abuse) gt 0
    tcp-request connection track-sc0 src table Abuse
    tcp-request connection reject if abuse_cnt
    default_backend btc_rpc_back

backend btc_rpc_back
    server bitcoin 127.0.0.1:8332 maxconn 100 check inter 10s
```

{% endcode %}

{% code title="Enable and start HAProxy" %}

```bash
sudo systemctl enable haproxy
sudo systemctl start haproxy
```

{% endcode %}

### macOS

On macOS, install HAProxy via Homebrew. The configuration omits `chroot`, `user`, and `group` directives since HAProxy runs as the current user through `launchd`.

{% code title="Install HAProxy" %}

```bash
brew install haproxy
```

{% endcode %}

{% code title="/opt/homebrew/etc/haproxy.cfg" %}

```
global
    log 127.0.0.1    local0
    maxconn 512

defaults
    log     global
    mode    tcp
    option  tcplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000

backend Abuse
    stick-table type ip size 100K expire 30m store gpc0,http_req_rate(10s)

frontend stacks_rpc
    bind *:20443
    maxconn 512
    acl is_abuse src_http_req_rate(Abuse) ge 25
    acl inc_abuse_cnt src_inc_gpc0(Abuse) gt 0
    acl abuse_cnt src_get_gpc0(Abuse) gt 0
    tcp-request connection track-sc0 src table Abuse
    tcp-request connection reject if abuse_cnt
    default_backend stacks_rpc_back

backend stacks_rpc_back
    server stacks-node 127.0.0.1:30443 maxconn 100 check inter 10s

frontend stacks_p2p
    bind *:20444
    maxconn 512
    acl is_abuse src_http_req_rate(Abuse) ge 10
    acl inc_abuse_cnt src_inc_gpc0(Abuse) gt 0
    acl abuse_cnt src_get_gpc0(Abuse) gt 0
    tcp-request connection track-sc0 src table Abuse
    tcp-request connection reject if abuse_cnt
    default_backend stacks_p2p_back

backend stacks_p2p_back
    server stacks-node 127.0.0.1:30444 maxconn 100 check inter 10s

frontend stacks_api
    bind *:3999
    maxconn 512
    acl is_abuse src_http_req_rate(Abuse) ge 25
    acl inc_abuse_cnt src_inc_gpc0(Abuse) gt 0
    acl abuse_cnt src_get_gpc0(Abuse) gt 0
    tcp-request connection track-sc0 src table Abuse
    tcp-request connection reject if abuse_cnt
    default_backend stacks_api_back

backend stacks_api_back
    server stacks-api 127.0.0.1:33999 maxconn 100 check inter 10s
```

{% endcode %}

{% code title="Validate and start HAProxy" %}

```bash
haproxy -c -f /opt/homebrew/etc/haproxy.cfg
brew services start haproxy
```

{% endcode %}

### Verify

{% code title="Test the RPC endpoint through the proxy" %}

```bash
curl -s localhost:20443/v2/info | jq
```

{% endcode %}

{% hint style="info" %}
**How the abuse table works:** HAProxy tracks each client IP's request rate. When a client exceeds the threshold (e.g. 25 requests in 10 seconds for RPC), its `gpc0` counter is incremented and all subsequent connections from that IP are rejected. The stick-table entry expires after 30 minutes, lifting the block automatically.
{% endhint %}

## Nginx

Nginx can serve as a reverse proxy with basic rate limiting using the `limit_req` module. The configuration below rate-limits both the Stacks RPC and Stacks API endpoints.

{% code title="/etc/nginx/sites-available/stacks-node" %}

```nginx
limit_req_zone $binary_remote_addr zone=stacks_rpc:10m rate=5r/s;
limit_req_zone $binary_remote_addr zone=stacks_api:10m rate=10r/s;

server {
    listen 80;

    # Stacks RPC
    location /v2/ {
        limit_req zone=stacks_rpc burst=20 nodelay;
        proxy_pass http://127.0.0.1:20443;
    }

    # Stacks API (if running)
    location / {
        limit_req zone=stacks_api burst=40 nodelay;
        proxy_pass http://127.0.0.1:3999;
    }
}
```

{% endcode %}

Enable the site and restart Nginx:

{% code title="Enable and start Nginx" %}

```bash
sudo ln -s /etc/nginx/sites-available/stacks-node /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

{% endcode %}

{% hint style="info" %}
HAProxy's stick tables offer more granular abuse detection (tracking multiple dimensions per IP, automatic blocking) compared to Nginx's `limit_req`. If fine-grained rate limiting is your priority, HAProxy is the stronger choice.
{% endhint %}

## Firewall considerations

Ensure that only the proxy's listening ports and the P2P ports are reachable from the public internet. The node's RPC should only be accessible via the proxy (localhost).

{% code title="UFW example" %}

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp              # SSH
sudo ufw allow 20443/tcp           # Stacks RPC (via proxy)
sudo ufw allow 20444/tcp           # Stacks P2P (direct or via proxy)
sudo ufw allow 8333/tcp            # Bitcoin P2P (direct)
sudo ufw enable
```

{% endcode %}

{% hint style="warning" %}
**Docker users:** Docker manipulates `iptables` directly and bypasses UFW rules. If your node runs in Docker, bind container ports to `127.0.0.1` explicitly (e.g. `-p 127.0.0.1:20443:20443`) or use the `DOCKER-USER` iptables chain to enforce restrictions. See the [Docker documentation](https://docs.docker.com/engine/network/packet-filtering-firewalls/) for details.
{% endhint %}
