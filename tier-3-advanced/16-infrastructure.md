# Infrastructure

## What is it?

Infrastructure is the underlying compute, storage, and networking layer that runs your application. Before containers, deploying an app meant renting a server (physical or virtual), installing your runtime (Python, Node, etc.), configuring system-level settings, and hoping your local environment matched production closely enough that things didn't break at 2am.

Cloud providers abstract this. Instead of buying servers, you rent them by the minute. Instead of configuring hardware, you configure services. Instead of guessing what "large enough" means, you scale up or down automatically.

**IaaS (Infrastructure as a Service)** — You rent raw compute: virtual machines, block storage, networking. AWS EC2, GCP Compute Engine, Azure VMs. You control the OS and are responsible for everything above it (runtime, libraries, app code).

**PaaS (Platform as a Service)** — You deploy code, the platform handles the runtime and scaling. Heroku, Railway, Render, Cloudflare Workers, AWS Elastic Beanstalk. You don't think about servers — you think about your application and its resource needs.

**FaaS (Function as a Service)** — You write discrete functions that run on-demand. AWS Lambda, GCP Cloud Functions, Azure Functions. You pay per invocation and never think about servers at all.

Think of it like housing:

- **IaaS** is owning a house. You handle the roof, plumbing, everything.
- **PaaS** is renting a furnished apartment. Someone else handles the furnace, but you arrange the furniture.
- **FaaS** is staying in a hotel. A room appears when you need it, disappears when you don't.

## Why it matters for vibe coding

AI code generators produce Dockerfiles, GitHub Actions workflows, and infrastructure configs constantly. Without a mental model for what's happening, you can't debug when things break — and they will break.

**Without infrastructure knowledge, AI will expose your secrets.** AI generates Dockerfiles that bake API keys into container images, then pushes those images to public registries. Every secret in a Dockerfile is public forever.

**Without containers knowledge, you won't understand why your app works locally but fails in CI.** The AI generated a Python script that imports `psycopg2`. It works on your machine because you installed it. In the Docker container, it's not there. The error message is cryptic. You need to know what a Dockerfile does to fix it.

**Without environment parity understanding, you'll chase phantom bugs.** The AI generated a service that reads `DATABASE_URL` from the environment. It worked in testing with a mock. In production, the variable is named `POSTGRES_CONNECTION_STRING`. The app starts, connects to nothing, and fails silently.

**Without Kubernetes basics, you can't read AI-generated deployment configs.** AI will produce a Kubernetes manifest. You deploy it. It sits in `Pending` state forever because you didn't know about `nodeSelector` or `resource limits`. You need to know what the AI generated to know why it's failing.

## The 20% you need to know

### Containers and Docker

A container is an isolated process with its own filesystem, network, and process space. It's not a virtual machine — it's lighter. Multiple containers share the host kernel but stay separated from each other. Think of it like shipping containers on a cargo ship: standardized, isolated, moveable.

**Image** — The template your container starts from. Immutable. Built from a Dockerfile. Contains your app and all its dependencies.

**Container** — A running instance of an image. Writable layer on top of a read-only image. You can run many containers from the same image.

**Dockerfile** — A recipe for building an image. Instructions like `FROM python:3.11`, `COPY . .`, `RUN pip install -r requirements.txt`, `CMD ["python", "app.py"]`.

```
# Dockerfile anatomy
FROM python:3.11              # Start with Python 3.11 image
WORKDIR /app                 # Set working directory
COPY requirements.txt .      # Copy dependency list
RUN pip install -r requirements.txt  # Install deps into image
COPY . .                     # Copy application code
EXPOSE 3000                 # Document the port
CMD ["python", "app.py"]    # Command to run
```

The `FROM` line pulls a base image. Every Dockerfile starts with one. The image layers your app on top of the base runtime.

**Key Docker commands:**

```bash
docker build -t myapp .           # Build image from Dockerfile in current dir
docker run -p 3000:3000 myapp     # Run container, map port 3000 to host 3000
docker ps                         # List running containers
docker logs <container_id>        # View container logs
docker stop <container_id>        # Stop a running container
docker exec -it <container_id> sh # Shell into running container
```

### Port mappings

When you run `docker run -p 3000:3000 myapp`, the left side is the **host port** and the right side is the **container port**. The container thinks it's listening on port 3000. The host maps external port 3000 to that container port.

You can map to different ports: `docker run -p 8080:3000 myapp` means traffic to host port 8080 goes to container port 3000. This is how you run multiple instances of the same container on one host.

AI frequently gets port mappings wrong or omits them entirely. If your container exposes port 3000 but you never map it, external traffic can't reach your app.

### Kubernetes at a conceptual level

Kubernetes (K8s) is an orchestrator for containers. It manages groups of containers across multiple machines, handles scaling, networking, and rollouts.

**Pod** — The smallest deployable unit. A pod is one or more containers that share network and storage. Usually one container per pod.

**Deployment** — A declarative spec for how many replicas of a pod should be running. "Keep 3 copies of my app running at all times."

**Service** — A stable network endpoint that load-balances across pods. Your pods might die and restart with new IPs, but the service keeps the same address.

**Ingress** — HTTP routing. Maps external URLs to services inside the cluster.

**kubectl commands:**

```bash
kubectl get pods                    # List pods
kubectl get deployments             # List deployments
kubectl get services                # List services
kubectl describe pod <name>         # Inspect pod details
kubectl logs <pod_name>             # View pod logs
kubectl delete pod <name>           # Delete a pod
kubectl apply -f deployment.yaml    # Apply a manifest
```

Most AI-generated Kubernetes configs will have common patterns. You don't need to be a K8s expert to review them — you need to know what a Pod spec looks like, what a Deployment does, and how Services expose pods.

### Secrets management

Never hardcode secrets. Never put them in Dockerfiles. Never put them in code. They go in environment variables at runtime, injected from a secrets manager.

**Patterns:**

- **Environment variables** — Set at runtime. `docker run -e API_KEY=secret myapp`. In Kubernetes, inject via `env` or `secretKeyRef`.
- **Vault (HashiCorp)** — A dedicated secrets manager. Stores secrets, provides short-lived dynamic credentials, audits access. For serious production workloads.
- **AWS Secrets Manager / GCP Secret Manager / Azure Key Vault** — Cloud-provider-managed secrets. Integrates with their other services.
- **Docker secrets** — For Swarm-based deployments. Secrets are encrypted and only exposed to containers that need them.

The practical minimum: store secrets in environment variables, never in Dockerfiles or source code. If your AI-generated Dockerfile has an `ENV API_KEY=xxx` line, that's a critical failure.

### Environment parity

"Your code works on my machine" is the cardinal sin of deployment. The goal is that dev, staging, and production all behave identically — same OS, same runtime versions, same environment variables, same dependencies.

**Docker solves this** by packaging the environment with the code. The image you build locally is the image you deploy.

**Environment variables** are the standard for configuring apps per-environment. `DATABASE_URL` is different in dev vs. production. `DEBUG=false` in production. These are injected at runtime, not baked into the image.

**The rule:** code should be environment-agnostic. Configuration comes from environment variables. The same image runs everywhere.

### Terraform basics (declarative infra)

Terraform is a tool for describing infrastructure in code. You write HCL (HashiCorp Configuration Language) that declares what resources should exist, and Terraform creates/destroys them to match.

```
# Terraform provider and resource
provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_type = "t3.micro"

  tags = {
    Name = "web-server"
  }
}
```

Terraform files typically have `.tf` extension. You run `terraform init`, `terraform plan` (preview changes), `terraform apply` (make it so). Terraform state tracks what's deployed.

For vibe coding, you mainly need to read and review AI-generated Terraform. You want to know: does this provision a VM? Does it set up a database? Does it expose anything to the internet?

## Hands-on exercise

**Containerize a simple web app and run it locally.**

Time: 15 minutes

Prerequisites: Docker Desktop installed and running

1. Create the project:
```bash
mkdir container-demo && cd container-demo
```

2. Create a simple Node.js app:
```javascript
// app.js
const http = require('http');
const port = process.env.PORT || 3000;

const server = http.createServer((req, res) => {
  res.end(`Hello from container! Host: ${process.env.HOSTNAME || 'unknown'}\n`);
});

server.listen(port, () => {
  console.log(`App listening on port ${port}`);
});
```

3. Create the package.json:
```json
{
  "name": "container-demo",
  "version": "1.0.0",
  "main": "app.js"
}
```

4. Create the Dockerfile:
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json .
RUN npm install --production
COPY app.js .
EXPOSE 3000
ENV PORT=3000
CMD ["node", "app.js"]
```

5. Build and run:
```bash
docker build -t container-demo .
docker run -p 3000:3000 container-demo
```

6. Test it:
```bash
curl http://localhost:3000
```

7. Run a second instance on a different port to verify port mapping:
```bash
docker run -p 3001:3000 container-demo
curl http://localhost:3001
```

8. Inspect running containers:
```bash
docker ps
```

9. View logs:
```bash
docker logs $(docker ps -q --format "{{.ID}}" | head -1)
```

**Expected output:**
- `curl http://localhost:3000` returns "Hello from container!"
- `docker ps` shows a running container with the image `container-demo`
- Two containers can run simultaneously on different ports
- Logs show the startup message "App listening on port 3000"

**Verify the environment isolation:**
```bash
docker run -e HOSTNAME=my-special-container container-demo
curl http://localhost:3000
```

The second container sees `HOSTNAME=my-special-container` in its environment. The first container doesn't. They're isolated.

## Common mistakes

**Mistake 1: Baking secrets into Dockerfiles with ENV or ARG.**

What happens: Secrets persist in the image layer history. If the image is pushed to a public registry or accessible to unauthorized users, every secret is readable with a single command: `docker history` or `docker run IMAGE printenv | grep SECRET`.

Why it happens: It works locally. The app starts, reads the secret, runs. AI generates this pattern because it's the simplest way to make the secret available.

How to fix: Pass secrets at runtime with `-e` flags or via a secrets manager. Never hardcode `ENV API_KEY=xxx` in a Dockerfile. For production, use Docker secrets, Kubernetes secrets, or a vault sidecar. The image should contain code, not configuration.

**Mistake 2: Forgetting to map ports or mapping them incorrectly.**

What happens: The app starts in the container but is unreachable from the host. `curl localhost:3000` hangs or times out. The AI-generated code does `docker run myapp` without `-p 3000:3000` and you spend an hour debugging a networking issue that doesn't exist.

Why it happens: Port mapping feels like an extra step. The container is running — the app should be reachable. This mental model confuses the container's internal network with the host's network.

How to fix: Always verify port mappings. `docker run -p HOST_PORT:CONTAINER_PORT`. If the app inside the container listens on port 3000 and you want to reach it from the host on port 8080, use `-p 8080:3000`. Check `docker port <container>` to see current mappings.

**Mistake 3: Building locally but deploying to a different architecture.**

What happens: `docker build` works on your Mac (amd64). You push to a registry, pull on a ARM-based server (Raspberry Pi, AWS Graviton), and the image fails to start with a cryptic error about an executable format.

Why it happens: Docker images have an architecture. `node:20-alpine` has variants for both amd64 and arm64. If you build on one and deploy to another, it may not work unless the base image supports multi-arch.

How to fix: Use the `--platform` flag when building: `docker build --platform linux/amd64`. Or use multi-arch base images. For AI-generated Dockerfiles, verify the base image supports your target architecture.

**Mistake 4: Running containers with `--privileged` or exposing the Docker socket.**

What happens: A container gains full access to the host system. Any vulnerability in the container can escape to the host. Malware in the container owns the machine.

Why it happens: AI generates configs that need access to Docker-in-Docker for testing, or suggests `--privileged` for hardware access. It works, so it ships.

How to fix: Never run containers with `--privileged`. If you need Docker-in-Docker, use a Docker-in-Docker sidecar (dind) that is properly isolated. Avoid mounting the Docker socket (`-v /var/run/docker.sock`) unless absolutely necessary and understood. Principle of least privilege: containers should only have the capabilities they need.

**Mistake 5: Not setting resource limits on containers or pods.**

What happens: A runaway process consumes all CPU/memory. One container takes down the entire node. In Kubernetes, a pod with no resource limits can exhaust node resources, affecting other pods.

Why it happens: Limits feel like premature optimization. The app works fine without them. Until it doesn't.

How to fix: Always set resource limits. In Docker: `docker run --memory=256m --cpus=0.5`. In Kubernetes, set `resources.requests` and `resources.limits` in the pod spec. Memory limits should match the app's actual needs — measure first.

## AI-specific pitfalls

**AI generates Dockerfiles with secrets baked in via ENV.** This is the most common and most dangerous AI-generated infrastructure mistake. Look for any `ENV` line that sets a variable name like `API_KEY`, `SECRET`, `PASSWORD`, `TOKEN`. These are baked into the image. Reject any Dockerfile with hardcoded secrets. Pass them at runtime with `-e` or use a secrets injection mechanism.

**AI generates Dockerfiles that inherit fat base images.** AI will sometimes generate `FROM ubuntu:latest` or `FROM python:latest` instead of `python:3.11-slim` or `node:20-alpine`. Fat images are hundreds of MBs, slower to pull, and have larger attack surfaces. When reviewing AI-generated Dockerfiles, verify the base image is a slim variant appropriate for the language.

**AI generates Dockerfiles that run as root.** By default, Docker containers run as root unless you specify otherwise. AI won't add a `USER` directive unless prompted. Non-root containers are more secure — if a container is compromised, the attacker has limited host access. Add `RUN addgroup -S appgroup && adduser -S appuser -G appgroup` and `USER appuser` to AI-generated Dockerfiles.

**AI doesn't understand port mapping and omits -p flags.** When AI generates Docker run commands, it frequently omits port mappings. If the generated Dockerfile has `EXPOSE 3000` but the run command doesn't have `-p 3000:3000`, the app won't be reachable from outside the container.

**AI generates Kubernetes manifests with missing resource limits.** Every Kubernetes manifest for a production workload needs `resources.requests` (guaranteed minimum) and `resources.limits` (maximum). AI-generated manifests often omit these. Without limits, pods can be evicted or nodes can be exhausted.

**AI generates Terraform that exposes services directly to the internet.** Terraform for a web service might create an EC2 instance without a security group, or with a security group that allows all traffic (`0.0.0.0/0`). Review AI-generated Terraform carefully for security group rules. Production services should never be wide open.

**AI doesn't understand the difference between environment variables and secrets.** AI will suggest putting a database URL in an env var (fine) and an API key in an env var (also fine) but won't differentiate between them. For secrets, you should be using a secrets manager, not just env vars. When prompting AI about infra, be explicit: "secrets should be injected from AWS Secrets Manager, not hardcoded."

## Quick reference

### IaaS vs PaaS vs FaaS

| Model | You manage | Provider manages | Use when |
|---|---|---|---|
| IaaS | OS, runtime, libraries | Hardware, hypervisor, networking | You need full control |
| PaaS | App code, configuration | Runtime, scaling, servers | You want to deploy without managing infra |
| FaaS | Individual functions | Everything | Event-driven, intermittent workloads |

### Docker command reference

| Command | What it does |
|---|---|
| `docker build -t name .` | Build image from Dockerfile |
| `docker run -p 8080:3000 name` | Run container with port mapping |
| `docker run -e KEY=value name` | Run container with env var |
| `docker run --memory=256m name` | Run with memory limit |
| `docker ps` | List running containers |
| `docker logs name` | View container stdout/stderr |
| `docker exec -it name sh` | Shell into running container |
| `docker stop name` | Stop a container |
| `docker rm name` | Remove a stopped container |
| `docker images` | List local images |

### Kubernetes core concepts

| Concept | What it is |
|---|---|
| Pod | Smallest deployable unit. One or more containers sharing network/storage |
| Deployment | Manages replica count, rolling updates, rollbacks |
| Service | Stable network endpoint exposing pods |
| Ingress | HTTP routing from outside to services inside the cluster |
| ConfigMap | Non-sensitive configuration injected as env vars or files |
| Secret | Sensitive data (keys, tokens) injected as env vars or files |

### Secrets injection patterns

```dockerfile
# BAD — secret baked into image
FROM python:3.11
ENV API_KEY=super-secret-key
CMD ["python", "app.py"]
```

```dockerfile
# GOOD — secret injected at runtime
FROM python:3.11
CMD ["python", "app.py"]
# Run with: docker run -e API_KEY=super-secret-key myapp
```

```yaml
# Kubernetes: inject secret as environment variable
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: app
      env:
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: my-secrets
              key: api-key
```

### Environment parity checklist

- [ ] Same base image in dev and production Dockerfiles
- [ ] Same runtime version locally and in container
- [ ] Environment variables documented and consistent
- [ ] Secrets injected at runtime, not baked into images
- [ ] Port mappings documented and consistent

## Go deeper

- [Docker official getting started guide](https://docs.docker.com/get-started/) (verified 2026-04) — Official Docker tutorial covering images, containers, and basic commands.
- [Kubernetes interactive tutorial](https://kubernetes.io/docs/tutorials/) (verified 2026-04) — Official K8s walkthroughs from the Kubernetes documentation.
- [Terraform getting started](https://developer.hashicorp.com/terraform/tutorials/aws-get-started) (verified 2026-04) — HashiCorp's official Terraform AWS getting started guide.
- [OWASP Docker security cheatsheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html) (verified 2026-04) — Security best practices for container deployments.
- [Kubernetes production best practices (Azure)](https://learn.microsoft.com/en-us/azure/aks/best-practices) (verified 2026-04) — Cloud-agnostic K8s production guidance from Microsoft's AKS team.
