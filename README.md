# Lab 01 — CI/CD Pipeline with Docker Swarm

Deploy the [Strong Password Generator API](https://github.com/redhat-ops/test-api) using Docker Swarm, GitHub Actions, and expose it via Cloudflare + HAProxy.

---

## Prerequisites

- GitHub account with access to [redhat-ops](https://github.com/redhat-ops) organization
- SSH access to 2 Linux VMs (Ubuntu 22.04+ or RHEL 9+)
- Docker Hub or GHCR account for container registry
- Cloudflare account with a managed domain
- Basic knowledge of Docker, Git, and Linux networking

---

## Tasks

### Task 1 — Provision 2 VMs

1. Create 2 Linux VMs (cloud or on-prem)
2. Assign static IPs or configure DNS records
3. Open required ports between VMs:
   - `2377/tcp` — Swarm management
   - `7946/tcp+udp` — node communication
   - `4789/udp` — overlay network
   - `80/tcp`, `443/tcp` — HTTP/HTTPS traffic
4. Install Docker Engine on both VMs
5. Verify: `docker --version` works on both

---

### Task 2 — Create Docker Swarm Cluster

1. On **VM-1** (manager), initialize Swarm:
   ```bash
   docker swarm init --advertise-addr <VM1_IP>
   ```
2. Copy the join token from the output
3. On **VM-2** (worker), join the Swarm:
   ```bash
   docker swarm join --token <TOKEN> <VM1_IP>:2377
   ```
4. Verify on manager:
   ```bash
   docker node ls
   ```
   Both nodes should appear as `Ready`

---

### Task 3 — Write Dockerfile for test-api

1. Fork or clone [test-api](https://github.com/redhat-ops/test-api)
2. Create a `Dockerfile` in the repo root:
   - Use multi-stage build (SDK for build, ASP.NET runtime for run)
   - Expose port `8080`
   - Copy the wordlist file
3. Test locally:
   ```bash
   docker build -t test-api .
   docker run -p 8080:8080 test-api
   curl http://localhost:8080/swagger
   ```

---

### Task 4 — Set Up GitHub Actions CI/CD

1. Create `.github/workflows/deploy.yml` in the test-api repo
2. **Build & Test** stage:
   - Checkout code
   - Setup .NET SDK
   - Run `dotnet restore`, `dotnet build`, `dotnet test`
3. **Build & Push Docker Image** stage:
   - Build Docker image
   - Tag with commit SHA and `latest`
   - Push to container registry (Docker Hub or GHCR)
4. **Deploy to Swarm** stage:
   - SSH into VM-1 (manager)
   - Pull the new image
   - Update the Swarm service
5. Add required secrets in GitHub repo settings:
   - `REGISTRY_USERNAME` / `REGISTRY_PASSWORD`
   - `SWARM_HOST` / `SWARM_SSH_KEY`
6. Trigger: push to `main` branch

---

### Task 5 — Deploy test-api to Docker Swarm

1. On the Swarm manager, create the service:
   ```bash
   docker service create \
     --name test-api \
     --replicas 2 \
     --publish 8080:8080 \
     <registry>/test-api:latest
   ```
2. Verify:
   ```bash
   docker service ls
   docker service ps test-api
   curl http://<VM1_IP>:8080/api/v1/passwords/capabilities
   ```
3. Test rolling updates work when a new image is pushed

---

### Task 6 — Configure HAProxy Load Balancer

1. Install HAProxy on VM-1:
   ```bash
   sudo apt install haproxy    # Ubuntu
   sudo dnf install haproxy    # RHEL
   ```
2. Configure `/etc/haproxy/haproxy.cfg`:
   ```
   frontend http_front
       bind *:80
       default_backend api_back

   backend api_back
       balance roundrobin
       server vm1 <VM1_IP>:8080 check
       server vm2 <VM2_IP>:8080 check
   ```
3. Restart and verify:
   ```bash
   sudo systemctl restart haproxy
   curl http://<VM1_IP>/api/v1/passwords/capabilities
   ```

---

### Task 7 — Expose via Cloudflare

1. Add a DNS A record in Cloudflare pointing to VM-1's public IP:
   - Name: `api` (or your chosen subdomain)
   - Proxy status: **Proxied** (orange cloud)
2. Set SSL/TLS mode to **Full** in Cloudflare dashboard
3. Install a Cloudflare Origin Certificate on VM-1
4. Update HAProxy to terminate TLS with the origin cert:
   ```
   frontend https_front
       bind *:443 ssl crt /etc/ssl/cloudflare/cert.pem
       default_backend api_back
   ```
5. Verify the API is accessible:
   ```bash
   curl https://api.yourdomain.com/swagger
   curl -X POST https://api.yourdomain.com/api/v1/passwords/generate \
     -H "Content-Type: application/json" \
     -d '{"method":"policy","length":20,"count":1}'
   ```

---

## Deliverables

- [ ] 2 VMs provisioned and Docker installed
- [ ] Docker Swarm cluster running with 2 nodes
- [ ] Dockerfile builds and runs test-api correctly
- [ ] GitHub Actions pipeline: build → test → push image → deploy
- [ ] test-api running as a Swarm service with 2 replicas
- [ ] HAProxy load balancing across both nodes
- [ ] API publicly accessible via Cloudflare with HTTPS

---

## Architecture

```
Internet → Cloudflare (DNS + CDN + SSL)
              ↓
         VM-1 (Manager)
         ├── HAProxy :443 → :8080
         ├── test-api replica 1
         └── Docker Swarm Manager
              ↓
         VM-2 (Worker)
         └── test-api replica 2
```
