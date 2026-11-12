# Deploying FFIS

FFIS runs in Docker on the service VM `ffis`, behind the EDEN entry VM. The
entry VM, SSH access and the hostname are documented in
[eden-infra](https://github.com/EOSC-EDEN/eden-infra/blob/main/docs/entry-vm.md).
This document covers the FFIS-specific steps.

| Item          | Value                      |
| ------------- | -------------------------- |
| VM            | `ffis`                     |
| Private IP    | `192.168.0.203`            |
| Flavour       | `M1-NET-2vCPU-8R`          |
| Port          | 8000                       |
| Hostname      | `eden-ffis.vm.fedcloud.eu` |
| Code          | `/opt/ffis`                |
| SSH           | `ssh -J jump@62.3.175.206 ubuntu@192.168.0.203` |

## 1. VM

Follow [eden-infra, Part A](https://github.com/EOSC-EDEN/eden-infra/blob/main/docs/entry-vm.md#part-a---deploying-a-new-service)
with these values:

| Field        | Value  |
| ------------ | ------ |
| Name         | `ffis` |
| Minimum vCPU | 2      |
| Minimum RAM  | 4 GB   |
| Volume       | 40 GB  |
| Minimum volume | 20 GB |
| Port         | 8000   |

`eden-service` already allows port 8000 from the entry VM.

Memory use:

| Item                               | Value           |
| ---------------------------------- | --------------- |
| uvicorn workers                    | 2               |
| Upload limit per request           | 100 MB          |
| Upload buffers, worst case         | about 200 MB    |
| Magika model                       | in memory       |

The image is 2 to 3 GB. Build cache, logs and the result cache use the rest
of the volume.

## 2. Docker

```bash
sudo apt -y install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null
sudo apt update
sudo apt -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker ubuntu   # log out and back in
```

## 3. Start FFIS

```bash
sudo mkdir -p /opt/ffis && sudo chown ubuntu:ubuntu /opt/ffis
git clone https://github.com/EOSC-EDEN/eden-ffis.git /opt/ffis
cd /opt/ffis

echo "FFIS_BIND_ADDR=192.168.0.203" > deploy/.env
docker compose --env-file deploy/.env -f deploy/docker-compose.prod.yml up -d --build
```

The first image build downloads the Siegfried binary, the PRONOM signature
file (`sf -update`), Magika and onnxruntime, and takes several minutes.

`deploy/docker-compose.prod.yml` publishes port 8000 on `FFIS_BIND_ADDR`
only. Docker inserts its iptables rules before the host firewall, so a port
published on `0.0.0.0` is open on every interface. Without `FFIS_BIND_ADDR`,
the port is published on loopback and the entry VM returns 502.
`deploy/.env` is in `.gitignore`.

```bash
docker compose --env-file deploy/.env -f deploy/docker-compose.prod.yml ps
curl -s 192.168.0.203:8000/health
curl -s 192.168.0.203:8000/tools | head
```

`curl localhost:8000` fails, because the port is published on the private IP
only.

## 4. Publish

Send an entry VM admin the hostname, `192.168.0.203:8000` and
[`deploy/nginx-ffis.conf`](../deploy/nginx-ffis.conf). The admin follows
[eden-infra, Part B](https://github.com/EOSC-EDEN/eden-infra/blob/main/docs/entry-vm.md#part-b---publishing-a-service).

| Setting in `nginx-ffis.conf`         | Value   | Reason                                         |
| ------------------------------------ | ------- | ---------------------------------------------- |
| `client_max_body_size`               | `100m`  | nginx default is 1 MB. Must equal `FFIS_MAX_UPLOAD_BYTES` |
| `proxy_read_timeout` on `/identify`  | `180s`  | Identification of a large file takes longer than the 60 s default |
| `limit_req` rate on `/identify`      | 2 r/s   | Per client IP                                  |
| `limit_req` burst on `/identify`     | 10      | Per client IP                                  |
| `upstream ffis_backend`              | `192.168.0.203:8000` | Must equal `FFIS_BIND_ADDR`       |

## 5. Verify

```bash
curl -I https://eden-ffis.vm.fedcloud.eu/                    # 200
curl -s https://eden-ffis.vm.fedcloud.eu/health              # {"status":"ok",...}
curl -s -o /dev/null -w '%{http_code}\n' https://eden-ffis.vm.fedcloud.eu/docs   # 200
curl -s -X POST https://eden-ffis.vm.fedcloud.eu/identify -F "file=@some-document.pdf" | head -40
```

## 6. Operations

Update:

```bash
cd /opt/ffis
git pull
docker compose --env-file deploy/.env -f deploy/docker-compose.prod.yml up -d --build
```

- **PRONOM signatures.** The signature file is fetched at build time. Rebuild
  the image monthly to update the signatures.
- **Logs.** `docker compose --env-file deploy/.env -f deploy/docker-compose.prod.yml logs -f ffis`.
  Container logs are limited to 3 files of 10 MB. The nginx logs are on the
  entry VM.
- **Reboots.** `restart: unless-stopped` restarts the container. On Ubuntu
  24.04, `docker.service` starts after `network-online.target`, so the
  private IP is available. Check the container after the first reboot.
- **nginx changes.** `git pull` on the FFIS VM leaves the entry VM unchanged.
  An entry VM admin installs a new `nginx-ffis.conf`.

## 7. Security

| Item                     | Setting                                                             |
| ------------------------ | ------------------------------------------------------------------- |
| `/identify/path`         | Denied while `FFIS_ALLOWED_PATH_PREFIXES` is unset. Keep `FFIS_ALLOWED_PATH_PREFIXES` unset on a public instance, because the prefix list is the only limit on local file reads |
| CORS                     | `allow_origins=["*"]` in `src/ffis/main.py`. Narrow the allowed origins if authentication is added |
| Authentication           | None. The nginx rate limit limits CPU use. For a private pilot, add an IP allowlist or basic auth to `location ~ ^/identify` |
| Port 8000                | Plain HTTP, open to the entry VM only through `eden-service`        |
| Upload memory            | `/identify` reads the whole upload into memory (`src/ffis/api/routes.py`). Lower `FFIS_MAX_UPLOAD_BYTES` for small files |

## 8. Troubleshooting

General entry VM problems: [eden-infra, Part D](https://github.com/EOSC-EDEN/eden-infra/blob/main/docs/entry-vm.md#part-d---operations).

| Symptom                                      | Cause                                                  |
| -------------------------------------------- | ------------------------------------------------------ |
| 502                                          | Container down, or `FFIS_BIND_ADDR` unset              |
| 504 on a large file                          | `proxy_read_timeout` too low on `/identify`            |
| 413 on upload                                | `client_max_body_size` below `FFIS_MAX_UPLOAD_BYTES`   |
| 429 on `/identify`                           | nginx rate limit                                       |
| Build fails at `sf -update`                  | PRONOM download failed; run the build again            |
| `cannot assign requested address` after reboot | Docker started before the private IP was up; run `docker compose up -d` and check `systemctl cat docker` |
