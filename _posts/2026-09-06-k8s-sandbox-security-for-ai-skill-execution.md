---
layout: default
title: "Sandboxing AI Skills in Kubernetes: How We Let Users Run Arbitrary Code Without Losing Sleep"
date: 2026-09-06
categories: [AI Architecture, Kubernetes, Security, SkillHub]
---

# Sandboxing AI Skills in Kubernetes: How We Let Users Run Arbitrary Code Without Losing Sleep

*September 6, 2026 · 24 min read*

**Key Takeaways**

- An AI skill marketplace that lets users publish and execute arbitrary code packages needs defense in depth: pre-publish scanning, hardened ephemeral containers, and network-level isolation — each layer catches what the others miss.
- Kubernetes Jobs provide the right execution primitive for AI skill sandboxing: one-shot, ephemeral, automatically cleaned up, with security contexts that make privilege escalation physically impossible.
- The hardest production problem isn't the security model itself — it's the cold-start latency of large container images (1.5GB office runtime) and the surprisingly subtle bugs that occur when your prepuller DaemonSet disagrees with your image resolution logic about which tag to pull.

---

Our enterprise AI workbench includes a skill marketplace. Users — internal analysts, operations teams, finance staff — can publish code packages that the AI agent invokes during conversations. A user asks "convert this contract to PDF," and the agent calls a skill that spins up a container with LibreOffice, processes the document, and returns the result.

This is the feature that makes the platform useful. It's also the feature that keeps the security team awake at night.

The fundamental tension is architectural: you *want* users to run their code — that's the product. You *cannot* trust their code — that's security. Every other design decision flows from this contradiction.

Over six months in production, serving ~300 users on-premise under China's PIPL regulations, we built a three-layer security model that resolves this tension. This post describes the architecture, the trade-offs, and the production incidents that shaped the design.

![Figure 1. Three-Layer Sandbox Security Model](/assets/images/fig24-sandbox-security-layers.svg)
*Figure 1. Every skill passes through three independent security layers. Each layer operates on a different principle: Layer 1 scans the code before it enters the system, Layer 2 constrains the execution environment, Layer 3 restricts what the running code can reach over the network.*

---

## The threat model

Before describing the solution, we need to be precise about what we're defending against. Our skill marketplace has two classes of publishers:

**Internal skills** are authored by the platform team. They've been code-reviewed, tested, and deployed through CI/CD. Examples: document conversion (LibreOffice-based), data export, report generation. These skills may need to call external services (a document standardization API, for instance).

**External skills** are authored by users. They haven't been reviewed by the platform team. They could be anything from a helpful data processing script to — accidentally or intentionally — a network scanner, a cryptocurrency miner, or a data exfiltration tool.

The threat model is straightforward:

| Threat | Impact | Layer that stops it |
|--------|--------|-------------------|
| Known malware patterns | System compromise | Layer 1 (YARA scan) |
| Zip-slip or path traversal | File system escape | Layer 1 (package policy) |
| Privilege escalation | Container escape | Layer 2 (K8s security context) |
| Resource exhaustion (CPU/memory) | Denial of service | Layer 2 (resource limits) |
| Runaway execution time | Resource waste | Layer 2 (activeDeadlineSeconds) |
| Data exfiltration via network | Data breach | Layer 3 (NetworkPolicy) |
| Lateral movement within cluster | Wider compromise | Layer 3 (namespace isolation) |

No single layer addresses all threats. That's why there are three.

---

## Layer 1: Pre-publish scanning

Every skill package is scanned before it enters the marketplace. The scanning pipeline has two paths: a remote scanning microservice (primary) and local YARA-X rules (fallback).

### The remote scanner

The primary scanner is an HTTP microservice (based on cisco's skill-scanner) that accepts a `.zip` upload and returns a structured finding list:

```python
async def _scan_remote(url: str, zip_bytes: bytes) -> ScanResult:
    files = {"file": ("package.zip", zip_bytes, "application/zip")}
    data = {"use_behavioral": "true", "use_llm": "false"}
    async with httpx.AsyncClient(timeout=settings.skill_scan_timeout) as client:
        resp = await client.post(f"{url}/scan-upload", files=files, data=data)
        resp.raise_for_status()
        return _map_remote(resp.json())
```

Two configuration choices are worth noting:

1. **Behavioral analysis is enabled.** The scanner examines data flows within the package, not just static patterns. This catches obfuscated malware that YARA rules miss.
2. **LLM analysis is disabled.** Using an LLM to analyze code for security issues sounds appealing, but in practice it introduces non-determinism into the security pipeline. A skill that passes scanning on Monday might fail on Tuesday because the LLM changed its assessment. For a security gate, determinism matters more than intelligence.

### The YARA-X fallback

If the remote scanner is unavailable (network issue, service restart, timeout), we fall back to local YARA-X rules:

```python
async def scan(zip_bytes: bytes) -> ScanResult:
    if not settings.skill_scan_enabled:
        return ScanResult(scanned=False)
    
    url = (settings.skill_scanner_url or "").strip()
    if url:
        try:
            return await _scan_remote(url, zip_bytes)
        except Exception as exc:
            logger.warning("Remote scanner unavailable, falling back to local YARA-X: %s", exc)
    
    return await run_in_threadpool(package_scanner.scan_package, zip_bytes)
```

The design principle: **the publishing pipeline must never be blocked by a scanner outage.** If both scanning paths fail, the skill is marked as `scanned=False` — it can be published but the platform logs a warning and the admin review queue flags it.

### Package policy validation

Before scanning even begins, a structural validation layer checks:

- **Magic bytes**: Is this actually a zip file? (Prevents disguised executables)
- **Size limits**: Reject packages over the configured maximum
- **Zip-slip prevention**: No path traversal in filenames (`../../etc/passwd`)
- **File whitelist**: Only expected file types (`.py`, `.js`, `.md`, `.json`, etc.)
- **UTF-8 validation**: Source files must be valid UTF-8 (prevents encoding-based attacks)

These are deterministic, zero-cost checks that catch the most basic attacks before any heavyweight scanning begins.

### Trade-off: scan mode

The `skill_scan_mode` setting controls what happens when scanning finds issues:

- `"block"`: Findings that exceed the severity threshold → hard reject. The skill cannot be published.
- `"warn"`: Findings are logged and visible to admin reviewers, but publishing proceeds.
- `"off"`: Scanning is disabled entirely.

In production, we run `"warn"` for internal skills (platform team code shouldn't trigger YARA rules, but if it does, we want to know) and `"block"` for external skills.

---

## Layer 2: Kubernetes ephemeral Job execution

This is the core of the sandbox architecture. Every skill execution creates a fresh Kubernetes Job with a single Pod that runs the skill code, collects the output, and is destroyed.

### The execution primitive: K8s Jobs

We chose Kubernetes Jobs over three alternatives:

| Alternative | Why we rejected it |
|---|---|
| Long-running Pods (persistent sandbox) | Shared state between runs. One skill's side effects can affect the next. Security isolation requires fresh environments. |
| Docker-in-Docker | Requires privileged mode in the host container — defeats the purpose of sandboxing. |
| In-process execution (subprocess) | No resource limits, no network isolation, process crash takes down the entire platform. |
| WebAssembly (Wasm) | Promising for CPU-bound computation, but our skills need system tools (LibreOffice, curl, git) that Wasm can't provide. |

The K8s Job model, adapted from [kubiyabot/skill](https://github.com/kubiyabot/skill)'s Docker runtime (MIT), gives us exactly the properties we need: ephemeral (fresh per run), resource-bounded, network-isolated, and automatically cleaned up.

### The hardened security context

Every skill Pod runs with a security context that makes privilege escalation physically impossible:

```python
security_context = V1SecurityContext(
    run_as_non_root=True,
    run_as_user=1000,
    allow_privilege_escalation=False,
    read_only_root_filesystem=True,
    capabilities=V1Capabilities(drop=["ALL"]),
)
```

Let's break down what each line does:

| Setting | Effect | What it prevents |
|---------|--------|-----------------|
| `runAsNonRoot: true` | K8s rejects the Pod if the container tries to run as root | Container escape via root privileges |
| `runAsUser: 1000` | Forces a specific non-root UID | UID 0 escalation |
| `allowPrivilegeEscalation: false` | No `setuid` binaries, no `sudo` | SUID-based privilege escalation |
| `readOnlyRootFilesystem: true` | Container filesystem is immutable | Writing malware to disk, modifying system binaries |
| `capabilities: drop ALL` | Removes every Linux capability | Network manipulation, raw socket access, kernel exploits |

With `readOnlyRootFilesystem: true`, the skill code can only write to explicitly mounted `emptyDir` volumes: `/workspace` (outputs) and `/tmp` (scratch). These are ephemeral — they vanish when the Pod terminates.

### Resource limits and timeouts

```python
resources = V1ResourceRequirements(
    requests={"cpu": settings.skill_cpu_request, "memory": settings.skill_memory_request},
    limits={"cpu": settings.skill_cpu_limit, "memory": settings.skill_memory_limit},
)
```

If a skill exceeds its memory limit, the kernel OOM-kills the container. If it exceeds `activeDeadlineSeconds`, Kubernetes terminates the Job. Both of these are kernel-level enforcement — no amount of clever code in the skill can circumvent them.

### Credential isolation: presigned URLs

The workspace engine uses MinIO (S3-compatible object storage) for skill package download and output upload. But **no MinIO credentials enter the sandbox**. Instead:

1. The platform generates time-limited presigned URLs for the specific objects the skill needs.
2. The `initContainer` downloads the skill package using a presigned GET URL.
3. After execution, a sync step uploads outputs using a presigned PUT URL.

If a presigned URL leaks, the blast radius is limited: it expires after a configured TTL and grants access to exactly one object, not the entire storage system.

### Fallback: local subprocess

When K8s is unavailable (local development, single-machine deployment), the runtime falls back to `LocalSubprocessRuntime`:

```python
try:
    batch_api, core_api = k8s_client._apis()
except K8sUnavailable:
    logger.warning("K8s unavailable, falling back to local subprocess")
    return await _run_local(package_bytes, meta, inputs)
```

The local runtime runs skills as subprocesses with reduced privileges. It provides no network isolation and limited resource control — it exists solely for development convenience and is never used in production.

---

## Layer 3: NetworkPolicy isolation

The third layer controls what the running skill can reach over the network. This is where the trust boundary between internal and external skills is enforced.

![Figure 3. NetworkPolicy: External vs Internal](/assets/images/fig26-network-isolation.svg)
*Figure 3. External skills (user-authored) can only reach DNS and MinIO — no internet. Internal skills (platform-authored) get full internet access. The boundary is enforced by Kubernetes NetworkPolicy at the CNI level.*

### The two-tier egress model

We implement two NetworkPolicies in the `skillhub-run` namespace:

**Baseline policy** (applies to ALL skill Jobs):

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: skill-job-egress-baseline
  namespace: skillhub-run
spec:
  podSelector:
    matchExpressions:
      - {key: app, operator: In, values: [skillhub-runner, skillhub-workspace]}
  policyTypes: [Egress]
  egress:
    - ports:
        - {protocol: UDP, port: 53}   # DNS
        - {protocol: TCP, port: 53}
    - to:
        - ipBlock: {cidr: 172.16.2.191/32}  # MinIO
```

**Internal-only policy** (applies only to Jobs with the egress-allow label):

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: skill-job-egress-internal
  namespace: skillhub-run
spec:
  podSelector:
    matchLabels:
      skillhub.klwk/egress: allow
  policyTypes: [Egress]
  egress:
    - {}  # Allow all egress
```

The `runtime_service` stamps the egress-allow label on Jobs whose skill manifest has `origin: internal`:

```python
if manifest.origin == "internal":
    job_labels["skillhub.klwk/egress"] = "allow"
```

Because NetworkPolicy egress rules are additive, internal skills get: DNS + MinIO (from baseline) + full internet (from internal policy). External skills get only: DNS + MinIO.

### Why not a firewall?

NetworkPolicy operates at the Kubernetes CNI level (Calico or Cilium). It's enforced per-Pod, integrated with the K8s scheduler, and requires no additional infrastructure. A traditional firewall would need to understand Pod IPs (which change every run) and would be a single point of failure external to the cluster.

### The blast radius warning

The baseline NetworkPolicy has a critical property: if you misconfigure the MinIO CIDR, **all** skill execution breaks. Every skill needs MinIO to download its package and upload its outputs. We learned this the hard way during an infrastructure migration when the MinIO IP changed but the NetworkPolicy wasn't updated. The symptoms were confusing: skills would start but immediately fail with "connection timeout" in the initContainer, and the error looked like a packaging problem, not a network problem.

The deploy README now includes a blast-radius warning and a mandatory pre-apply checklist:

> ⚠️ blast radius: baseline policy misconfiguration blocks ALL skill execution. Confirm `MINIO_ENDPOINT` matches the CIDR in `networkpolicy.yaml` before applying.

---

## The sandbox images

Every skill runs in a container built from one of our custom images. We maintain four:

| Image | Base | Size | Use case |
|-------|------|------|----------|
| `skill-python-runner:3.12` | python:3.12-slim | ~350MB | Default Python runtime |
| `skill-office:0.1.0` | python:3.12-slim + LibreOffice | ~1.5GB | Document conversion |
| `skill-node-runner:20` | Node.js 20 | ~300MB | JavaScript skills |
| `python:3.12-slim` (base) | Docker Hub retag | ~120MB | Build base only |

### The China infrastructure problem

Docker Hub is unreliable from within China — pulls frequently timeout or fail entirely due to network and compliance restrictions. Our solution: retag upstream images to Alibaba Cloud ACR (Container Registry):

```dockerfile
# base.Dockerfile — retag Docker Hub image to ACR
FROM python:3.12-slim
RUN mkdir -p /etc/pip && \
    printf '[global]\nindex-url = https://mirrors.aliyun.com/pypi/simple/\n' > /etc/pip.conf
```

```bash
docker build -t registry.cn-beijing.aliyuncs.com/klwk/python:3.12-slim \
  -f deploy/sandbox-images/base.Dockerfile .
docker push registry.cn-beijing.aliyuncs.com/klwk/python:3.12-slim
```

All downstream images (`python-runner`, `office`, `node-runner`) use this ACR base. Cluster-internal pulls from ACR are fast and reliable.

The base image also pre-configures the Aliyun PyPI mirror at the system level (`/etc/pip.conf`). This means `pip install` commands in downstream Dockerfiles and in user skills automatically use the domestic mirror — no per-file configuration needed.

### The python-runner image

The default runtime includes pre-installed libraries covering the most common skill use cases:

```dockerfile
FROM registry.cn-beijing.aliyuncs.com/klwk/python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
        unzip ca-certificates curl jq git \
    && rm -rf /var/lib/apt/lists/*

RUN groupadd -g 1000 app && \
    useradd -u 1000 -g 1000 -m -d /home/app -s /bin/bash app && \
    mkdir -p /workspace && chown -R 1000:1000 /workspace /tmp /home/app

RUN pip install --no-cache-dir \
        numpy pandas matplotlib requests beautifulsoup4 lxml openpyxl

USER 1000
WORKDIR /workspace
```

Key design decisions:

- **UID 1000 created explicitly.** The security context enforces `runAsUser: 1000` and `runAsNonRoot: true`. If the image doesn't have a user with UID 1000, the Pod fails to start with a cryptic error. We create it in every image.
- **System tools included.** Skills frequently need `curl` (fetch data), `jq` (parse JSON), `git` (clone repos), `unzip` (extract archives). Including them in the base image means skills don't need to install them at runtime.
- **Libraries pre-installed.** The data processing stack (numpy, pandas, matplotlib, requests) is pre-installed. This avoids the ~30-second `pip install` penalty on every skill run.

### The office image

The document processing image is the largest at 1.5GB:

```dockerfile
FROM registry.cn-beijing.aliyuncs.com/klwk/python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
        libreoffice-core libreoffice-writer libreoffice-impress libreoffice-calc \
        pandoc nodejs npm \
        fonts-noto-cjk fonts-dejavu \
        curl unzip ca-certificates

RUN pip install --no-cache-dir \
        python-docx python-pptx Pillow openpyxl lxml

RUN npm install -g docx@9.0.2

USER 1000
WORKDIR /workspace
```

CJK fonts (`fonts-noto-cjk`) are essential — our users create Chinese-language documents. Without CJK fonts, LibreOffice renders all Chinese text as empty boxes.

---

## The prepuller: solving cold-start latency

### The problem

The office image is 1.5GB. When a skill Job lands on a Kubernetes node that doesn't have the image cached, the pull takes 3–5 minutes. The user clicks "convert this document" and waits five minutes for a container to start. That's unacceptable.

### The solution: DaemonSet prepuller

A DaemonSet runs a `sleep infinity` container for each image on every node. Because the Pod is always running, the image is always cached. Kubernetes doesn't garbage-collect images from running Pods.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: skill-image-prepuller
  namespace: skillhub-run
spec:
  template:
    spec:
      securityContext: {runAsNonRoot: true, runAsUser: 1000}
      containers:
        - name: office
          image: registry.cn-beijing.aliyuncs.com/klwk/skill-office:0.1.0
          command: ["sleep", "infinity"]
          resources:
            requests: {cpu: "10m", memory: "16Mi"}
            limits: {cpu: "50m", memory: "64Mi"}
        - name: python
          image: registry.cn-beijing.aliyuncs.com/klwk/skill-python-runner:3.12
          command: ["sleep", "infinity"]
          resources:
            requests: {cpu: "10m", memory: "16Mi"}
            limits: {cpu: "50m", memory: "64Mi"}
        - name: node
          image: registry.cn-beijing.aliyuncs.com/klwk/skill-node-runner:20
          command: ["sleep", "infinity"]
          resources:
            requests: {cpu: "10m", memory: "16Mi"}
            limits: {cpu: "50m", memory: "64Mi"}
```

The resource footprint is minimal — each sleep container uses ~10MB of RAM. Across a 3-node cluster, that's ~90MB total. The five-minute cold-start elimination is worth it.

### The production incident: tag mismatch

Our most embarrassing prepuller bug: the prepuller was configured with `:latest` tags, but `resolve_image()` in runtime_service resolved to `:0.1.0`. These are different image references to Kubernetes — even if they point to the same image content, the node has `:latest` cached but the Job requests `:0.1.0`, which isn't cached. The prepuller was running happily, consuming resources, and doing absolutely nothing useful.

The symptom: office skills took 5+ minutes even though the prepuller DaemonSet was healthy. The fix was trivial (change the tag), but the debugging took hours because the prepuller appeared to be working correctly by every metric we checked.

**Lesson learned:** Prepuller image references must be generated by the same `resolve_image()` function that the runtime uses, not hardcoded in YAML. We haven't automated this yet (it's on the backlog), but the README now has a comment in all-caps:

> ★ Tag MUST exactly match resolve_image() output. Mismatch = prepull is useless.

---

## Async long tasks: beyond the 300-second barrier

### The problem

Some skills run for minutes: batch document processing, data migration, large report generation. Our synchronous HTTP timeout is 300 seconds. A skill that takes 10 minutes can't return a result through the synchronous path.

### The solution: submit, poll, retrieve

Skills declare `mode: async` in their manifest. The execution path changes:

1. **Submit**: `task_service.execute_resolved_async()` creates a K8s Job and returns a `run_id` immediately. The user sees "Skill is running in the background" within seconds.
2. **Poll**: The `async_poller` is a background asyncio loop that periodically scans MongoDB for `status=running` records with `async_ref` (the K8s Job reference).
3. **Collect**: When the Job completes, the poller reads logs, downloads output files, updates the status, and fires a completion event.
4. **Archive**: Output files are downloaded from any external URLs (if the skill fetched them), stored in MinIO, and attached to the conversation as downloadable links.

### State externalization

The critical design decision: **all recovery information is stored in MongoDB, not in memory.** The `async_ref` field contains the K8s Job name, namespace, and deadline. If the platform process restarts:

- The async_poller's first scan picks up all `status=running` records.
- K8s Jobs continue running independently — they don't know or care that the platform restarted.
- The poller adopts them and monitors to completion.

This is fundamentally different from background tasks that run in-process. If an in-process background task crashes, the work is lost. With state externalization, the platform can crash and restart without losing any running skills.

```python
# From async_poller.py — the invariant that makes recovery work:
# "All recovery info is in MongoDB, not memory. Process restart →
# poller's first scan auto-adopts running Jobs."
```

![Figure 2. Full Execution Flow](/assets/images/fig25-skill-execution-flow.svg)
*Figure 2. The execution flow from skill publish through async completion. Note the state externalization pattern: all recovery info lives in MongoDB, making the system resilient to process restarts.*

---

## The SkillHub graph: orchestrating the pipeline

The SkillHub execution is modeled as a LangGraph sub-graph with observable nodes:

```
[resolve_skill]
    ├── executable ──→ [clarify?] → [sandbox_graph] → [normalize] → [archive]
    └── instruction ──→ [inject SKILL.md] → [normalize] → [archive]
```

### Two skill types

**Executable skills** have an `x-astron-entry` field in their manifest (e.g., `python scripts/run.py`). They run code in the sandbox.

**Instruction skills** have no entry point. They provide a `SKILL.md` file that gets injected into the conversation agent's context. The agent uses the instructions to guide the user or perform a workflow — no code execution, no sandbox needed.

### The clarify node

Before execution, the system checks whether all required inputs are provided. If a skill needs `customer_id` and the user didn't provide it, the clarify node asks the user — not the LLM. This prevents the common failure mode where the LLM hallucinates a plausible-looking customer ID and the skill silently processes wrong data.

Platform-injected credentials (authentication tokens, internal API keys) are excluded from the missing-input check — users shouldn't be asked for things the platform provides automatically.

### The sandbox sub-graph

```
[prepare_sandbox] → [run_in_sandbox] → [collect_output] → [teardown]
```

Each step is instrumented with our lineage system (`node_run` context manager). If `run_in_sandbox` fails, the lineage trace shows exactly which step failed, with what error, and what the Pod's stdout contained. This four-step decomposition turns "the skill failed" into "the skill failed at step 2 (run_in_sandbox) after 45 seconds with exit code 137 (OOM-killed)" — actionable diagnostic information.

---

## MCP integration

Skills and MCP (Model Context Protocol) tools share the same sandbox infrastructure. When the AI agent calls an MCP tool, the tool execution follows the same path: resolve → sandbox → collect → archive. MCP tools get the same security context, the same NetworkPolicy isolation, and the same resource limits.

This was a deliberate design choice. We could have implemented a lighter-weight execution path for MCP tools (many are simple API wrappers that don't need full sandbox isolation). But maintaining two execution paths means two sets of security configurations, two sets of monitoring dashboards, and two sets of failure modes to understand. For our scale (~300 users), one well-understood path is better than two optimized-but-independently-maintained paths.

---

## Five trade-offs we made

### 1. One-shot Jobs vs. persistent sandboxes

We chose one-shot K8s Jobs (fresh Pod per execution) over persistent sandbox Pods. The cost: 2–3 second startup overhead per execution (with prepuller). The benefit: perfect isolation between runs. A skill that writes to `/tmp`, corrupts its environment, or leaks memory can't affect the next execution.

For our use case (skills that run for seconds to minutes, not interactive sessions), the startup overhead is negligible. If we were building an interactive coding environment (like a Jupyter notebook), we'd need persistent sandboxes — and a much more complex security model.

### 2. Internal vs. external trust boundary

The privilege gap between internal and external skills is deliberate. We could have given all skills the same network restrictions (maximum security) or the same network access (maximum capability). We chose a two-tier model because:

- Internal skills need internet access to call external services (document standardization API, data enrichment services). Blocking them breaks platform functionality.
- External skills have no legitimate reason to access the internet. Their inputs come from the platform, and their outputs go back to the platform.

The boundary is enforced at the K8s NetworkPolicy level, not in application code. A malicious skill can't override its `origin` field — that's set by the platform's publish pipeline, not by the skill manifest.

### 3. Prepuller cost vs. cold-start latency

The prepuller DaemonSet consumes ~200MB of RAM across the cluster (3 images × 3 nodes × ~16MB per sleep container, plus the cached image layers in the node's container runtime). We accepted this cost because a 5-minute cold-start makes the entire feature feel broken. Users don't care about container image caching — they care about "why does converting a document take five minutes?"

### 4. Presigned URLs vs. mounted credentials

We could mount MinIO credentials as Kubernetes Secrets into the sandbox Pod. This would be simpler (no URL expiry management) but would mean every skill has read/write access to our entire object storage. A single skill with a vulnerability could exfiltrate every file in the system.

Presigned URLs are scoped to one object, one operation (GET or PUT), and one time window. The complexity cost is real — we need to handle URL expiry and retry logic — but the security benefit is decisive.

### 5. State externalization vs. in-memory state

Storing async task state in MongoDB (rather than in-process memory) adds latency to every status update (~2ms for the async write). The benefit: the platform can crash and restart without losing track of any running skills. K8s Jobs are independent processes — they continue running regardless of what happens to the platform process. When the platform restarts, the poller's first scan finds them and resumes monitoring.

---

## What we'd do differently

1. **Automate prepuller tag synchronization.** The tag mismatch incident was entirely preventable. The prepuller YAML should be generated from the same `resolve_image()` function that the runtime uses, not maintained manually.

2. **Add runtime metrics from day one.** We added Prometheus metrics for skill execution (duration, exit code, image pull time) in month four. The debugging we did in months one through three — "is it slow because of the image pull or because of the skill itself?" — would have been trivial with metrics.

3. **Build a skill development sandbox.** Users currently develop skills locally and discover K8s-specific issues (like `readOnlyRootFilesystem` breaking their `pip install`) only after publishing. A "test in sandbox" button that runs the skill in the production security context without publishing it would save hours of debugging.

4. **Consider gVisor for defense in depth.** Our current security context is strong, but it relies on the Linux kernel's security mechanisms (seccomp, capabilities, namespaces). gVisor provides an additional layer by intercepting system calls at the user-space level. For our scale and threat model, the current setup is sufficient, but gVisor would be appropriate if we opened the platform to untrusted third-party developers.

5. **Image layer sharing is an unoptimized opportunity.** Our python-runner and office images share the same base (python:3.12-slim). In theory, only the delta layers need to be pulled for the second image. In practice, the way we build and tag images sometimes defeats layer sharing. A more disciplined multi-stage build strategy would reduce total storage and pull times.

---

## For the CTO evaluating AI platform security

If your organization is building an AI platform that executes user-provided code, here's what matters:

1. **Defense in depth is not optional.** No single mechanism (scanning, containers, network policy) is sufficient. Scanners miss novel malware. Containers have escape vulnerabilities. Network policies don't prevent data leakage through covert channels. Three independent layers with different failure modes provide genuine security.

2. **Kubernetes is the right execution platform.** Not because it's trendy, but because it provides the security primitives (security contexts, resource limits, network policies, namespace isolation) that you'd otherwise have to build yourself. If you're already running K8s for other workloads, skill sandboxing is a natural extension.

3. **Cold-start latency is a product problem, not just an infrastructure problem.** Users don't know or care about container image caching. If the AI says "I'll convert your document" and then nothing happens for five minutes, they'll stop using the feature. The prepuller DaemonSet is a ~200MB investment that makes the difference between "this feature works" and "this feature is broken."

4. **State externalization is worth the complexity.** If your AI platform crashes and loses track of running background tasks, users lose work and trust. Externalizing state to a database makes the system resilient to failures that will inevitably occur in production.

5. **The trust boundary must be enforced at the infrastructure level.** Application-level trust (checking a flag in code before allowing network access) can be bypassed by a sufficiently clever attacker. Network-level trust (Kubernetes NetworkPolicy enforced by the CNI) cannot be bypassed from within a Pod. Put your trust boundaries where the enforcement is strongest.

---

## References

- [kubiyabot/skill](https://github.com/kubiyabot/skill) — Docker runtime one-shot container model (MIT), adapted to K8s Job
- [Kubernetes NetworkPolicy documentation](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [gVisor](https://gvisor.dev/) — User-space kernel for container sandboxing
- [YARA-X](https://virustotal.github.io/yara-x/) — Pattern matching for malware detection

---

*The system described serves ~300 enterprise users on-premise. All infrastructure runs within domestic boundaries in compliance with China's PIPL. No internal identifiers, company names, or proprietary configurations are shared in this post.*
