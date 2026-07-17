# curl, dig, and traceroute: The Networking Trio for Incident Response

"It's probably DNS" has become such a running joke in infrastructure circles that it's practically a meme at this point — and the reason it's funny is that it's true disturbingly often. When two services can't talk to each other, the instinct is to assume something exotic: a firewall misconfiguration, a broken service mesh policy, a certificate that expired at the worst possible moment. Sometimes it is one of those. Most of the time, it's something you can diagnose with three tools that have existed since before most of us were writing code.

## curl: Not Just for Checking If Something's Up

Everyone knows `curl` as "hit an endpoint and see what comes back." The daily-driver version of curl goes a bit further, because the response body is rarely the interesting part during an incident — the headers and timing usually are.

```bash
curl -w "\nDNS: %{time_namelookup}s | Connect: %{time_connect}s | TTFB: %{time_starttransfer}s | Total: %{time_total}s\n" \
  -o /dev/null -s https://internal-api.company.com/health
```

That breakdown tells you *where* the time is going, which matters enormously. A slow `time_namelookup` points at DNS. A slow `time_connect` with fast DNS points at network path or a saturated connection pool. A fast connect but slow `time_starttransfer` usually means the server received the request fine and is just slow to process it — which points you at application code, not infrastructure at all.

```bash
# What's actually coming back, headers included, without the noise of a browser
curl -sv https://internal-api.company.com/health 2>&1 | grep -E "^[<>]"
```

At a company running dozens of internal services behind a shared ingress layer, this kind of header inspection is often how you catch a misrouted request — a response coming back with the wrong service's headers is a strong signal that a routing rule somewhere is misconfigured, well before you'd catch that from a dashboard.

## dig: Confirming or Ruling Out DNS in Ten Seconds

Given how often DNS actually is the culprit, it's worth having the muscle memory to rule it in or out immediately, rather than several steps into a longer investigation.

```bash
# What does DNS actually resolve to, and which server answered?
dig internal-api.company.com +short

# Is this consistent across resolvers, or is caching lying to you?
dig internal-api.company.com @8.8.8.8 +short
dig internal-api.company.com @1.1.1.1 +short
```

The second pattern matters more than people expect. A stale DNS cache on one resolver giving a different answer than another is a classic cause of "it works for me but not for my coworker" reports during a migration, and it's the kind of thing that looks like a flaky service until you actually compare resolver answers side by side.

For internal Kubernetes DNS specifically, the same instinct applies inside the cluster:

```bash
kubectl run -it --rm debug --image=busybox --restart=Never -- \
  nslookup my-service.production.svc.cluster.local
```

## traceroute: Underused, but the Right Tool for "Where Exactly Does This Break"

`traceroute` gets less daily use than curl or dig simply because most infrastructure today lives behind load balancers and cloud networking layers where a classic hop-by-hop trace isn't as informative as it used to be on flatter networks. But when you're debugging cross-region latency, or trying to figure out whether a request is exiting through the network path you expect, it's still the right tool.

```bash
traceroute -n internal-api.us-west-2.company.com
```

The `-n` flag matters here — skipping reverse DNS lookups on every hop makes the trace return in seconds instead of sometimes timing out entirely on hops that don't have reverse DNS configured, which is common inside cloud provider networks.

## Putting It Together: A Real Sequence

The actual order these get used in during an incident tends to look like this, and it's worth internalizing as a default sequence rather than reasoning it out fresh every time:

```
1. dig the hostname — does it resolve, and to what?
2. curl -w with timing — where is the time actually going?
3. If DNS and connect look fine but requests still fail — check the app logs, not the network
4. If cross-region — traceroute to confirm the path matches expectations
```

## A Small Script That Automates the First Two Steps

```python
import subprocess
import time

def check_endpoint(hostname):
    dig = subprocess.run(
        ["dig", hostname, "+short"], capture_output=True, text=True
    )
    resolved_ips = dig.stdout.strip().split("\n")

    curl = subprocess.run(
        ["curl", "-w", "%{time_namelookup},%{time_connect},%{time_starttransfer},%{time_total}",
         "-o", "/dev/null", "-s", f"https://{hostname}/health"],
        capture_output=True, text=True
    )
    dns_t, conn_t, ttfb_t, total_t = curl.stdout.split(",")

    print(f"{hostname} -> {resolved_ips}")
    print(f"  DNS: {dns_t}s  Connect: {conn_t}s  TTFB: {ttfb_t}s  Total: {total_t}s")

if __name__ == "__main__":
    check_endpoint("internal-api.company.com")
```

None of this is complicated. What matters is having it fast, and having it automatic enough that you're not reasoning from first principles every time someone pings you saying a service "feels slow" — a phrase that, on its own, tells you almost nothing until you've run these three tools and turned a feeling into a specific, fixable layer.
