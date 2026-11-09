# Project-02: Deploy a Dockerised App with Jenkins CI/CD on AWS EC2

A React + TypeScript Kanban dashboard, containerised with Docker and deployed to AWS EC2 through a fully automated Jenkins pipeline. Every `git push` to `main` builds a new image, pushes it to Docker Hub, deploys it on EC2 with a blue-green switch, health-checks it and rolls back automatically on failure.

**Author:** Anjanvignesh

---

## 1. Live links

| Item | Link |
|---|---|
| Application | http://3.111.73.151 |
| Jenkins | http://3.111.73.151:8080 |
| GitHub repository | https://github.com/anjan2701/kanban-dashboard |
| Docker Hub repository | https://hub.docker.com/r/anjan27/kanban-dashboard |
| Original source repo | https://github.com/Vennilavanguvi/kanban-dashboard |

> The EC2 address is an Elastic IP. It will be released after evaluation to avoid AWS charges.

---

## 2. Architecture

```
Developer ──git push──▶ GitHub ──webhook──▶ Jenkins (EC2, :8080)
                                                │
                     ┌──────────────────────────┤
                     ▼                          ▼
              docker build              docker push
              (tag = build no.)         ▶ Docker Hub (anjan27/kanban-dashboard:<n>)
                     │
                     ▼
        start NEW colour container (blue or green)
                     │
                     ▼
        Docker HEALTHCHECK + HTTP check on NEW container
                     │ healthy?
          ┌──────────┴───────────┐
          ▼ yes                  ▼ no
   nginx proxy switches     traffic never moves,
   :80 → NEW colour         old version keeps serving,
   stop OLD colour          build marked FAILED
```

Flow: **GitHub → Jenkins → Build → Docker image → Registry → EC2 → Docker deployment → Validation.** There are no manual deployment steps once the pipeline is live.

---

## 3. Application details

| Item | Value |
|---|---|
| Stack | React 18, TypeScript, Tailwind CSS, Vite |
| Backend / database | None (front-end only) |
| Dev server command | `npm run dev` (port **5173**) |
| Build command | `npm run build` (output in `dist/`) |
| Port inside container | **8080** (nginx, non-root) |
| Environment variables | None required |

---

## 4. Repository structure

```
kanban-dashboard/
├── src/                 # React application source
├── Dockerfile           # Multi-stage, non-root image
├── .dockerignore        # Keeps secrets and junk out of the image
├── Jenkinsfile          # Declarative CI/CD pipeline
├── package.json
└── README.md
```

---

## 5. Run locally (no Docker)

```bash
git clone https://github.com/anjan2701/kanban-dashboard.git
cd kanban-dashboard
npm install
npm run dev
# open http://localhost:5173
curl -I http://localhost:5173      # expect HTTP/1.1 200 OK
```

---

## 6. Docker

### Multi-stage Dockerfile

1. **Build stage**: `node:20-alpine` installs dependencies (`npm ci`) and runs `npm run build`.
2. **Runtime stage**: `nginxinc/nginx-unprivileged:alpine` receives only the built `dist/` files. No Node, no source code and no `node_modules` end up in the final image (about 25 MB).

Other Dockerfile choices:

- `EXPOSE 8080` only.
- `HEALTHCHECK` that requests `http://127.0.0.1:8080/` every 30 seconds.
- `USER nginx` so the process never runs as root.

### Build and run

```bash
# On an Apple Silicon Mac, build for the amd64 EC2 server:
docker buildx build --platform linux/amd64 -t kanban-dashboard:1.0 --load .
docker run -d --name kanban-test -p 8080:8080 kanban-dashboard:1.0
# open http://localhost:8080
docker ps          # STATUS shows (healthy) after about 30-60 s
```

### Security measures

| Requirement | How it is met |
|---|---|
| Minimal base image | Alpine-based `node` and `nginx-unprivileged` images |
| Non-root user | `USER nginx`; verified with `docker inspect -f '{{.Config.User}}'` and `docker exec <c> whoami` |
| No secrets in image | Nothing sensitive in the Dockerfile or build args; `.env` excluded by `.dockerignore`; `docker history` shows clean layers |
| `.dockerignore` | `node_modules`, `dist`, `.git`, `.gitignore`, `.env`, `*.log`, `Dockerfile`, `README.md`, `Jenkinsfile` (build context dropped from 171 MB to a few MB) |
| Unused ports / packages | Only port 8080 is exposed; the runtime stage contains no build tools |
| Credentials | Stored only in the Jenkins Credentials store; never in code or console logs |

---

## 7. Container registry and versioning

- Registry: **Docker Hub**, repository `anjan27/kanban-dashboard`.
- Every pipeline run pushes a tag equal to the Jenkins **build number** (`:1`, `:2`, `:3` …), so `latest` is never relied on.
- Authentication uses a Docker Hub **personal access token** (Read & Write), not the account password.
- The pushed image was pulled and run on EC2 first to prove it works on a clean machine.

---

## 8. AWS infrastructure

| Setting | Value |
|---|---|
| Region | Asia Pacific (Mumbai) `ap-south-1` |
| Instance | `t3.small` (2 vCPU, 2 GB RAM), Ubuntu 24.04 LTS, 20 GiB gp3 |
| Public address | Elastic IP (stays fixed when the instance restarts) |
| Installed | Docker (`docker.io`), OpenJDK 21, Jenkins (LTS) |

### Security group (inbound)

| Port | Source | Purpose |
|---|---|---|
| 22 (SSH) | My IPv4 address only (`/32`) | Administration |
| 8080 | 0.0.0.0/0 | Jenkins UI and GitHub webhook |
| 80 | 0.0.0.0/0 | Application |

Trade-off: port 8080 must be reachable from the internet for GitHub to deliver webhooks. A stricter setup would allow only GitHub's published webhook IP ranges and add HTTPS.

---

## 9. Jenkins configuration

- Plugins: Git, Credentials Binding, **Docker Pipeline**, GitHub (plus the suggested set).
- Credential: `dockerhub-creds` (username with password, where the password is the Docker Hub token).
- The `jenkins` user is in the `docker` group so pipeline steps can run Docker commands.
- Job `kanban-pipeline`: **Pipeline script from SCM** → Git → `https://github.com/anjan2701/kanban-dashboard.git` → branch `*/main` → script path `Jenkinsfile`.
- Build trigger: **GitHub hook trigger for GITScm polling**.

---

## 10. CI/CD pipeline (Jenkinsfile)

| # | Stage | What it does |
|---|---|---|
| 1 | Checkout | Pulls the source from GitHub |
| 2 | Build Image | `docker build` and tag with the build number |
| 3 | Push to Docker Hub | Logs in with the stored credential and pushes the versioned tag |
| 4 | Deploy (new colour) | Starts the new version as `kanban-blue` or `kanban-green` beside the live one, with CPU and memory limits |
| 5 | Health Check | Waits for Docker `healthy` status and makes a direct HTTP request to the new container |
| 6 | Switch Traffic | Points the nginx proxy at the new container and reloads it gracefully |
| 7 | Verify | Requests the app through port 80 to confirm end-to-end success |
| 8 | Cleanup Old Version | Removes the previous container only after everything passed |
| Post | Success / Failure | Prints pass or fail; on failure the previous version is kept (or restored) |

A deployment is only marked successful if **all** checks pass. A running container alone is not success.

---

## 11. GitHub webhook

- GitHub repo → **Settings → Webhooks** → Payload URL `http://3.111.73.151:8080/github-webhook/`, content type `application/json`, event: push.
- Result: a `git push` to `main` starts a Jenkins build with no manual click. The build page says "Started by GitHub push".

---

## 12. Validation, logging and monitoring

```bash
docker ps                                                  # container running, STATUS (healthy)
docker inspect -f '{{.State.Health.Status}}' kanban-blue   # or kanban-green
docker logs --tail 50 kanban-blue
curl -I http://localhost                                   # expect 200 OK
docker stats --no-stream
```

- **Jenkins build history** shows a passed run and a deliberately failed run.
- **Console Output** of the failed build shows the `Health:` lines and `DEPLOYMENT FAILED`, which pinpoints the failure.
- **Failure test performed:** the Dockerfile health check URL was changed to a non-existent path and pushed. Build #3 failed at the Health Check stage and the previous version kept serving. Reverting the change produced a green build #4.

---

## 13. Docker resource limits

Applied to every app container:

```
--memory=256m --cpus=0.5
```

(The small nginx proxy gets `--memory=64m --cpus=0.25`.)

**Why these numbers:** the app is static files served by nginx, which normally uses about 10-30 MB. The `t3.small` has 2 vCPUs and 2 GB RAM, and Jenkins plus Docker builds use most of it. During a deployment two app containers and the proxy run together (about 576 MB at the caps), which fits comfortably. The caps stop a runaway container from starving Jenkins or the host. Verified with `docker stats` and `docker inspect` (`Memory=268435456`, `NanoCpus=500000000`).

---

## 14. Deployment strategy: blue-green (bonus)

An nginx proxy container (`kanban-proxy`) owns port 80 and forwards to the active colour on a private Docker network (`kanban-net`).

1. Active colour is read from a small state file (`$JENKINS_HOME/proxy/active`).
2. The new version starts as the other colour. The old one keeps serving users.
3. The new container must pass the Docker health check and a direct HTTP check **before** it gets any traffic.
4. The proxy config is rewritten and `nginx -s reload` switches traffic gracefully with no dropped connections.
5. A final check goes through port 80. Only then is the old container removed.

**Rollback:** if the new container is unhealthy, traffic is never switched and the old version continues serving. If a check fails after the switch, the proxy is pointed back to the old colour and the new container is removed.

**Note:** the very first run on a server that used the earlier single-container setup frees port 80 once, which causes a brief gap. Every deployment after that is zero-downtime.

**Zero-downtime test:** run this while a deployment is in progress and every line stays `200`:

```bash
while true; do curl -s -o /dev/null -w "%{http_code}\n" http://3.111.73.151/; sleep 0.5; done
```

---

## 15. Troubleshooting log (problems solved)

| Problem | Cause | Fix |
|---|---|---|
| Container stayed `unhealthy` although the app worked | The health check used `localhost`, which failed inside the container (`wget: Connection refused`) while nginx answered on IPv4 | Changed the health check to `127.0.0.1` |
| `docker build` context was 171 MB | `node_modules` and `.git` were being sent | Added `.dockerignore` |
| Possible `exec format error` on EC2 | Image built on an Apple Silicon (arm64) Mac | Built with `--platform linux/amd64` |
| `git push` rejected | GitHub no longer accepts passwords for Git | Used a Personal Access Token |
| SSH to EC2 timed out | Security group SSH rule held an IPv6 range instead of my IPv4 address | Replaced it with my IPv4 `/32` |
| Browser "EC2 Instance Connect" failed | SSH rule intentionally allows only my IP | Used SSH from my own terminal |
| Jenkins service crashed on start | Installed Java version was not supported by this Jenkins release | Installed OpenJDK 21 and made it the default |
| Build stuck at "Waiting for next available executor" | Jenkins marked the built-in node offline because free temp space was under 1 GiB | Ticked "Don't mark agents temporarily offline" on the temp-space monitor and brought the node back online |
| Jenkins unreachable after attaching the Elastic IP | The old auto-assigned public IP is released | Used the new address; updated the Jenkins URL and webhook |

---

## 16. Evidence (screenshots)

Screenshots are stored in `docs/screenshots/`.

| # | Evidence | File |
|---|---|---|
| 1 | App running locally in the browser | `01-app-local.png` |
| 2 | `curl` response from the local app | `02-curl-local.png` |
| 3 | Successful `docker build` | `03-docker-build.png` |
| 4 | Container serving the app, `docker ps` healthy | `04-container-healthy.png` |
| 5 | Non-root proof (`docker inspect`, `whoami`) | `05-non-root.png` |
| 6 | `.dockerignore` contents | `06-dockerignore.png` |
| 7 | Docker Hub repository with versioned tags | `07-dockerhub-tags.png` |
| 8 | EC2 instance running | `08-ec2-running.png` |
| 9 | Security group inbound rules | `09-security-group.png` |
| 10 | App served from the EC2 public IP | `10-app-on-ec2.png` |
| 11 | Jenkins plugins installed | `11-jenkins-plugins.png` |
| 12 | Jenkins credentials list | `12-jenkins-credentials.png` |
| 13 | Pipeline job configuration | `13-job-config.png` |
| 14 | Successful run, all stages green | `14-stages-green.png` |
| 15 | Console output ending in `Finished: SUCCESS` | `15-console-success.png` |
| 16 | Docker Hub tags after pipeline runs | `16-dockerhub-pipeline-tags.png` |
| 17 | GitHub webhook with green tick | `17-webhook.png` |
| 18 | Build "Started by GitHub push" | `18-webhook-build.png` |
| 19 | `docker ps`, `docker logs`, health status | `19-validation.png` |
| 20 | Failed build with the red Health Check stage | `20-failed-stage.png` |
| 21 | Build history: red then green | `21-build-history.png` |
| 22 | `docker stats` / `docker inspect` resource limits | `22-resource-limits.png` |
| 23 | Blue-green deploy with zero-downtime curl loop | `23-zero-downtime.png` |

---

## 17. Clean-up (after evaluation)

1. EC2 → Instances → **Terminate** (or Stop) `jenkins-server`.
2. EC2 → Elastic IPs → **Release** the address.
3. Delete the Docker Hub access token and the GitHub token if no longer needed.
