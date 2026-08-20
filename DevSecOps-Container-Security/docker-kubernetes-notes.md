# 🐳 DevSecOps & Container Security Notes

Essential Docker and Kubernetes commands, container isolation concepts, and Dockerfile security hardening guidelines.

---

## 🛠️ Essential Docker Commands

```bash
# Container lifecycle management
docker run -d -p 8080:80 --name app_container app_image
docker ps -a
docker exec -it app_container /bin/bash
docker logs -f app_container

# Image management & cleanup
docker build -t app_image:latest .
docker image prune -a
```

---

## 🔒 Container Security Best Practices

1. **Non-Root User Execution**: Explicitly define `USER 10001` in Dockerfiles to prevent container breakout exploits.
2. **Minimal Base Images**: Use Alpine (`python:3.12-alpine`) or Distroless images to reduce vulnerability attack surface.
3. **Multi-Stage Builds**: Separate build-time tools (compilers, build headers) from production runtime assets.
4. **Vulnerability Scanning**: Scan container images with Trivy before pushing to image registries:
   ```bash
   trivy image app_image:latest
   ```
