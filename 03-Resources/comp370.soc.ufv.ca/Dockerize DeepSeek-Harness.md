You are working on my existing **deepseek-harness** application on the `comp370` Ubuntu VM.

Your task is to thoroughly inspect the existing application and Dockerize it so that it can be deployed as a reliable, isolated service behind the COMP 370 project portal / NGINX reverse proxy.

The application must integrate cleanly with the architecture:

```text
                         INTERNET
                            │
                            ▼
                  comp370.soc.ufv.ca
                            │
                     UFV network/NAT
                            │
                            ▼
                     COMP 370 VM
                            │
                            ▼
                    ┌──────────────┐
                    │    NGINX     │
                    │ Reverse Proxy│
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
          Landing       DeepSeek      Other
           Page         Harness      Projects
           :3000          :3000         :3000
```

The intended public URL for this application is:

```text
https://comp370.soc.ufv.ca/projects/deepseek-harness/
```

The application itself should **not require users to specify a port**.

---

# 1. Inspect the Existing Application First

Before making changes, thoroughly inspect the repository.

Determine:

- application language
    
- framework
    
- entry point
    
- package manager
    
- runtime version
    
- current startup command
    
- current development command
    
- current production command
    
- required environment variables
    
- configuration files
    
- database dependencies
    
- Redis dependencies
    
- external APIs
    
- LLM/model dependencies
    
- Ollama dependencies
    
- filesystem dependencies
    
- mounted directories
    
- generated files
    
- uploaded files
    
- logs
    
- background workers
    
- subprocesses
    
- networking requirements
    
- GPU requirements
    
- current ports
    
- health-check endpoints
    
- WebSocket requirements
    
- frontend/backend architecture
    
- existing Docker files, if any
    

Do not immediately rewrite the application.

First understand how it currently runs.

Identify all commands currently required to run it successfully.

---

# 2. Preserve Existing Functionality

The goal is **containerization**, not an unnecessary rewrite.

Preserve:

- application behavior
    
- existing API
    
- existing UI
    
- existing configuration
    
- existing model integration
    
- existing database behavior
    
- existing authentication
    
- existing project structure
    

Only modify the application where necessary to make it container-friendly and reverse-proxy-compatible.

If changes are required, explain why.

---

# 3. Create a Production Docker Image

Create a production-quality Dockerfile.

Requirements:

- use an appropriate official base image
    
- pin the runtime version appropriately
    
- install only required dependencies
    
- avoid unnecessary packages
    
- use Docker layer caching effectively
    
- do not copy unnecessary files
    
- use `.dockerignore`
    
- do not copy secrets
    
- do not embed `.env` values into the image
    
- run as a non-root user where practical
    
- use an appropriate production startup command
    
- make the container restart-safe
    
- handle SIGTERM correctly
    

The final image should be suitable for:

```bash
docker compose up -d
```

---

# 4. Determine the Correct Internal Port

Inspect the application and determine its actual HTTP listening port.

Prefer an internal application port such as:

```text
3000
```

or:

```text
8000
```

depending on the framework.

Do not expose the port publicly unless necessary.

The preferred production model is:

```yaml
expose:
  - "3000"
```

rather than:

```yaml
ports:
  - "3000:3000"
```

NGINX should communicate with the application through the Docker network.

---

# 5. Docker Compose Integration

Create or update Docker Compose configuration so that `deepseek-harness` can run as an isolated service.

Example architecture:

```yaml
services:

  nginx:
    ...

  landing:
    ...

  deepseek-harness:
    build:
      context: ./deepseek-harness
    expose:
      - "3000"
    restart: unless-stopped
```

Use the actual project structure discovered in the repository.

Do not blindly copy this example.

---

# 6. Internal Docker Network

The service must be connected to the same internal network as NGINX.

For example:

```yaml
networks:
  comp370:
    driver: bridge
```

Then:

```yaml
services:

  nginx:
    networks:
      - comp370

  landing:
    networks:
      - comp370

  deepseek-harness:
    networks:
      - comp370
```

NGINX should be able to reach the service using:

```text
http://deepseek-harness:<internal-port>
```

Do not use:

```text
localhost
127.0.0.1
host.docker.internal
```

for communication between Docker services unless there is a specific reason.

---

# 7. Reverse Proxy Path

The application must work behind:

```text
/projects/deepseek-harness/
```

The public URL should be:

```text
https://comp370.soc.ufv.ca/projects/deepseek-harness/
```

The user should not need:

```text
https://comp370.soc.ufv.ca:3001
```

or:

```text
http://172.30.255.29:3001
```

or any internal Docker address.

---

# 8. Subpath Compatibility

This is a critical requirement.

The application must correctly operate when hosted under:

```text
/projects/deepseek-harness/
```

rather than only:

```text
/
```

Inspect the framework and configure its equivalent of:

- base path
    
- root path
    
- public path
    
- asset prefix
    
- API prefix
    
- URL prefix
    
- forwarded prefix
    

depending on the technology.

Make sure the following work correctly:

```text
/projects/deepseek-harness/
/projects/deepseek-harness/api/...
/projects/deepseek-harness/assets/...
```

No broken:

```text
/static/...
/assets/...
/api/...
```

references.

Do not solve this only with NGINX path rewriting if the application itself needs to understand its base path.

---

# 9. API Routing

If the application has an API, ensure API requests work through the same public prefix.

For example:

```text
https://comp370.soc.ufv.ca/projects/deepseek-harness/api/...
```

should internally become something like:

```text
http://deepseek-harness:3000/api/...
```

Do not expose the API port directly to the Internet.

If the frontend communicates with the backend, make sure it uses relative/proxied URLs rather than hard-coded:

```text
localhost
127.0.0.1
172.x.x.x
```

or arbitrary host ports.

---

# 10. WebSocket / Streaming Support

DeepSeek Harness may potentially use:

- streaming responses
    
- WebSockets
    
- Server-Sent Events
    
- long-running HTTP connections
    

Inspect the application and determine whether these are used.

If applicable, configure NGINX correctly for:

- HTTP/1.1
    
- WebSocket upgrade
    
- connection upgrade
    
- streaming
    
- long read timeouts
    
- buffering behavior
    

For example, where appropriate:

```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

For streaming endpoints, investigate whether:

```nginx
proxy_buffering off;
```

or appropriate timeout settings are required.

Do not blindly apply these settings globally; apply them where needed.

---

# 11. Environment Variables

Inspect all required environment variables.

Create or update:

```text
.env.example
```

Document variables such as:

```text
PORT=
HOST=
OLLAMA_BASE_URL=
DATABASE_URL=
REDIS_URL=
MODEL=
API_KEY=
```

Use the actual variables discovered in the project.

Do not invent variables that the application does not use.

Never commit:

```text
.env
```

containing secrets.

---

# 12. Ollama / LLM Integration

DeepSeek Harness may depend on an LLM runtime such as Ollama.

Determine exactly how the current application connects to Ollama.

Do not assume that:

```text
localhost:11434
```

inside the container means the host's Ollama.

If Ollama runs on the VM host, design the connection explicitly.

If Ollama runs in Docker, prefer Docker service discovery, for example:

```text
http://ollama:11434
```

If Ollama remains on the host, determine the correct networking approach for this VM and Docker environment.

Document the chosen architecture.

The application must not silently fail because `localhost` points to the wrong container.

---

# 13. GPU Requirements

Inspect whether DeepSeek Harness itself requires GPU access.

If it does:

- determine which component actually needs GPU
    
- configure NVIDIA Container Toolkit if required
    
- use the appropriate Docker Compose GPU configuration
    
- do not unnecessarily give GPU access to services that don't need it
    

If the GPU is actually used by Ollama rather than DeepSeek Harness, keep GPU access isolated to the appropriate service.

Do not assume GPU access is necessary just because the application uses an LLM.

---

# 14. Persistent Data

Identify anything that must survive container recreation.

Examples:

```text
data/
models/
uploads/
logs/
cache/
configuration/
generated/
```

Separate:

### Ephemeral container data

from:

### Persistent application data

Use Docker volumes or bind mounts where necessary.

Do not put important persistent data only inside the container filesystem.

Do not mount the entire host filesystem.

Document every volume.

---

# 15. Database / Redis Dependencies

If DeepSeek Harness requires:

- PostgreSQL
    
- pgvector
    
- Redis
    
- Neo4j
    
- Elasticsearch
    
- other services
    

inspect how they are currently provided.

If they already exist elsewhere on the VM, do not automatically create duplicate services.

Determine whether the correct architecture is:

```text
deepseek-harness
       │
       ├── PostgreSQL
       ├── Redis
       └── Ollama
```

or:

```text
deepseek-harness
       │
       ├── existing PostgreSQL
       ├── existing Redis
       └── existing Ollama
```

Use service names rather than hard-coded IP addresses wherever possible.

---

# 16. Health Check

Add an appropriate health endpoint if one already exists.

Prefer something like:

```text
/health
```

or:

```text
/api/health
```

Do not create a fake health check that merely returns `200` without checking whether the application is actually functional.

Use Docker health checks where practical.

Example:

```yaml
healthcheck:
  test:
    - CMD
    - curl
    - -f
    - http://localhost:3000/health
  interval: 30s
  timeout: 5s
  retries: 3
```

Adapt to the actual application.

---

# 17. NGINX Integration

Provide the NGINX configuration required to route:

```text
/projects/deepseek-harness/
```

to the Docker service.

Conceptually:

```nginx
location /projects/deepseek-harness/ {
    proxy_pass http://deepseek-harness:3000/;

    proxy_http_version 1.1;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

However, adapt this carefully to the actual application.

Pay particular attention to trailing slashes and whether the upstream application expects:

```text
/
```

or:

```text
/projects/deepseek-harness/
```

Do not introduce double prefixes.

---

# 18. Landing Page Integration

The landing page should have a project entry for DeepSeek Harness.

Use the landing page's centralized project configuration.

Example:

```typescript
{
  id: "deepseek-harness",
  name: "DeepSeek Harness",
  description: "A software engineering application powered by DeepSeek and related AI tooling.",
  path: "/projects/deepseek-harness/",
  status: "active",
  technologies: [
    "Docker",
    "DeepSeek",
    "Python"
  ]
}
```

Use the actual technologies after inspecting the project.

The landing page button should navigate to:

```text
/projects/deepseek-harness/
```

Never:

```text
localhost:3000
localhost:3001
172.30.255.29
```

and never hard-code a public port.

---

# 19. Service Naming

Use a clear Docker service name:

```text
deepseek-harness
```

The service should be discoverable internally as:

```text
deepseek-harness
```

Avoid arbitrary generated container names.

---

# 20. Logging

Make sure application logs are written to stdout/stderr where practical so that:

```bash
docker compose logs -f deepseek-harness
```

works.

Do not require users to enter the container to find basic application logs.

If persistent logs are required, document why.

---

# 21. Security

The containerized application must:

- not expose unnecessary ports
    
- not expose databases publicly
    
- not expose Redis publicly
    
- not expose Ollama publicly unless explicitly required
    
- not contain secrets in the image
    
- not run privileged unless absolutely necessary
    
- avoid host networking unless required
    
- avoid unnecessary host mounts
    
- run as non-root where practical
    

The preferred production model is:

```text
PUBLIC
──────
80 / 443
    │
    ▼
NGINX

INTERNAL
────────
deepseek-harness:3000
database:5432
redis:6379
ollama:11434
```

---

# 22. Do Not Assume Public 443 Belongs to the VM

This is extremely important for this environment.

The VM currently has:

```text
172.30.255.29
```

while:

```text
comp370.soc.ufv.ca
```

resolves to:

```text
198.162.116.20
```

There is therefore infrastructure between the Internet and this VM.

Do NOT:

- modify DNS
    
- assume the VM owns public `443`
    
- replace existing institutional proxy/NAT configuration
    
- install certificates blindly
    
- change firewall rules unnecessarily
    
- assume that binding NGINX to `443` will make it publicly accessible
    

First document how the existing UFV network forwards traffic to the VM.

The Dockerized application must work within that existing environment.

---

# 23. Development Mode

Preserve a simple local development workflow.

Developers should be able to run the application independently, for example:

```bash
docker compose up deepseek-harness
```

or the appropriate project-specific command.

Also document how to run it without Docker if that is currently part of the development workflow.

Do not make Docker mandatory for every development task unless necessary.

---

# 24. Production Mode

Provide a production deployment configuration.

Prefer:

```bash
docker compose up -d
```

and verify:

```bash
docker compose ps
```

The application should automatically restart after a VM reboot if the overall deployment architecture requires it.

Use:

```yaml
restart: unless-stopped
```

where appropriate.

---

# 25. Testing

Actually test the Dockerized application.

At minimum verify:

### Container

```bash
docker compose ps
```

### Logs

```bash
docker compose logs -f deepseek-harness
```

### Internal HTTP

```bash
curl http://deepseek-harness:<port>/health
```

from the appropriate Docker network/container.

### Public route

Verify:

```text
https://comp370.soc.ufv.ca/projects/deepseek-harness/
```

### API

Verify representative API calls through:

```text
https://comp370.soc.ufv.ca/projects/deepseek-harness/api/...
```

### Assets

Verify JavaScript, CSS, images, fonts, etc.

### Streaming/WebSockets

Verify them if the application uses them.

### Restart

Test:

```bash
docker compose restart deepseek-harness
```

and confirm the application recovers.

### Isolation

Verify the application is NOT unnecessarily accessible through a public port such as:

```text
http://comp370.soc.ufv.ca:3001
```

---

# 26. Landing Page Independence

The landing page must remain functional if DeepSeek Harness is down.

For example:

```text
DeepSeek Harness OFFLINE
```

should not cause:

```text
https://comp370.soc.ufv.ca/
```

to fail.

The landing page should simply show the configured project status.

---

# 27. Documentation

Create/update documentation covering:

## Architecture

```text
Internet
   │
   ▼
comp370.soc.ufv.ca
   │
   ▼
UFV network/NAT
   │
   ▼
NGINX
   │
   ▼
deepseek-harness
```

## Running

```bash
docker compose up -d
```

## Stopping

```bash
docker compose down
```

## Logs

```bash
docker compose logs -f deepseek-harness
```

## Rebuilding

```bash
docker compose build --no-cache deepseek-harness
docker compose up -d deepseek-harness
```

## Health

Document the actual health endpoint.

## Configuration

Document all required environment variables.

## Ollama

Document exactly how DeepSeek Harness connects to Ollama.

## GPU

Document whether GPU access is required and which service receives it.

## Landing Page

Document how the project is registered in the COMP 370 portal.

---

# 28. Final Expected Architecture

The final system should conceptually look like:

```text
                         INTERNET
                            │
                            │ HTTPS
                            ▼
                  comp370.soc.ufv.ca
                            │
                     UFV Network/NAT
                            │
                            ▼
                    COMP 370 VM
                            │
                            ▼
                  ┌─────────────────┐
                  │      NGINX      │
                  │ Reverse Proxy   │
                  └────────┬────────┘
                           │
             ┌─────────────┼─────────────┐
             │             │             │
             ▼             ▼             ▼
          Landing       DeepSeek       Other
           Page         Harness       Projects
           :3000          :3000         :3000
                           │
                           ├─────────────► Ollama
                           │
                           ├─────────────► PostgreSQL
                           │
                           └─────────────► Redis
```

Publicly:

```text
https://comp370.soc.ufv.ca/
```

and:

```text
https://comp370.soc.ufv.ca/projects/deepseek-harness/
```

Internally:

```text
landing:3000
deepseek-harness:3000
ollama:11434
postgres:5432
redis:6379
```

The public user should never need to know these internal ports.

---

# 29. Final Deliverables

When finished, provide:

1. Complete Dockerfile.
    
2. `.dockerignore`.
    
3. Docker Compose changes.
    
4. NGINX configuration required for DeepSeek Harness.
    
5. Landing-page project configuration change.
    
6. `.env.example` updates.
    
7. Health-check configuration.
    
8. Any application configuration required for the `/projects/deepseek-harness/` base path.
    
9. Documentation.
    
10. Exact commands to build and deploy.
    
11. Exact commands to inspect logs and troubleshoot.
    
12. Testing results.
    
13. Any remaining infrastructure dependencies.
    

Before finishing, show the final architecture and explicitly state:

- which ports are public
    
- which ports are internal
    
- how NGINX reaches DeepSeek Harness
    
- how DeepSeek Harness reaches Ollama
    
- how the landing page reaches DeepSeek Harness
    
- how the public URL maps to the Docker service
    

## Most Important Principle

Do not simply "put DeepSeek Harness in Docker."

Make it a **first-class service in the COMP 370 project platform**:

```text
                    COMP 370 PORTAL
                           │
                           ▼
                    /projects/
                           │
                           ▼
                 deepseek-harness
                           │
                 ┌─────────┴─────────┐
                 ▼                   ▼
              Ollama             Other Services
```

It must be isolated, reproducible, internally networked, reverse-proxy compatible, secure, restartable, and accessible through:

```text
https://comp370.soc.ufv.ca/projects/deepseek-harness/
```

without exposing its application port directly to the Internet.