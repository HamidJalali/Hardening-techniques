# Container Hardening Techniques

Practical guidance for reducing the attack surface of Docker and Podman images and containers.

## 1. Use a trusted, minimal base image

- Prefer official, maintained images from trusted publishers.
- Use slim, distroless, or `scratch` images when practical.
- Pin production images to a version and, where reproducibility matters, to a digest rather than using `latest`.
- Rebuild regularly so security fixes in the base image are included.

```dockerfile
FROM python:3.12-slim
```

A digest can be used for stronger reproducibility:

```dockerfile
FROM python:3.12-slim@sha256:<digest>
```

## 2. Use multi-stage builds

Keep compilers, build tools, test dependencies, and package managers out of the runtime image.

```dockerfile
FROM python:3.12-slim AS builder

WORKDIR /build
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt
COPY . .

FROM gcr.io/distroless/python3-debian12:nonroot
WORKDIR /app
COPY --from=builder /install /usr/local
COPY --from=builder /build /app
USER nonroot:nonroot
CMD ["app.py"]
```

## 3. Remove unnecessary packages and files

- Do not install shells, editors, debugging tools, or package managers unless required at runtime.
- Clean package-manager caches.
- Remove temporary files, tests, source maps, and build artifacts that are not needed in production.
- Keep the runtime image as small as possible.

## 4. Run as a non-root user

Define and use a dedicated unprivileged user in the image or specify one at runtime:

```dockerfile
USER 10001:10001
```

```bash
docker run --user 10001:10001 myapp:1.0
```

## 5. Keep secrets out of images

Never bake passwords, API keys, private keys, certificates, or `.env` files into image layers. Use a secret manager, secret mount, or an appropriate orchestrator mechanism at runtime.

Use a `.dockerignore` file:

```dockerignore
.git
.env
.env.*
*.key
*.pem
__pycache__
.pytest_cache
```

## 6. Restrict the runtime

Example Docker command with several defensive controls:

```bash
docker run -d \
  --name myapp \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --cap-drop=ALL \
  --security-opt=no-new-privileges:true \
  --pids-limit=100 \
  --memory=512m \
  --cpus=1 \
  --user 10001:10001 \
  -p 8080:8080 \
  myapp:1.0
```

Podman supports equivalent options:

```bash
podman run -d \
  --name myapp \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --cap-drop=ALL \
  --security-opt=no-new-privileges:true \
  --pids-limit=100 \
  --memory=512m \
  --cpus=1 \
  --user 10001:10001 \
  -p 8080:8080 \
  myapp:1.0
```

### Read-only root filesystem

Use `--read-only` and provide writable `tmpfs` mounts or named volumes only for paths that genuinely require writes. Test the application first because some applications expect writable temporary or cache directories.

### Linux capabilities

Start with all capabilities dropped:

```bash
--cap-drop=ALL
```

Add back only a specifically required capability, for example:

```bash
--cap-add=<required-capability>
```

### No new privileges

Prevent privilege escalation by using:

```bash
--security-opt=no-new-privileges:true
```

### Resource limits

Use memory, CPU, process, and file-descriptor limits appropriate to the workload. This reduces denial-of-service impact but should be validated against normal application behavior.

## 7. Use security profiles

Use the default seccomp profile unless the application requires a documented exception. On systems that support them, use AppArmor or SELinux policies to restrict system calls, files, devices, and capabilities.

Avoid `--privileged` unless there is a documented and unavoidable requirement. Prefer narrowly scoped devices, capabilities, and mounts.

## 8. Limit filesystem and network exposure

- Mount only required host paths and use read-only bind mounts where possible.
- Avoid mounting the Docker or Podman socket into application containers.
- Expose only required ports.
- Avoid host networking unless it is necessary.
- Use explicit network policies and isolate services that do not need to communicate.
- Restrict writable paths and avoid world-writable directories.

## 9. Scan, sign, and verify images

Scan images during development and in CI/CD. Examples include Trivy, Grype, Snyk, Anchore, and Docker Scout:

```bash
docker build --pull -t myapp:1.0 .
trivy image --severity HIGH,CRITICAL myapp:1.0
```

Consider signing images with tools such as Cosign and verifying signatures and provenance before deployment. Fail the pipeline according to an agreed vulnerability policy rather than blindly ignoring all findings.

## 10. Pin and update dependencies

- Pin application dependencies where practical.
- Generate and review a software bill of materials (SBOM).
- Rebuild images regularly, even when application code has not changed.
- Monitor base-image and dependency advisories.
- Remove or replace packages that are no longer needed.

## 11. Volumes and container lifecycle

Containers created by Docker are normally managed by Docker; containers created by Podman are managed by Podman. The two engines use separate metadata and usually separate volume locations.

A volume mount is fixed when a container is created. To change mounts, recreate the container:

```bash
docker stop <container>
docker rm <container>
docker run -d \
  --name <container> \
  -v <volume_name>:/path/inside/container \
  <image_name>
```

Inspect volume usage before deleting data:

```bash
docker system df -v
docker volume inspect <volume_name>
docker volume rm <volume_name>
```

Do not assume a Docker volume can be used directly by Podman, especially with rootless containers. Export and import data explicitly when migrating between engines.

## 12. Validation checklist

Before releasing an image, verify that:

- The base image is trusted, current, and pinned appropriately.
- The image contains no credentials or unnecessary tools.
- The application runs as a non-root user.
- The root filesystem can be read-only.
- All unnecessary capabilities are dropped.
- `no-new-privileges` is enabled.
- Seccomp, AppArmor, or SELinux controls are applied where available.
- Ports, devices, host paths, and networks are limited.
- CPU, memory, process, and storage limits are configured.
- Vulnerability and secret scans pass the organization’s policy.
- The image is signed and provenance is verified where required.
- Volumes are backed up and deletion procedures are understood.

## Important caveats

- Distroless and `scratch` images usually do not include a shell, so `docker exec -it <container> sh` may not work.
- `docker volume prune` and `podman volume prune` can permanently delete unused data; review the list before confirming.
- Security controls must be tested with the application. A control that prevents the service from starting should be tuned narrowly rather than removed without investigation.
