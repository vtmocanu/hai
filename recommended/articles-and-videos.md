# Articles & Videos

### [AI Just Killed the Software Engineer (And Created Something Better)](https://www.youtube.com/watch?v=sDy55DIkIJ0)

Viktor Farcic's take on how AI is rewriting the software engineering profession right now, not in some distant future. Starts with hard data on AI's impact on productivity and revenue, then gets into the human side: identity crisis for devs whose craft is shifting under them. Covers new development cycles, spec-first workflows, how much autonomy to give AI agents, and why context engineering is the single biggest factor most teams ignore. The uncomfortable part: org restructuring, pod models, platform teams, which roles expand vs shrink, and what happens to junior developers when nobody is hiring them. Ends with concrete Monday-morning moves.

### [I Built Custom AI Agents for My Team: Here's the Complete Blueprint](https://www.youtube.com/watch?v=98yYcWwD95I)

Argues that generic AI coding assistants like Claude Code and Cursor only know public information, leaving engineers to repeatedly reinvent the same internal solutions. Lays out a production blueprint for custom agents: system context with company policies and conventions, custom tools for internal APIs, vector-database retrieval for runbooks and docs, multi-agent orchestration with narrow specialists rather than one monolith, security guardrails with human-in-the-loop for critical operations, and OpenTelemetry observability for continuous improvement. End-to-end and practical, not theoretical.

### [Stop Using CPU Limits](https://home.robusta.dev/blog/stop-using-cpu-limits)

Makes the case that CPU limits are an antipattern in Kubernetes. CPU is compressible—pods get throttled but keep running—so limits just prevent pods from using available resources. Memory is different: it's incompressible and needs limits to prevent OOM chaos. The takeaway: use CPU requests (not limits) and always set memory limits.

### [Production-Ready Dockerfiles](https://www.youtube.com/watch?v=ueTe-VQaD7c)

Tackles a critical but often overlooked aspect of containerization: writing secure, optimized Dockerfiles for production. Exposes common mistakes—running as root, using latest tags, copying secrets, bloated images—then systematically walks through best practices: minimal base images, multi-stage builds, layer caching, non-root users, and pinned versions. Concludes with AI-powered tools that can automatically generate production-ready Dockerfiles.

### [Why KRM is the Royal Road to AI Ops](https://www.linkedin.com/pulse/why-krm-kubernetes-resource-model-royal-road-ai-ops-mialon-sgife/)

Makes the case for Kubernetes Resource Model (KRM) as the foundation for AI-driven infrastructure operations. The argument: KRM provides a universal, declarative API for managing all infrastructure — not just containers. Tools like Crossplane extend this to cloud resources, making everything manageable through the same Kubernetes control plane. This uniformity is what enables AI agents to reason about and operate infrastructure at scale.

### [KRM-Native GitOps: Without Flux There is Nothing](https://www.linkedin.com/pulse/krm-native-gitops-yes-without-flux-fluxcd-nothing-mialon-wsmue/)

A deep technical comparison of Flux vs ArgoCD. The core argument: Flux aligns with Kubernetes' design principles while ArgoCD introduces architectural divergence. Key points: Flux delegates to Kubernetes RBAC (ArgoCD adds its own layer), Flux integrates SOPS natively for secrets (ArgoCD says "GitOps is not for secrets"), Flux uses the actual Helm SDK (ArgoCD uses `helm template`). "What you see is what you deploy."

