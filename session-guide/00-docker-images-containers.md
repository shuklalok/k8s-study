# Topic 0 - Docker, Images and containers (~15 minutes)

---
## 0.1 - What problems does docker solve?

> **Ref:** https://kubernetes.io/docs/concepts/overview/#going-back-in-time

**SAY THIS**

 Before docker deploying software was painful
 You would have heard "it works on my machine" all the time.
 The app runs fine on a developer's laptop but breaks in staging because of a different OS, missing library, or a different Java version.
 
 Docker solves this by **packaging the application together with everything it needs** - the code, runtime, libraries, OS-level dependencies into a single portable unit called container image. 

 Key points to make 

- A **container image** is a lightweight, stand alone, executable package. 
    - Think of it as a **snapshot** of a file system + a startup command.
    - Images are **immutable** - once built, they never change. You build a new version, you get a new image.
    - Images are stored in a **registry** (Docker hub, GitHub Container Registry, AWS ECR, your private registry).

- A **container** is a running instance of an image.
    - It's an **isolated process** on the host OS - it has its own file system network and process tree.
    - But unlike a VM it shares the **host kernel** - so it starts in milliseconds and uses very little overhead.
    - You can run multiple containers from the same image each is independent Dockerfile is the recipe to build an image.

```Dockerfile
FROM python:3.12-slim       # Start from a base image
WORKDIR /app                # Set the working directory
COPY requirements.txt       # Copy dependency list
RUN pip install --no-cache-dir -r requirements.txt  # Install dependencies
COPY . .                    # Copy application code
EXPOSE 8080                 # Document the port
CMD ["python", "app.py"]    # Default startup command
```

**Analogy:**
> Image = a **class** in OOP. Container = an **instance** of the class.
> Or: Image = a **recipe**. Container = the **dish** you cook from it.

---

## 0.2 - Docker vs Virtual Machines - why containers?

# Insert the image below 
```text
      +---------------------------------+---------------------------------+
      |        VIRTUAL MACHINES         |            CONTAINERS           |
      +---------------------------------+---------------------------------+
      |    +---------+    +---------+   |    +---------+    +---------+   |
      |    |  App A  |    |  App B  |   |    |  App A  |    |  App B  |   |
      |    |---------|    |---------|   |    |---------|    |---------|   |
      |    | Bins/   |    | Bins/   |   |    | Bins/   |    | Bins/   |   |
      |    |  Libs   |    |  Libs   |   |    |  Libs   |    |  Libs   |   |
      |    +---------+    +---------+   |    +---------+    +---------+   |
      |    | Guest OS|    | Guest OS|   |                                 |
      |    +---------+    +---------+   |    +-------------------------+  |
      |            Hypervisor           |    | Container Runtime/Engine|  |
      +---------------------------------+    |   (Docker/Containerd)   |  |
      |             Host OS             |    +-------------------------+  |
      |                                 |          Host OS Kernel         |
      +---------------------------------+---------------------------------+
      |                  Infrastructure (Server)                          |
      +-------------------------------------------------------------------+
```

> **Ref:** [Going back in time](https://kubernetes.io/docs/concepts/overview/#going-back-in-time)

**Key differences to highlight:**
- **Size:** VM are in GBs (full OS). Container images are in MBs (just app + deps).
- **Startup:** VMs take minutes. Containers start in milliseconds.
- **Isolation:** VMS have stronger isolation (separate kernel). Containers share the host kernel.
- **Density:** You can run 100s of containers on a single host versus 10s of VMS.

## 0.3 - key docker commands (Quick Reference)
```bash
# Build an image from Dockerfile
docker build -t my-app:1.0 .
# List local images
docker images
# Run a container
docker run -d -p 8080:8080 --name my-running-app my-app:1.0
# List running containers
docker ps
# View logs 
docker logs my-running-app
# Execute a command inside a running container 
docker exec -it my-running-app /bin/sh
# Stop and remove
docker stop my-running-app
docker rm my-running-app
# Push to a registry
docker push your-registry/my-app:1.0
``` 

## 0.4 - So... Why Do We Need Kubernetes?

>**Ref:** https://kubernetes.io/docs/concepts/overview/

**SAY THIS:**

Docker is great for running containers on **one machine**.
But in production, you have questions Docker alone can't answer:

- **Scheduling:** I need five copies of this container across three servers. Who decides which server it runs on?
- **Self-healing:** A container crashed at 3:00 AM. Who restarts it?
- **Rolling Updates:** I'm pushing a new version. How do I update without downtime?
- **Service Discovery:** Service A needs to talk to service B. How do they find each other?
- **Scaling:** Black Friday traffic is spiking! How do I auto-scale my application?
- **Secrets Management:** This container needs a database password. How do I pass it securely?

This is exactly what **Kubernetes** does. It is the **orchestration** that sits on top of the container runtime and manages the lifecycle, networking, scaling, and configuration of containers across a cluster of machines.

```text
 Docker = runs containers on ONE machine
 Kubernetes = orchestrate containers across MANY machines
```

> **Fun Fact:** Google runs **billions** of container per week internally (using Borg - the k8s' predecessor). They donated K8s to the Cloud Native Computing Foundation (CNCF) in 2015. Today, every major cloud provider offers managed K8s EKS(AWS), GKE(Google), AKS(Azure).

> **Fun Fact:** The name "Kubernetes" come from Greek **κυβερνήτης** (kubernḗtēs)meaning "helmsman". Google originally code named it **Project Seven** (of 9) - a Star Trek Borg reference. The logo's 7-spoke wheel is a nod to the origin.

> **Special Note:** For today's demos, we are using **k3s** - Rancher's lightweight k3s distro. Same certified Kubernetes API, but a single ~70MB binary. The name is a joke. k3s = k(8 - 5)s - "5 less than k8s. Under the hood it uses **containerd** as the container runtime (not Docker), which is the same runtime used by EKS, GKE and most production clusters.

### Q&A Bullets - Topic 0

- **Q: Do I need docker installed to use Kubernetes?**
  - No. k8s uses a **container runtime** (containerd, CRI-O), not docker directly. Docker is useful for **building** images, but docker doesn't use `docker run` under the hood. Since k8s 1.24, dockershim was removed entirely.

- **Q: What is the difference between a container and a Pod?**
  - A container is a single runtime process a pod is a creators wrapper that can hold one or more containers sharing the same network and storage we will cover this in topic 1.
 
- **Q: Is Docker Compose of same as Kubernetes?**
  - No. Docker Compose is for local multi-container setups on a single host `docker-compose.yaml`. K8s is a production grade orchestration across a cluster. Docker Compose is great for local dev. K8s is for deployment.

- **Q: What is a Container Registry?**
  - A server that stores container images. Public: Docker Hub, ghcr.io. Private: AWS ECR, Google Artifact Registry, Harbor. You `docker push` to upload and k8s `pulls` image from the registry when creating Pods.

- **Q: Why Containerd instead of docker?**
  - Docker was a monolith (CLI + demon + build + runtime). K8s only needs the runtime part. containerd is that runtime, extracted and optimized. Docker Desktop still uses containerd under the hood.

- **Q: Can I use images built with Docker in Kubernetes?**
  - Yes. Docker images are OCI-compliant container images. K8S doesn't care how the image was built (Docker, Buildah, Kaniko etc.) - it only needs a valid OCI image in a registry. Build with whatever you like, deploy with k8s.

- **Q: What are the difference between `CMD` and `ENTRYPOINT`?**
  - `ENTRYPOINT` sets the main executable. `CMD` provides default arguments. In k8s YAML, `command` overrides `ENTRYPOINT` and `ARGS` overrides `CMD`. If you set `command` in your Pod spec, the images `ENTRYPOINT` is completely ignored.

- **Q: What is a multi-stage Docker build?**
  - Multiple `FROM` statements. You use one stage to build/combine (with heavy tools like JDK, gcc, npm) and copy only the final artifact into a minimal content image. Result: Production images are much smaller (e.g. 800 MB build stage → 50MB runtimes). Example `FROM goal and 1.22 AS builder` → `FROM alpine:3.20` → `COPY --from=builder /app/binary /app/binary`. 

- **Q: Should I use latest as image tag?**
  - **No.** `lates` is mutable - it points to whatever was pushed last. If someone pushes a new version, your Pods might pull a different image on restart. Always use **specific, immutable tags** (e.g., `nginx:1.27.0`, `my-app:v2.3.1`) or better yet, pin by **digest** (`nginx@sha256:abc123...`). This ensures reproducible deployments.

- **Q: What is `imagePullPolicy` in K8s and when does it matter?**
  -  Controls when kubelet pulls the image: `Always` (pull every time), `IfNotPresent` (pull only if not cached), `Never` (only use cashed). Default depends on the tag: `:latest` defaults to `Always`, any other tag defaults to `IfNotPresent`. In production, use specific tags + `IfNotPresent` to avoid unnecessary pulls and registry rate limits.
  
---

**HOME** [Session Guide Index](README.md) | **NEXT:** [Topic 1 - Basic Concepts of K8s](01-basic-concepts.md)