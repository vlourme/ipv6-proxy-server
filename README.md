# Go Proxy Server

A configurable proxy server in Go, supporting rotating IPv4/IPv6 addresses, session and external authentication.

## Features

- IPv4 or IPv6 back-connect
- HTTP and SOCKS5(h) support
- Multiple IPv6-IPv4 prefixes supported, with per-country prefixes
- Session and timeout support to re-use generated IP
- Up to 14,000 requests per second
- DNS resolution and caching
- Automatic IPv6 routing and sysctl setup
- Authentication with Redis or credentials
- One listener per core via `SO_REUSEPORT` and `SO_REUSEADDR`

## Setup

### Prerequisites

- [Go](https://golang.org/dl/)
- Server with IPv6 support and large-enough subnet ([Hetzner](https://hetzner.cloud/?ref=BV2rohR8OBWQ) offers /64 subnets, *sponsored link*).

### Configuration

Copy the `config.example.yaml` file to `config.yaml` and modify the settings as needed.

Example:
```yaml
listen_address: "::"
listen_port: 8080
debug_mode: false # Enable pretty-print logs, don't enable in production
test_port: -1 # Enable a test server on port 8081 for benchmarking
network_type: "tcp6" # tcp = dual-stack, tcp6 = IPv6 only, tcp4 = IPv4 only
max_timeout: 30 # Maximum timeout for a request in seconds
auth:
  type: "credentials" # none, credentials, redis
  credentials:
    username: "username"
    password: "password"
  redis:
    dsn: "redis://localhost:6379" # Will compare the username as key to the password
bind_prefixes: # IPv4/IPv6 prefixes to bind to
  - "2a14:dead:beef::1/48" # List of prefixes to bind to
  - "2a14:dead:feed::1/48"
enable_fallback: true # Fallback to IPv4 if the target does not match generated IP family above
fallback_prefixes:
  - "1.2.3.4/32" # List of prefixes to fallback to, should be IPv4
located_prefixes:
  ch:
    - "2a14:dead:beef::/48"
  uk:
    - "2a14:dead:feed::/48"
replace_ips:
  "1.2.3.0/24": "2a14:dead:beef::"
deleted_headers: # List of headers to delete (HTTP only), this make proxy anonymous
  - "Proxy-Authorization"
  - "Proxy-Connection"
```

### Build

```bash
go build .
./go-proxy
```

### Usage example

```bash
# Without credentials, if enabled
curl -x http://localhost:8080 http://api.ipquery.io

# Session
curl -x http://john-session-abcdef1234:doe@localhost:8080 http://api.ipquery.io

# Session with timeout
curl -x http://john-session-abcdef1234-timeout-10:doe@localhost:8080 http://api.ipquery.io

# Location override
curl -x http://john-country-ch:doe@localhost:8080 http://api.ipquery.io

# Disable fallback (disable switching to IPv4 if the target is not IPv6-capable)
# Recommended when using IPv6-only websites to ensure the IP is always IPv6
curl -x http://john-fallback-no:doe@localhost:8080 http://api.ipquery.io

# SOCKS5
curl -x socks5://john:doe@localhost:8080 http://api.ipquery.io

# SOCKS5h (server-side DNS resolution, recommended)
curl -x socks5h://john:doe@localhost:8080 http://api.ipquery.io
```

> Session ID must be alphanumeric, between 6 and 24 characters.
> The timeout is in minutes, between 1 and 30 minutes.

### IP Override

IP override is a map of CIDR to IP.
If the resolved IP is present in any defined CIDR, it will be replaced with the override.
Some domains does not reply any `AAAA` record, but in fact, they support IPv6 by replacing the DNS resolution.
This can be used to bypass some IPv6 limitations of CDN and DNS providers.

Example config:
```yaml
replace_ips:
  "1.2.3.0/24": "2a14:dead:beef::"
```

If you're domain resolve to `1.2.3.4`, it will be replaced with `2a14:dead:beef::` by the proxy.


## Benchmark

Benchmark can be run with `go test -bench=.`. Configuration should have `test_port` set to 8081 and credentials
set to `username:password`.

**Benchmark results on 24-cores dedicated server:**
```sh
$ go run ./cmd/test/
Running benchmark: 100 concurrency, 600 total requests
Fastest:  1.62083ms
Slowest:  56.380979ms
Average:  20.106005ms
Total:    128.167636ms
Throughput: 4681.37 req/s

Running benchmark: 250 concurrency, 1500 total requests
Fastest:  2.491974ms
Slowest:  96.381628ms
Average:  26.371493ms
Total:    169.393879ms
Throughput: 8855.10 req/s

Running benchmark: 500 concurrency, 4000 total requests
Fastest:  2.485129ms
Slowest:  148.471635ms
Average:  41.59583ms
Total:    357.834735ms
Throughput: 11178.34 req/s

Running benchmark: 1000 concurrency, 6000 total requests
Fastest:  1.481635ms
Slowest:  220.899026ms
Average:  68.733736ms
Total:    437.22757ms
Throughput: 13722.83 req/s

Running benchmark: 2500 concurrency, 10000 total requests
Fastest:  29.777132ms
Slowest:  487.61035ms
Average:  164.518955ms
Total:    721.735415ms
Throughput: 13855.49 req/s

Running benchmark: 5000 concurrency, 20000 total requests
Fastest:  970.738µs
Slowest:  729.66545ms
Average:  332.911955ms
Total:    1.48569428s
Throughput: 13461.72 req/s

Running benchmark: 10000 concurrency, 40000 total requests
Fastest:  67.003994ms
Slowest:  1.244676058s
Average:  661.01839ms
Total:    2.919776624s
Throughput: 13699.68 req/s
```