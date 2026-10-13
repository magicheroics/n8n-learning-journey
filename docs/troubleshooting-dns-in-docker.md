# Troubleshooting: "The DNS server returned an error" in n8n (Docker)

**Environment:** n8n Community Edition in Docker on Ubuntu
**Where it happened:** a workflow that worked at home started failing when I ran it from a public Wi-Fi network (a café)

---

## Summary

| | |
|---|---|
| **Problem** | An n8n workflow failed with: *"The DNS server returned an error, perhaps the server is offline"* |
| **Root cause** | The n8n container was still configured with my **home network's DNS servers** (private `192.168.x.x` addresses), which are unreachable from any other network |
| **Fix** | Recreate the container with explicit public DNS servers using `--dns` |
| **Bonus hardening** | Bind n8n's port to `127.0.0.1` so it is not exposed on public Wi-Fi |
| **Data loss** | None. Workflows and credentials live in the `n8n_data` Docker volume |

---

## 1. The Problem

Running a workflow whose nodes call Google services (the Gemini model and Gmail) failed with:

```
The DNS server returned an error, perhaps the server is offline
```

The same workflow had worked earlier on my home network.

**What the message means:** a hostname such as `generativelanguage.googleapis.com` could not be turned into an IP address. That is a DNS problem, not a credentials or workflow problem. A credentials problem would show a `401` or `invalid_grant` message instead.

---

## 2. Diagnosis: narrowing it down

I worked from the outside in, separating "no internet", "broken DNS on the laptop", and "broken DNS inside the container".

### Step 1: Is the laptop online?

```bash
ping -c 3 8.8.8.8
```

Replies came back, so the laptop had internet access.

### Step 2: Does name resolution work on the laptop?

```bash
ping -c 3 google.com
nslookup google.com 1.1.1.1
```

Both worked. The laptop's DNS was fine, so the problem was not the network or the café's DNS.

### Step 3: Does DNS work inside the container?

```bash
docker exec n8n nslookup generativelanguage.googleapis.com
```

Result:

```
;; connection timed out; no servers could be reached
```

The laptop could resolve names but the container could not. That pointed at the container's own DNS configuration.

### Step 4: Which DNS servers is the container using?

```bash
docker exec n8n cat /etc/resolv.conf
```

The file listed two `nameserver` entries, both **private addresses from my home network** (`192.168.x.x`: my router and a local DNS server).

### Diagnosis table

| Test | Result | Conclusion |
|------|--------|------------|
| `ping 8.8.8.8` (laptop) | Success | Internet connectivity is fine |
| `ping google.com` (laptop) | Success | Laptop DNS is fine |
| `nslookup google.com 1.1.1.1` (laptop) | Success | Public DNS is reachable from this network |
| `nslookup` inside the container | Timed out | Container DNS is broken |
| `cat /etc/resolv.conf` in the container | Home-network DNS servers | Those servers don't exist on the café network |

---

## 3. Root cause

Docker configures a container's DNS (`/etc/resolv.conf`) from the host's settings when the container is set up. My n8n container had picked up my **home** DNS servers. When I moved to a different network, the laptop switched to the new network's DNS automatically, but the container kept using the old servers. Those private `192.168.x.x` addresses are not reachable from another network, so every lookup timed out.

In short: **the laptop adapted to the new network, but the container's DNS settings did not.**

---

## 4. The fix

Recreate the container with explicit public DNS servers. Because the data is stored in a Docker **volume**, removing the container does not delete workflows or credentials.

```bash
docker rm -f n8n
docker run -d --restart unless-stopped --name n8n \
  -p 127.0.0.1:5678:5678 \
  --dns 1.1.1.1 --dns 8.8.8.8 \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

What changed compared with my original command:

| Option | Purpose |
|--------|---------|
| `--dns 1.1.1.1 --dns 8.8.8.8` | The container always uses public DNS, no matter which network the laptop is on |
| `-p 127.0.0.1:5678:5678` | Publishes the port to the laptop only, instead of every network interface |

### Verifying the fix

```bash
docker exec n8n nslookup generativelanguage.googleapis.com
```

This now returned Google's IP addresses, with `1.1.1.1` shown as the server. After about a minute for n8n to start, I reopened `http://localhost:5678`, and the workflow ran successfully.

```bash
docker ps
```

The ports column should read `127.0.0.1:5678->5678/tcp`.

---

## 5. Security note: don't expose n8n on public Wi-Fi

My original command used `-p 5678:5678`. That publishes n8n on **all** of the laptop's network interfaces, so on a shared network other people could reach my n8n login page at the laptop's IP address.

Using `-p 127.0.0.1:5678:5678` restricts access to the laptop itself. `http://localhost:5678` still works exactly as before.

If I ever need to reach n8n from another device, I should expose it deliberately and protect it (for example, behind a VPN or reverse proxy with authentication), not leave it open by default.

---

## 6. Tradeoffs

- **DNS filtering is bypassed for n8n.** With fixed public DNS, the container no longer uses any local DNS filtering from my home network. That's fine here, because n8n only talks to Google's APIs. The rest of my devices are unaffected.
- **The container no longer follows network DNS changes.** If I ever need n8n to resolve internal hostnames (for example, local servers by name), I'd need to add those DNS servers deliberately or use IP addresses.

---

## 7. Lessons learned

1. **Isolate the layer.** Testing the laptop, then the container, showed in minutes that the fault was inside the container.
2. **Compare IP connectivity with name resolution.** `ping 8.8.8.8` working while `ping google.com` fails means DNS. Both failing means connectivity.
3. **Query a specific DNS server** (`nslookup name 1.1.1.1`) to tell whether the network's DNS or the local configuration is at fault.
4. **A container has its own network view.** What works on the host doesn't automatically work inside a container.
5. **Read the error literally.** "DNS server returned an error" pointed straight at name resolution, not at credentials or the workflow logic.
6. **Check exposure when changing networks.** Public Wi-Fi is the place where a service listening on all interfaces becomes a real risk.

---

## Quick reference

```bash
# Is the laptop online?
ping -c 3 8.8.8.8

# Does name resolution work on the laptop?
ping -c 3 google.com
nslookup google.com 1.1.1.1

# Does name resolution work inside the container?
docker exec n8n nslookup generativelanguage.googleapis.com

# Which DNS servers does the container use?
docker exec n8n cat /etc/resolv.conf

# Recreate the container with fixed DNS and localhost-only access
docker rm -f n8n
docker run -d --restart unless-stopped --name n8n \
  -p 127.0.0.1:5678:5678 \
  --dns 1.1.1.1 --dns 8.8.8.8 \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n

# Confirm the port binding
docker ps
```
