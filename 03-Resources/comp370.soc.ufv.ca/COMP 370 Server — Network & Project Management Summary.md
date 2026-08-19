
Based on the tests we performed, this is our current understanding of the `comp370.soc.ufv.ca` environment.

### 1. Network topology

The VM is **not directly assigned the public IP** for the website.

The VM's network interface is:

```text
ens18
  IP:      172.30.255.29/16
  Gateway: 172.30.0.1
```

The domain resolves to:

```text
comp370.soc.ufv.ca
        ↓
198.162.116.20
```

The VM reports a different public/egress IP:

```text
198.162.104.18
```

Therefore, there is an **external UFV network/NAT/proxy layer** between the Internet and the VM.

Conceptually:

```text
                    INTERNET
                        │
                        │ HTTPS
                        ▼
              comp370.soc.ufv.ca
                 198.162.116.20
                        │
                        │
                UFV network /
                NAT / gateway
                        │
                        ▼
               ┌────────────────┐
               │   COMP 370 VM  │
               │ 172.30.255.29  │
               └────────────────┘
```

---

## 2. HTTPS is handled upstream

We confirmed that:

```bash
curl https://comp370.soc.ufv.ca/
```

reaches:

```text
198.162.116.20:443
```

and successfully completes a TLS handshake.

The endpoint presents a Let's Encrypt certificate for:

```text
comp370.soc.ufv.ca
```

Therefore:

```text
Internet
   │
   ▼
198.162.116.20:443
   │
   ▼
TLS endpoint
```

exists somewhere upstream.

However, on the VM itself:

```bash
sudo ss -tulpn | grep -E ':80|:443'
```

showed **nothing listening on 80 or 443**.

NGINX is installed but currently:

```text
inactive (dead)
disabled
```

So we should **not assume that the VM currently owns public HTTPS**.

---

# 3. Important architectural principle

The VM should be treated as an **application server behind the UFV network**, rather than as a traditional Internet-facing server with its own public IP.

That means:

```text
PUBLIC INTERNET
      │
      ▼
UFV infrastructure
      │
      ▼
COMP 370 VM
      │
      ├── NGINX
      ├── Landing Page
      ├── Project A
      ├── Project B
      ├── Project C
      └── supporting services
```

The exact forwarding rule from the UFV infrastructure to the VM still needs to be documented/confirmed.

---

# 4. Recommended project architecture

The VM should operate as a **COMP 370 Project Platform**.

Instead of giving every project a separate public port:

```text
❌ comp370.soc.ufv.ca:3001
❌ comp370.soc.ufv.ca:3002
❌ comp370.soc.ufv.ca:3003
```

use a single public entry point:

```text
https://comp370.soc.ufv.ca/
```

with path-based routing:

```text
https://comp370.soc.ufv.ca/projects/project-a/
https://comp370.soc.ufv.ca/projects/project-b/
https://comp370.soc.ufv.ca/projects/deepseek-harness/
```

Conceptually:

```text
                         INTERNET
                            │
                            ▼
                  comp370.soc.ufv.ca
                            │
                      UFV Gateway
                            │
                            ▼
                    ┌──────────────┐
                    │    NGINX     │
                    │ Reverse Proxy│
                    └──────┬───────┘
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
          Landing       Project A     Project B
           Page
```

---

# 5. Use Docker for projects

Each project should ideally be an independent Docker service.

For example:

```text
COMP 370 VM
│
├── nginx
│
├── landing
│
├── deepseek-harness
│
├── project-a
│
├── project-b
│
├── postgres
│
├── redis
│
└── ollama
```

Projects should communicate through Docker's internal network.

For example:

```text
NGINX
  │
  ├── http://landing:3000
  │
  ├── http://deepseek-harness:3000
  │
  └── http://project-a:3000
```

The applications don't need unique host ports.

---

# 6. Avoid exposing application ports

The preferred model is:

```yaml
expose:
  - "3000"
```

rather than:

```yaml
ports:
  - "3001:3000"
```

This means the application is reachable by other Docker services but isn't unnecessarily exposed through the VM's network interface.

For example:

```text
                 NGINX
                   │
                   │ Docker network
                   ▼
          deepseek-harness:3000
```

rather than:

```text
                 Internet
                    │
                    ▼
             VM:3001
                    │
                    ▼
          deepseek-harness
```

---

# 7. Landing page = project portal

The landing page should be the central catalog for all projects.

For example:

```text
https://comp370.soc.ufv.ca/

COMP 370 PROJECTS

┌───────────────────────────────┐
│ DeepSeek Harness              │
│ AI / Software Engineering     │
│                               │
│ [Open Project]                │
└───────────────────────────────┘

┌───────────────────────────────┐
│ Project A                     │
│                               │
│ [Open Project]                │
└───────────────────────────────┘
```

Each button should point to:

```text
/projects/<project-name>/
```

The landing page should **not contain knowledge of Docker IP addresses or internal ports**.

---

# 8. NGINX = traffic router

NGINX should be responsible for translating public URLs into internal services.

For example:

```text
/projects/deepseek-harness/
             │
             ▼
      deepseek-harness:3000
```

and:

```text
/projects/project-a/
             │
             ▼
         project-a:3000
```

This gives us a clean separation:

|Component|Responsibility|
|---|---|
|UFV network|Public network/DNS/edge routing|
|NGINX|HTTP reverse proxy|
|Landing page|Project discovery|
|Docker|Application isolation|
|Project containers|Application logic|
|PostgreSQL/Redis/etc.|Supporting infrastructure|

---

# 9. Project structure

A clean organization could be:

```text
/srv/comp370/
│
├── infrastructure/
│   ├── nginx/
│   └── docker-compose.yml
│
├── landing/
│
├── projects/
│   ├── deepseek-harness/
│   │   ├── Dockerfile
│   │   ├── docker-compose.yml
│   │   └── ...
│   │
│   ├── project-a/
│   │   ├── Dockerfile
│   │   └── ...
│   │
│   └── project-b/
│       ├── Dockerfile
│       └── ...
│
└── data/
    ├── postgres/
    ├── redis/
    └── ...
```

The exact directory structure can differ, but the principle should remain:

**infrastructure is separated from individual projects.**

---

# 10. Each project should be independently deployable

For example, updating DeepSeek Harness should not require rebuilding the landing page.

You should ideally be able to do:

```bash
docker compose build deepseek-harness
docker compose up -d deepseek-harness
```

while:

```text
Landing Page
Project A
Project B
```

remain running.

Similarly, removing Project A should not affect DeepSeek Harness.

---

# 11. Projects should have a standard contract

Every project should ideally provide:

```text
Dockerfile
.env.example
README.md
health endpoint
Docker Compose configuration
```

and document:

```text
Internal port
Environment variables
Dependencies
Volumes
Health endpoint
GPU requirement
Database requirement
Ollama requirement
Public URL
```

For example:

```text
Project: DeepSeek Harness

Internal service:
deepseek-harness:3000

Public URL:
/projects/deepseek-harness/

Health:
/health

Dependencies:
Ollama
PostgreSQL
Redis
```

---

# 12. Security model

The desired security boundary is:

```text
PUBLIC
──────
comp370.soc.ufv.ca
        │
        ▼
      NGINX
        │
        ▼
INTERNAL DOCKER NETWORK
        │
 ┌──────┼──────────┐
 ▼      ▼          ▼
App    DB         Redis
```

Databases, Redis, Ollama, and application services should **not be unnecessarily exposed publicly**.

Only the public web entry point should need to be accessible externally.

---

# 13. Current state vs desired state

### Current state

```text
Domain
  │
  ▼
198.162.116.20
  │
  ▼
UFV network infrastructure
  │
  ▼
VM 172.30.255.29

NGINX: OFF
Docker containers: currently none
80: no listener
443: no listener
UFW: inactive
```

### Desired state

```text
Domain
  │
  ▼
UFV network infrastructure
  │
  ▼
COMP 370 VM
  │
  ▼
NGINX
  │
  ├── /
  │    └── Landing Page
  │
  ├── /projects/deepseek-harness/
  │    └── DeepSeek Harness
  │
  ├── /projects/project-a/
  │    └── Project A
  │
  └── /projects/project-b/
       └── Project B
```

---

## 14. One remaining unknown

There is still one piece we should **not assume**:

> Exactly how does the UFV infrastructure map `198.162.116.20` to `172.30.255.29` and which VM port does it forward to?

We know TLS reaches `198.162.116.20:443`, but we haven't yet established the complete forwarding path to the VM.

That should be documented before changing the production network configuration.

---

# Recommended operating model

Going forward, I would treat `comp370` as a small **containerized application platform**:

```text
                 comp370.soc.ufv.ca
                         │
                         ▼
                  UFV network edge
                         │
                         ▼
                       NGINX
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
       Landing       DeepSeek       Other Projects
                       Harness
          │              │
          └──────────────┴──────────────┐
                                         │
                                  Docker network
                                         │
                             ┌───────────┼──────────┐
                             ▼           ▼          ▼
                          Ollama      PostgreSQL   Redis
```

**The key design principle is: one public domain, one project portal, path-based routing, isolated Docker services, and no need for users to know or access individual application ports.**

This also makes adding a new COMP 370 project straightforward: **containerize it → attach it to the project network → add one NGINX route → register it on the landing page.**