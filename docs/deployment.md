# Deploying FFIS on an EOSC EU Node VM

End-to-end runbook for running this service as a public HTTPS endpoint in the
EOSC EU Node (PSNC OpenStack), with a hostname from EGI Dynamic DNS and a
Let's Encrypt certificate.

The FFIS VM has **no public IP**. It sits on the internal EDEN network, and
the EDEN shared entry VM terminates TLS for every service hostname and
proxies to it:

```
Internet --443--> entry VM (floating IP, nginx, certbot)
                     |  plain HTTP over eden-net
                     +--> ffis VM 192.168.0.20:8000 (Docker)
```

The entry VM is set up once per project and has its own runbook:
[eden-infra entry-vm.md](https://github.com/EOSC-EDEN/eden-infra/blob/main/docs/entry-vm.md). This document covers the FFIS side and what to
hand to the entry VM owner.

Worked example throughout:

| Placeholder        | Example value                 |
| ------------------ | ----------------------------- |
| project            | `eden`                        |
| service            | `ffis`                        |
| hostname           | `eden-ffis.vm.fedcloud.eu`    |
| VM login user      | `ubuntu`                      |
| FFIS private IP    | `192.168.0.20` (assigned in A5) |
| entry floating IP  | `62.3.175.x`                  |

Substitute your own values.

---

## Part A - The VM (OpenStack dashboard)

### A1. SSH key pair

Check for an existing key, and only generate one if you have none:

```bash
cat ~/.ssh/id_ed25519.pub            # macOS / Linux
# Get-Content ~\.ssh\id_ed25519.pub  # Windows PowerShell

ssh-keygen -t ed25519 -C "you@example.org"   # only if the above printed nothing
```

Import the public key under **Compute -> Key Pairs -> Import Public Key**, type
`SSH Key`. Upload the `.pub` file rather than pasting, to avoid stray line
breaks.

Send the same public key to the entry VM owner, together with your current
IPv4 address, to get jump access (eden-infra entry-vm.md, Part B). The key never needs to
be copied anywhere else.

### A2. Network and subnet (once per project)

Skip A2 to A4 if `eden-net` and `eden-router` already exist.

**Network -> Networks -> Create Network**

- Network Name: `eden-net`
- Subnet tab: Name `eden-subnet`, Network Address `192.168.0.0/24`
- Subnet Details: defaults

### A3. Router (once per project)

**Network -> Routers -> Create Router**

- Router Name: `eden-router`
- External Network: `PSNC-EXT-PUB1-EDU` (the IPv4 network that provides
  floating IPs; `PSNC-EXT-IPV6-PUB2-EDU` is IPv6 only and will not give you a
  floating IPv4)

The router's external gateway is also what gives the FFIS VM outbound
internet access (via SNAT) without a floating IP.

### A4. Attach the subnet to the router (once per project)

Open `eden-router` -> **Interfaces -> Add Interface** -> select `eden-subnet`
-> Submit.

### A5. Launch the instance

The `eden-service` security group must exist and allow ports 22 and 8000 from
the `eden-entry` group (eden-infra entry-vm.md, A2). Do not create a separate
FFIS group with public rules.

**Compute -> Instances -> Launch Instance**

- Details: name `ffis`
- Source: Images -> `Ubuntu 24.04 LTS`, Create New Volume: **Yes**, 40 GB.
  Default login user is `ubuntu`.
- Flavour: **at least 2 vCPU and 4 GB RAM**. FFIS is not a toy workload: the
  image build compiles nothing but does pull `onnxruntime` for Magika, the
  running container holds the Magika model in memory, and uvicorn runs 2
  workers. 1 vCPU / 2 GB will build slowly and swap under concurrent uploads.
- Networks: `eden-net`
- Security Groups: `eden-service` (remove `default`)
- Key Pair: the one from A1

Do **not** associate a floating IP.

Note the private IP shown on the instance row (`192.168.0.20` in this
example). It stays fixed for the lifetime of the instance, and both the
compose file (D3) and the entry VM's nginx config point at it.

Disk sizing: the built image is roughly 2 to 3 GB (Python 3.11 slim, Magika
and its ONNX runtime, the Siegfried binary and PRONOM signatures). With build
cache, logs and the result cache, 40 GB is comfortable and 20 GB is the floor.

### A6. First login

Add the FFIS VM to `~/.ssh/config` on your machine, going through the entry VM:

```
Host eden-entry
    HostName <entry-floating-IP>
    User jump

Host eden-ffis
    HostName 192.168.0.20
    User ubuntu
    ProxyJump eden-entry
```

Then:

```bash
ssh eden-ffis

# Outbound access works through the router, not a floating IP
curl -sI https://download.docker.com | head -1

sudo apt update && sudo apt -y upgrade
sudo apt -y install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

---

## Part B - Hostname (EGI Dynamic DNS)

Reference: https://docs.egi.eu/users/compute/cloud-compute/dynamic-dns/

### B1. Register the hostname

Log in at https://nsupdate.fedcloud.eu/ via EGI Check-in (your institutional
account works). **Overview -> Add host**: hostname `eden-ffis`, domain
`vm.fedcloud.eu`. The portal shows a secret exactly once - note it down. The
FFIS team keeps it; the entry VM owner does not need it.

Full name: `eden-ffis.vm.fedcloud.eu`

### B2. Point the name at the entry VM

The hostname must resolve to the **entry VM's** floating IP. Pass it
explicitly with `myip`; this can be run from anywhere:

```bash
curl "https://eden-ffis.vm.fedcloud.eu:<secret>@nsupdate.fedcloud.eu/nic/update?myip=<entry-floating-IP>"
```

Expected response: `good <entry-floating-IP>` (or `nochg <entry-floating-IP>`).

Do not leave out `myip` and run this from the FFIS VM. nsupdate would then
record the router's SNAT address, and the hostname would point at the wrong
place.

Registered records do not expire, so this is a one-off unless the entry VM's
floating IP changes.

### B3. Verify it resolves

From your local machine:

```bash
dig +short eden-ffis.vm.fedcloud.eu          # macOS / Linux
# Resolve-DnsName eden-ffis.vm.fedcloud.eu   # Windows PowerShell
```

The answer must be the entry floating IP. Do not ask for the certificate
(C2) until it is - certbot will fail the HTTP-01 challenge otherwise, and
repeated failures count against Let's Encrypt rate limits.

---

## Part C - Publishing through the entry VM

nginx and certbot run on the entry VM, not on the FFIS VM. If you are not the
entry VM owner, send them the items below and they follow eden-infra entry-vm.md,
Part D.

| Item                    | Value                                       |
| ----------------------- | ------------------------------------------- |
| Hostname                | `eden-ffis.vm.fedcloud.eu`                  |
| Backend private IP:port | `192.168.0.20:8000`                         |
| nginx site file         | `deploy/nginx-ffis.conf` in this repository |

### C1. The site config

`deploy/nginx-ffis.conf` is not just a `proxy_pass`. Two settings matter:

- `client_max_body_size 100m` - nginx defaults to **1 MB**. Without this, every
  upload over 1 MB is rejected with a 413 by nginx before FFIS sees it, while
  the app itself happily accepts 100 MB. Keep this number and
  `FFIS_MAX_UPLOAD_BYTES` in `deploy/docker-compose.prod.yml` in sync.
- `proxy_read_timeout 180s` on `/identify` - identification of a large file
  through Siegfried and Magika can exceed nginx's 60 second default, which
  would surface as a 504 mid-request.

It also rate limits `/identify` to 2 req/s per client IP with a burst of 10.
A public identification endpoint is free CPU for anyone who finds it; drop or
raise the limit deliberately, not by accident. The limit applies to the real
client IP, because the entry VM is the first hop.

The `upstream ffis_backend` block at the top holds the FFIS VM's private IP.
It must match `FFIS_BIND_ADDR` in D3.

### C2. Certificate

Issued on the entry VM, one certificate per service:

```bash
sudo certbot --nginx -d eden-ffis.vm.fedcloud.eu \
  --agree-tos --no-eff-email -m you@example.org --redirect
```

Until Part D is done, `https://eden-ffis.vm.fedcloud.eu/` answers 502. That
is expected: nginx is up, FFIS is not yet.

---

## Part D - Deploy FFIS

All of this runs on the FFIS VM (`ssh eden-ffis`).

### D1. Install Docker

```bash
sudo apt -y install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null
sudo apt update
sudo apt -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker ubuntu   # log out and back in for this to take effect
```

### D2. Get the code

```bash
sudo mkdir -p /opt/ffis && sudo chown ubuntu:ubuntu /opt/ffis
git clone https://github.com/<org>/eosc-ffis.git /opt/ffis
cd /opt/ffis
```

### D3. Start the service

Tell compose which address to publish on - the VM's private IP:

```bash
ip -4 -br addr show scope global     # the eden-net address, e.g. 192.168.0.20
echo "FFIS_BIND_ADDR=192.168.0.20" > deploy/.env

docker compose --env-file deploy/.env -f deploy/docker-compose.prod.yml up -d --build
```

`deploy/.env` is git-ignored, so `git pull` leaves it alone.

The first build takes several minutes: it downloads the Siegfried binary,
runs `sf -update` to fetch the PRONOM signature file, and installs Magika and
onnxruntime.

`deploy/docker-compose.prod.yml` differs from the development
`docker-compose.yml` in one respect that matters here: it publishes on one
address (`FFIS_BIND_ADDR`) instead of `0.0.0.0`. Docker inserts its own
iptables rules ahead of the host firewall, so with the dev file the app would
listen on every interface. The `eden-service` security group still limits
port 8000 to the entry VM, but relying on a single layer for that is a bad
habit. If `FFIS_BIND_ADDR` is unset, the port falls back to loopback: safe,
but the entry VM cannot reach it and answers 502.

Check it:

```bash
docker compose --env-file deploy/.env -f deploy/docker-compose.prod.yml ps
curl -s 192.168.0.20:8000/health
curl -s 192.168.0.20:8000/tools | head
```

`curl localhost:8000` does **not** work here, because the port is not bound
on loopback.

### D4. Verify the public endpoint

From your laptop:

```bash
curl -I https://eden-ffis.vm.fedcloud.eu/                      # 200, frontend
curl -s https://eden-ffis.vm.fedcloud.eu/health                # {"status":"ok",...}
curl -s -o /dev/null -w '%{http_code}\n' \
  https://eden-ffis.vm.fedcloud.eu/docs                        # 200, OpenAPI docs

curl -s -X POST https://eden-ffis.vm.fedcloud.eu/identify \
  -F "file=@some-document.pdf" | head -40
```

---

## Part E - Operations

### Updating the service

```bash
cd /opt/ffis
git pull
docker compose --env-file deploy/.env -f deploy/docker-compose.prod.yml up -d --build
```

The PRONOM signature file is baked in at build time, so a periodic rebuild is
also how signatures get refreshed. Rebuilding monthly is reasonable for a
preservation service.

Changes to `deploy/nginx-ffis.conf` do not take effect from `git pull` on the
FFIS VM. They have to be installed on the entry VM (eden-infra entry-vm.md, Part E).

### Logs

```bash
docker compose --env-file deploy/.env -f deploy/docker-compose.prod.yml logs -f ffis
```

Container logs are capped at 3 x 10 MB by the compose file so they cannot fill
the disk. The nginx access and error logs (`/var/log/nginx/ffis.*.log`) are on
the entry VM.

### Reboots

`restart: unless-stopped` plus an enabled `docker` service brings the container
back automatically. Because the port is bound to the private IP, Docker must
start after the network is up; on Ubuntu 24.04 `docker.service` already waits
for `network-online.target`. Confirm after the first reboot rather than
assuming it.

### Security notes for a public deployment

- `/identify/path` (by-reference mode) is denied outright unless
  `FFIS_ALLOWED_PATH_PREFIXES` is set. Leave it unset on a public instance
  unless you have mounted storage and intend to expose it. It is a
  read-a-file-from-the-server endpoint; the prefix allowlist is the only thing
  standing between it and arbitrary local file reads.
- CORS is `allow_origins=["*"]` (`src/ffis/main.py`). Fine for an anonymous
  read-only API, worth narrowing if authentication is ever added.
- The service has no authentication. Anyone who finds the hostname can spend
  its CPU. The nginx rate limit on the entry VM is the mitigation; for a
  non-public pilot, add an IP allowlist or HTTP basic auth to the
  `location ~ ^/identify` block instead.
- Port 8000 on the FFIS VM is plain HTTP. That is acceptable only because
  `eden-service` restricts it to the entry VM. Never open it more widely.
- Uploads are held in memory during identification (`await file.read()` in
  `src/ffis/api/routes.py`). With 2 workers and a 100 MB limit, the worst case
  is around 200 MB of request buffers on top of the Magika model. This is
  another reason for the 4 GB floor, and a reason to lower
  `FFIS_MAX_UPLOAD_BYTES` if your files are small.

### Troubleshooting

| Symptom                            | Likely cause                                                        |
| ---------------------------------- | ------------------------------------------------------------------- |
| `ssh eden-ffis` fails at first hop | Your IP changed; ask the entry VM owner to add it to `eden-entry`    |
| `ssh eden-ffis` fails at second hop | `eden-service` lacks port 22 from `eden-entry`                      |
| `apt` / `docker pull` hangs        | Router has no external gateway, or SNAT is disabled                  |
| Hostname resolves to wrong IP      | nsupdate called without `myip` (B2)                                  |
| certbot HTTP-01 challenge fails    | Hostname does not resolve to the entry floating IP yet               |
| 502 from nginx                     | Container down, or `FFIS_BIND_ADDR` unset so it bound to loopback    |
| 504, or entry VM cannot `curl` the backend | `eden-service` lacks port 8000 from `eden-entry`             |
| 413 on upload                      | `client_max_body_size` missing or below `FFIS_MAX_UPLOAD_BYTES`      |
| 504 on a large file                | `proxy_read_timeout` too low on `/identify`                          |
| 429 on `/identify`                 | nginx rate limit; raise it in the site config if legitimate traffic  |
| Build fails on `sf -update`        | Transient PRONOM fetch failure; rerun the build                      |
| Container fails to start after reboot ("cannot assign requested address") | Docker started before the private IP was up; `docker compose up -d` again and check `systemctl cat docker` |
