# EDEN shared entry VM

Runbook for the single public entry point in front of all EDEN service VMs in
the EOSC EU Node (PSNC OpenStack). This is done **once per project**, not once
per service. Per-service runbooks (for FFIS: [deployment.md](deployment.md))
assume this VM already exists.

- Setting up your own service VM behind it: [Part G](#part-g---checklist-for-a-new-service-team).
- Moving from the current standalone FFIS VM: [Part F](#part-f---cutover-plan-from-the-standalone-ffis-vm).

## Why

Floating IPs are scarce. Instead of one floating IP per service VM, only the
entry VM has one. Every service hostname resolves to that IP, nginx on the
entry VM terminates TLS and picks the backend by hostname, and the service VMs
sit on the internal network with no public address.

```
Internet --80/443--> entry VM (floating IP)
                       nginx, certbot, one site file per service hostname
                         |
                         |  plain HTTP over eden-net (192.168.0.0/24)
                         +--> ffis VM       192.168.0.20:8000
                         +--> harvester VM  192.168.0.x:<port>
                         +--> ...

Admin SSH:  laptop --22--> entry VM --22--> service VM   (ProxyJump)
```

Hostnames are under `vm.fedcloud.eu` for now. Moving them to a project
domain (`<service>.services.eden-fidelis.eu`) is planned separately in
[project-domain.md](project-domain.md); it only affects the entry VM's nginx
and certificates, never the service VMs.

## Ownership

| Role                       | Who                     | Responsible for                                          |
| -------------------------- | ----------------------- | -------------------------------------------------------- |
| Entry VM owner             | Mattias Levlin (CSC)    | The VM, its nginx and certbot, onboarding new services   |
| Entry VM backup owner      | TBD                     | Same, when the owner is unavailable                      |
| Service owner (per service) | The service team        | Its service VM, its hostname, the content of its site file |
| `eden-fidelis.eu` domain   | Trust-IT (request sent 2026-10-15) | Delegating a subdomain to EGI DNS, see [project-domain.md](project-domain.md) |

Conventions on the entry VM:

- One nginx site file per service hostname, in
  `/etc/nginx/sites-available/<service>`, named after the service. The service
  team writes it (it encodes their upload limits and timeouts); the entry VM
  owner installs it.
- One certificate per service, covering that service's hostnames. Adding or
  removing a service never touches another service's certificate.
- `limit_req_zone` names are prefixed with the service name
  (`ffis_identify`, `harvester_...`). They live in the shared `http` context
  and must not collide.

Worked example values:

| Placeholder           | Example value              |
| --------------------- | -------------------------- |
| entry VM name         | `eden-entry`               |
| entry floating IP     | `62.3.175.x`               |
| VM login user         | `ubuntu`                   |
| FFIS private IP       | `192.168.0.20`             |
| FFIS hostname         | `eden-ffis.vm.fedcloud.eu` |

---

## Part A - OpenStack

### A1. Network, subnet, router

`eden-net` (`192.168.0.0/24`), `eden-subnet` and `eden-router` (external
network `PSNC-EXT-PUB1-EDU`, subnet attached as an interface) are shared by all
EDEN VMs. If they do not exist yet, create them as in
[deployment.md, A2](deployment.md#a2-network-and-subnet-once-per-project).

The router's external gateway also gives the service VMs **outbound** internet
access via SNAT, so they can run `apt`, pull images and fetch signature files
without a floating IP. Do not disable SNAT on the router.

### A2. Security groups

Two groups, created once. Use **remote security group** rules for internal
traffic rather than CIDRs, so nobody has to track private IPs.

**`eden-entry`** - attached to the entry VM only:

| Port        | Remote                    | Why                                                       |
| ----------- | ------------------------- | --------------------------------------------------------- |
| 22 (SSH)    | CIDR `<admin-IP>/32`      | One rule per admin location. Never `0.0.0.0/0`            |
| 80 (HTTP)   | CIDR `0.0.0.0/0`          | Let's Encrypt HTTP-01 challenges and the 80 -> 443 redirect |
| 443 (HTTPS) | CIDR `0.0.0.0/0`          | All services                                              |

**`eden-service`** - attached to every service VM:

| Port                 | Remote                        | Why                                   |
| -------------------- | ----------------------------- | ------------------------------------- |
| 22 (SSH)             | Security group `eden-entry`   | SSH via the entry VM only             |
| 8000 (FFIS backend)  | Security group `eden-entry`   | nginx on the entry VM -> FFIS         |
| `<port>` (per service) | Security group `eden-entry` | One rule per backend port             |

In the dashboard: **your group -> Manage Rules -> Add Rule**, pick the port,
then **Remote: Security Group** and **Security Group: eden-entry**.

Nothing else gets ingress on the service VMs. A backend port opened in
`eden-service` applies to all service VMs, which is harmless: only the entry VM
can reach it.

### A3. Launch the entry VM

**Compute -> Instances -> Launch Instance**

- Details: name `eden-entry`
- Source: `Ubuntu 24.04 LTS`, Create New Volume: **Yes**, 20 GB
- Flavour: 2 vCPU / 2 GB RAM. nginx itself is light, but large uploads (FFIS
  accepts 100 MB) are buffered through this VM on their way to the backend.
- Networks: `eden-net`
- Security Groups: `eden-entry` (remove `default`)
- Key Pair: the entry VM owner's

### A4. Floating IP

**Network -> Floating IPs -> Allocate IP To Project**, pool
`PSNC-EXT-PUB1-EDU`, then **Associate Floating IP** on the `eden-entry` row.

The project has one floating IP. While the standalone `ffis` VM still holds
it, follow Part F instead of this step.

First boot:

```bash
ssh ubuntu@<entry-floating-IP>

sudo apt update && sudo apt -y upgrade
sudo apt -y install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

---

## Part B - SSH access for service admins

No new key pairs are needed. Admins keep their own keys; the entry VM only
forwards the TCP connection, and the private key never leaves the admin's
laptop.

### B1. A jump-only account

Service admins should not get the `ubuntu` account (which has sudo) on the
entry VM. Create a `jump` account that can only forward connections:

```bash
sudo adduser --disabled-password --gecos "" jump
sudo install -d -m 700 -o jump -g jump /home/jump/.ssh
sudo install -m 600 -o jump -g jump /dev/null /home/jump/.ssh/authorized_keys

sudo tee /etc/ssh/sshd_config.d/50-jump.conf >/dev/null <<'EOF'
Match User jump
    PermitTTY no
    X11Forwarding no
    AllowAgentForwarding no
    AllowTcpForwarding yes
    ForceCommand /usr/sbin/nologin
EOF
sudo sshd -t && sudo systemctl reload ssh
```

Add each admin's **public** key as one line in
`/home/jump/.ssh/authorized_keys`, with their name in the trailing comment so
keys can be removed when people leave. Also add their source IP to the
`eden-entry` port 22 rules.

### B2. Client config

Each admin adds this to `~/.ssh/config` on their own machine:

```
Host eden-entry
    HostName <entry-floating-IP>
    User jump

Host eden-ffis
    HostName 192.168.0.20
    User ubuntu
    ProxyJump eden-entry
```

Then `ssh eden-ffis` works directly, as does `scp`/`rsync`. One-off without a
config file: `ssh -J jump@<entry-floating-IP> ubuntu@192.168.0.20`.

Do not use agent forwarding (`ssh -A`) through the entry VM. `ProxyJump` does
not need it.

---

## Part C - nginx base config

### C1. Install

```bash
sudo apt -y install nginx
sudo systemctl enable --now nginx
```

### C2. Catch-all and global settings

Requests for hostnames the entry VM does not serve (scanners hitting the bare
IP, stale DNS) should be dropped, not routed to whichever service happens to
be first. Install the catch-all from this repository and remove the stock
default site:

```bash
sudo cp deploy/nginx-entry-default.conf /etc/nginx/sites-available/00-default
sudo ln -sf /etc/nginx/sites-available/00-default /etc/nginx/sites-enabled/00-default
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

`curl -I http://<entry-floating-IP>/` should now fail with an empty reply.
That is correct.

### C3. certbot

```bash
sudo snap install core; sudo snap refresh core
sudo snap install --classic certbot
sudo ln -sf /snap/bin/certbot /usr/local/bin/certbot
systemctl list-timers | grep -i certbot   # renewal timer is installed
```

---

## Part D - Onboarding a service

### D1. What the service team provides

| Item                    | FFIS example                               |
| ----------------------- | ------------------------------------------ |
| Hostname                | `eden-ffis.vm.fedcloud.eu`                 |
| Backend private IP:port | `192.168.0.20:8000`                        |
| nginx site file         | `deploy/nginx-ffis.conf` in this repository |
| Contact                 | The service team                           |

The site file carries the service-specific settings. Do not install a service
behind the entry VM with a generic `proxy_pass` only: nginx defaults to a
**1 MB** upload limit and a 60 second read timeout, which break FFIS uploads
and long identifications respectively.

### D2. Point the hostname at the entry VM

The hostname is registered at https://nsupdate.fedcloud.eu/ (EGI Check-in
login) by the service team, who keep its secret. They point it at the entry
VM's floating IP by passing `myip` explicitly:

```bash
curl "https://eden-ffis.vm.fedcloud.eu:<secret>@nsupdate.fedcloud.eu/nic/update?myip=<entry-floating-IP>"
```

`myip` matters. Without it, nsupdate records the source address of the
request. Run from a service VM, that is the router's SNAT address, not the
entry VM, and the hostname silently points at the wrong place.

Records do not expire, so this is a one-off unless the entry VM's floating IP
changes.

Verify before going further (certbot fails otherwise, and failures count
against Let's Encrypt rate limits):

```bash
dig +short eden-ffis.vm.fedcloud.eu    # must print the entry floating IP
```

### D3. Install the site file

On the entry VM, with the service's site file and its two placeholders
(hostname, backend address) filled in:

```bash
sudo cp nginx-ffis.conf /etc/nginx/sites-available/ffis
sudo sed -i 's/eden-ffis\.vm\.fedcloud\.eu/<hostname>/; s/192\.168\.0\.20:8000/<backend-ip>:<port>/' \
  /etc/nginx/sites-available/ffis
sudo ln -sf /etc/nginx/sites-available/ffis /etc/nginx/sites-enabled/ffis
sudo nginx -t && sudo systemctl reload nginx
```

Check the backend is reachable from the entry VM before involving certbot.
If this times out, the `eden-service` rule for the backend port is missing:

```bash
curl -s http://192.168.0.20:8000/health
```

### D4. Certificate

```bash
sudo certbot certonly --nginx -d eden-ffis.vm.fedcloud.eu --dry-run
sudo certbot --nginx -d eden-ffis.vm.fedcloud.eu \
  --agree-tos --no-eff-email -m <owner-email> --redirect
```

### D5. Verify from outside

```bash
curl -I http://eden-ffis.vm.fedcloud.eu/     # 301 to https
curl -s https://eden-ffis.vm.fedcloud.eu/health
```

### D6. Offboarding

```bash
sudo rm /etc/nginx/sites-enabled/<service> /etc/nginx/sites-available/<service>
sudo certbot delete --cert-name <hostname>
sudo nginx -t && sudo systemctl reload nginx
```

Ask the service team to delete or repoint the hostname in nsupdate.

---

## Part E - Operations

- **Single point of failure.** If the entry VM is down, every EDEN service is
  unreachable from outside, even though the service VMs are fine. Acceptable
  for the pilot phase; revisit before anything production-like.
- **Backups.** The state worth keeping is `/etc/nginx/sites-available/` and
  `/etc/letsencrypt/`. Everything else is reproducible from this runbook.
  Certificates can be reissued, but keep under the Let's Encrypt rate limits.
- **Logs.** Each site file sets its own `access_log`/`error_log` under
  `/var/log/nginx/<service>.*.log`. Service teams without an account on the
  entry VM have to ask the owner for them.
- **Changes to a site file** come from the service team (ideally as a change
  to the file in their repository) and are installed by the entry VM owner.
  Always `sudo nginx -t` before reloading: a syntax error in one service's file
  stops nginx reloading for all of them.

### Troubleshooting

| Symptom                                 | Likely cause                                                        |
| --------------------------------------- | ------------------------------------------------------------------- |
| Empty reply / connection closed         | Hostname not in any `server_name`; the catch-all dropped it          |
| 502 from nginx                          | Backend down, or bound to loopback instead of its private IP         |
| 504, or `curl` to backend times out     | `eden-service` lacks a rule for the backend port from `eden-entry`   |
| certbot HTTP-01 fails                   | Hostname does not yet resolve to the entry IP (check `myip`)         |
| Hostname resolves to an unexpected IP   | nsupdate was called from a service VM without `myip`                 |
| `ssh -J` fails at the second hop        | `eden-service` lacks port 22 from `eden-entry`, or wrong user/key    |
| 413 on upload                           | Site file `client_max_body_size` missing or too low                  |

---

## Part F - Cutover plan (from the standalone FFIS VM)

**Current state (2026-11-02).** The project has one VM, `ffis` (1 vCPU,
4 GB), holding the project's only floating IP (`62.3.175.206`, which
`eden-ffis.vm.fedcloud.eu` resolves to). It runs its own nginx, certbot and
DDNS cron. That standalone setup is retired: the VM is replaced, not
converted.

**Target state.**

| VM           | Network   | Security group  | Floating IP      | Runs                          |
| ------------ | --------- | --------------- | ---------------- | ----------------------------- |
| `eden-entry` | `eden-net` | `eden-entry`   | `62.3.175.206`   | nginx, certbot, `jump` account |
| `ffis`       | `eden-net` | `eden-service` | none             | FFIS in Docker, port 8000     |
| (next services) | `eden-net` | `eden-service` | none          | see Part G                    |

The floating IP address stays the same throughout, so no hostname has to be
repointed. FFIS is down only for the switch in F3 (minutes).

### F1. Preparation (no downtime)

1. Open the EGI Helpdesk ticket for `services.eden-fidelis.eu`
   ([project-domain.md](project-domain.md), step 2). Trust-IT's delegation is
   live since 2026-10-19, but EGI's name servers still refuse the zone.
   Nothing below waits for this.
2. Rename the old instance to `ffis-standalone`
   (**Instances -> Edit Instance**) so the new one can be called `ffis`.
3. Create the `eden-entry` and `eden-service` security groups (A2).
4. Add a **temporary** rule to `eden-entry`: port 22 from security group
   `ffis-web` (the old VM's group). Delete it in F4.

### F2. Build the new VMs (no downtime)

1. Launch `eden-entry` (A3), **no floating IP**.
2. Launch the new `ffis` (deployment.md, A5): 2 vCPU / 4 GB, `eden-service`,
   no floating IP.
3. Until the switch, the old VM is the way in:

   ```
   Host eden-old
       HostName 62.3.175.206
       User ubuntu

   Host eden-entry-tmp
       HostName <entry-private-IP>
       User ubuntu
       ProxyJump eden-old

   Host eden-ffis-tmp
       HostName <ffis-private-IP>
       User ubuntu
       ProxyJump eden-entry-tmp
   ```

4. On `eden-entry`: Parts B and C (jump account, nginx, catch-all, certbot
   install). Install the FFIS site file (D3) but do not request a
   certificate yet; HTTP-01 cannot reach this VM before it has the IP.
5. On `ffis`: deployment.md, Part D (Docker, code, `FFIS_BIND_ADDR`). Check
   from `eden-entry`:

   ```bash
   curl -s http://<ffis-private-IP>:8000/health
   ```

### F3. Switch (FFIS down for a few minutes)

1. **Network -> Floating IPs**: disassociate `62.3.175.206` from
   `ffis-standalone`, associate it with `eden-entry`.
2. From your laptop, with the normal SSH config (B2): `ssh eden-entry`.
3. Issue the certificate (D4) and verify from outside (D5).

If something fails here, roll back by moving the IP back to
`ffis-standalone`; it is untouched.

### F4. Remove the standalone setup

After a few days without problems:

1. Delete the temporary rule from F1.4.
2. Delete the `ffis-standalone` instance **and its volume**, then the
   `ffis-web` security group.
3. Remove `/etc/cron.weekly/ffis-ddns` with it (it goes with the VM). The
   nsupdate secret stays valid; keep it with the FFIS team.

### F5. Project domain (once EGI has created the zone)

Follow [project-domain.md](project-domain.md) steps 4 to 7:
register `ffis.services.eden-fidelis.eu`, point it at `62.3.175.206`, add it
to the FFIS site file and certificate. Only the entry VM changes.

### F6. Open question: the harvester

`eden-harvester.vm.fedcloud.eu` resolves to `62.3.175.110`, which is not this
project's floating IP, so the harvester VM lives in another OpenStack
project. Service VMs are only reachable over `eden-net` from inside this
project. Either:

- **Recommended:** the harvester team rebuilds it in this project as a
  service VM (Part G), which also frees their floating IP; or
- the entry VM proxies to the harvester's public address. This works, but
  the harvester keeps a public IP and the backend traffic leaves the cloud
  unencrypted unless the harvester also has TLS.

---

## Part G - Checklist for a new service team

For someone who wants their own VM behind the entry VM. Nobody needs their
own floating IP.

1. **Key and access.** Send the entry VM owner your SSH public key and your
   current public IPv4 (`curl -4 ifconfig.me`). They add you to the `jump`
   account and to the `eden-entry` port 22 rules (B1).
2. **Launch your VM.** Network `eden-net`, security group `eden-service`
   only, your own key pair, no floating IP. Note the private IP.
3. **Backend port.** If your service listens on something other than an
   already allowed port, ask for an `eden-service` rule for it from
   `eden-entry` (A2). Bind your service to the private IP, not `0.0.0.0` and
   not loopback.
4. **SSH config** on your laptop:

   ```
   Host eden-entry
       HostName <entry-floating-IP>
       User jump

   Host eden-<service>
       HostName <your-private-IP>
       User ubuntu
       ProxyJump eden-entry
   ```

   `ssh eden-<service>` then works. Outbound internet (apt, docker pull)
   works through the router.
5. **Hostname.** Register `eden-<service>.vm.fedcloud.eu` in nsupdate and
   point it at the entry IP with `myip` (D2).
6. **Site file.** Write an nginx site file for your service, using
   `deploy/nginx-ffis.conf` as the template (upload limit, timeouts, a
   `<service>_` prefix on rate limit zones), and send it with the hostname
   and backend `IP:port` to the entry VM owner (D1).
