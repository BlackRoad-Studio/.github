# BlackRoad Architecture Overview

> **The Core Thesis:** BlackRoad is a routing company, not an AI company.

---

## Executive Summary

We don't train models or buy GPUs. We connect users to intelligence that already exists (Claude, GPT, Llama, NumPy, legal databases, etc.) through an orchestration layer we own.

**The insight:** Intelligence is already trained. Libraries already exist. The value is in routing requests to the right tool at the right time—not in building another brain.

---

## Infrastructure (~$40/month recurring)

| Layer | Service | Role |
|-------|---------|------|
| Edge/CDN | Cloudflare | Handles millions of connections, DNS, DDoS |
| CRM/Data | Salesforce (Free Dev Edition) | Customer data, 15K API calls/day |
| Code/CI | GitHub Enterprise | 15 organizations, deployment |
| Mesh Network | Tailscale | Private encrypted connections between nodes |
| Cloud Node | Digital Ocean (Shellfish) | Internet-facing server |

---

## Hardware (Owned, Not Rented)

A Raspberry Pi cluster running specialized roles:

| Node | Hardware | Role |
|------|----------|------|
| **lucidia** | Pi 5 + Pironman + Hailo-8 | Salesforce sync, RoadChain/Bitcoin |
| **octavia** | Pi 5 + Pironman + Hailo-8 | AI routing decisions (26 TOPS), 3D printing |
| **aria** | Pi 5 | Agent orchestration, Cloudflare Workers |
| **alice** | Pi 400 | Kubernetes + VPN hub (mesh root) |
| **shellfish** | Digital Ocean droplet | Public-facing gateway |

Plus dev machines (Mac = "cecilia", iPhone = "arcadia") and edge devices (ESP32s, LoRa modules for future deployment).

### Local Inference via Raspberry Pi 5 Clusters

The system utilizes clusters of Raspberry Pi 5 nodes to host local LLMs, replacing centralized API calls with local, private inference and mitigating rate limits from cloud providers.

| Component | Technical Specification | Functional Role |
|-----------|------------------------|-----------------|
| Compute Node | Raspberry Pi 5 (8GB LPDDR4X) | General Purpose Inference and Control |
| Inference Accelerator | Raspberry Pi AI Hat 2 (40 TOPS) | Dedicated INT8 LLM Processing |
| Network Layer | Gigabit Ethernet with PoE+ HAT | Synchronized Node Communication |
| Storage | NVMe SSD (M.2 Interface, 256GB+) | Model Weights and Agent Memory |
| Software Stack | LiteLLM Proxy / Ollama / llama.cpp | API Hosting and Load Balancing |

The Raspberry Pi AI Hat 2 (Hailo 10H NPU) enables efficient processing of quantized GGUF models, achieving 5–15 tokens per second in clustered configurations using OpenMPI for parallelization.

### Copilot Proxy Configuration for Local Offloading

To bypass external rate limits and keep proprietary codebase context on the local network, the system overrides the default GitHub Copilot endpoints:

```bash
export GH_COPILOT_OVERRIDE_PROXY_URL="http://raspberrypi.local:4000"
```

The LiteLLM proxy translates requests into an OpenAI-compatible format and distributes them across the cluster using a round-robin load-balancing strategy.

---

## The Control Plane

```
┌─────────────────────────────────────────────────────────────┐
│              BLACKROAD UNIFIED CONTROL                       │
├─────────────────┬─────────────────┬─────────────────────────┤
│   SALESFORCE    │   CLOUDFLARE    │      GITHUB             │
│   CRM + API     │   Edge + DNS    │    Code + CI            │
└────────┬────────┴────────┬────────┴──────────┬──────────────┘
         │                 │                   │
         └────────────────┬┴───────────────────┘
                          ▼
                    ┌──────────┐
                    │ OPERATOR │  ← We own this
                    └────┬─────┘
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
    ┌─────────┐    ┌──────────┐    ┌─────────┐
    │ lucidia │    │ octavia  │    │  aria   │
    │ SF/Chain│    │ Hailo-8  │    │ Agents  │
    └────┬────┘    └────┬─────┘    └────┬────┘
         └───────────────┼───────────────┘
                         ▼
                    ┌─────────┐
                    │  alice  │  ← K8s + VPN hub
                    └─────────┘
```

**Key insight:** The OPERATOR sits between us and all external services. Cloudflare, Salesforce, and GitHub are utilities we command—not landlords we rent from. The control plane lives on hardware we own.

---

## The Routing Pattern

```
[User Request] → [Operator] → [Route to Right Tool] → [Answer]
                     │
                     ├── Physics question? → NumPy/SciPy
                     ├── Language task? → Claude/GPT API
                     ├── Customer lookup? → Salesforce API
                     ├── Legal question? → Legal database
                     └── Fast inference? → Hailo-8 local
```

The agent doesn't need to be smart. It needs to know **who to call.**

---

## The @BlackRoadBot Routing Matrix

When a user comments `@BlackRoadBot` on a GitHub issue or pull request, the bot identifies the target platforms based on the natural language intent and routes to the appropriate APIs.

### Salesforce CRM Integration

- **Task Creation:** An Apex middleware triggers creation of a Case or Custom Task object in Salesforce.
- **Data Activation:** Salesforce Data Cloud ingests telemetry from GitHub webhooks for real-time analytics on agent performance.
- **Webhooks:** The `salesforce-webhooks` package wires GitHub triggers directly to Salesforce endpoints.

### Hugging Face and Ollama Reasoning

- **Hugging Face Inference Endpoints:** The bot deploys dedicated model endpoints on Hugging Face via the `huggingface_hub` client for high-compute tasks.
- **Ollama Integration:** Routine tasks use the Ollama API, exposed via a Cloudflare Tunnel, running models such as `bartowski/Llama-3.2-3B-Instruct-GGUF`.

### DigitalOcean and Railway Infrastructure

- **DigitalOcean Droplets:** The bot uses the `doctl` CLI within GitHub Actions to manage droplets (rebuild, scale, etc.).
- **Railway Deployments:** Applications are deployed to Railway for hosting ephemeral test environments for new feature branches.

---

## The Business Model

| What We Own | What We Don't Need |
|-------------|-------------------|
| Customer relationships | Training models |
| Routing/orchestration logic | GPUs |
| Data and state | Data centers |
| The Operator | The intelligence itself |

When better models come out, we add them to the router. Infrastructure stays the same.

---

## The Math

At $1/user/month:

- 1M users = $12M/year
- 100M users = $1.2B/year
- 1B users = $12B/year

Ceiling: everyone who ever talks to AI.

---

## Organization Structure

BlackRoad operates across 15 specialized GitHub organizations within the [BlackRoad OS Enterprise](https://github.com/enterprises/blackroad-os). Each organization implements the principle of least privilege, isolating agents across functional domains.

| Organization | Primary Responsibility | Repository Examples |
|--------------|------------------------|---------------------|
| **Blackbox-Enterprises** | Corporate and Enterprise Integrations | blackbox-api, enterprise-bridge |
| **BlackRoad-AI** | Core LLM and Reasoning Engine Development | lucidia-core, blackroad-reasoning |
| **BlackRoad-Archive** | Long-term Data Persistence and Documentation | blackroad-os-docs, history-ledger |
| **BlackRoad-Cloud** | Infrastructure as Code and Orchestration | cloud-orchestrator, railway-deploy |
| **BlackRoad-Education** | Onboarding and Documentation Frameworks | br-help, onboarding-portal |
| **BlackRoad-Foundation** | Governance and Protocol Standards | protocol-specs, governance-rules |
| **BlackRoad-Gov** | Regulatory Compliance and Policy Enforcement | compliance-audit, regulatory-tools |
| **BlackRoad-Hardware** | SBC and IoT Device Management | blackroad-agent-os, pi-firmware |
| **BlackRoad-Interactive** | User Interface and Frontend Systems | blackroad-os-web, interactive-ui |
| **BlackRoad-Labs** | Experimental R&D and Prototyping | experimental-agents, quantum-lab |
| **BlackRoad-Media** | Content Delivery and Public Relations | media-engine, pr-automation |
| **BlackRoad-OS** | Core System Kernel and CLI Development | blackroad-cli, kernel-source |
| **BlackRoad-Security** | Auditing, Cryptography, and Security | security-audit, hash-witnessing |
| **BlackRoad-Studio** | Production Assets and Creative Tooling | lucidia-studio, creative-assets |
| **BlackRoad-Ventures** | Strategic Growth and Ecosystem Funding | tokenomics-api, venture-cap |

Cross-organization repository access is managed via GitHub Apps, preferred over Personal Access Tokens (PATs) for their short-lived, granular permissions and ability to act on behalf of an organization.

---

## Domain Architecture

The project manages primary domains orchestrated via Cloudflare. Cloudflare Tunnels securely expose local Raspberry Pi nodes to the public internet, allowing the bot to invoke local inference models without exposing internal network ports.

| Domain | Functional Use Case | Associated Organization |
|--------|---------------------|-------------------------|
| blackboxprogramming.io | Developer Education and APIs | Blackbox-Enterprises |
| blackroad.io | Core Project Landing Page | BlackRoad-OS |
| blackroad.company | Corporate and HR Operations | BlackRoad-Ventures |
| blackroad.me | Personal Agent Identity Nodes | BlackRoad-AI |
| blackroad.network | Distributed Network Interface | BlackRoad-Cloud |
| blackroad.systems | Infrastructure and System Ops | BlackRoad-Cloud |
| blackroadai.com | AI Research and API Hosting | BlackRoad-AI |
| blackroadinc.us | US-based Governance and Legal | BlackRoad-Gov |
| blackroadqi.com | Quantum Intelligence Research | BlackRoad-Labs |
| blackroadquantum.com | Primary Quantum Lab Interface | BlackRoad-Labs |
| lucidia.earth | Memory Layer and Personal AI | BlackRoad-AI |
| lucidia.studio | Creative and Asset Management | BlackRoad-Studio |
| roadchain.io | Blockchain and Witnessing Ledger | BlackRoad-Security |
| roadcoin.io | Tokenomics and Financial Interface | BlackRoad-Ventures |

---

## The @blackroad-agents Deca-Layered Scaffold

Every task entering the system is processed through a ten-step scaffolding workflow, ensuring high-fidelity execution through review, distribution, and deployment stages.

| Step | Name | Description |
|------|------|-------------|
| 1 | **Initial Reviewer** | Layer 6 (Lucidia Core) agent reviews the request for clarity, security compliance, and resource availability, then generates a preliminary execution plan. |
| 2 | **Task Distributor to Organization** | Routes the task to one of the 15 BlackRoad organizations based on functional domain (hardware, security, cloud, etc.). |
| 3 | **Task Distribution to Team** | Refines and distributes the task to a specific team within the organization; pauses for human-in-the-loop (HITL) approval on high-risk operations. |
| 4 | **Task Update to Project** | Records the task in a GitHub Project board and synchronizes metadata (Request ID, timeline) with Salesforce for an enterprise-level audit trail. |
| 5 | **Task Distribution to Agent** | Instantiates or assigns a specialized autonomous agent (e.g., "fastapi-coder-agent", "doctl-infrastructure-agent") using a Planner-Executor-Reflector design pattern. |
| 6 | **Task Distribution to Repository** | Identifies the target repository, creates a new branch following GitHub Flow, and ensures changes are tested in isolation. |
| 7 | **Task Distribution to Device** | Routes tasks requiring physical execution (firmware updates, droplet rebuilds) to the device layer for local hardware offloading. |
| 8 | **Task Distribution to Drive** | Distributes artifacts (logs, reports, documentation) to Google Drive via a Service Account (GSA) pattern for consistent write access. |
| 9 | **Task Distribution to Cloudflare** | Executes network configuration changes (new tunnels, DNS record modifications) to ensure newly deployed services are immediately reachable and secured. |
| 10 | **Task Distribution to Website Editor** | Routes changes to an AI-driven website editor or headless CMS for autonomous generation of landing pages, blog posts, or documentation updates. |

---

## BlackRoad CLI v3 — Layered Architecture

The BlackRoad CLI v3 is built upon a modular architecture that defines the behavior of agents and infrastructure across eight layers.

| Layer | Name | Responsibility |
|-------|------|----------------|
| 3 | Agents/System | Autonomous agent lifecycle management and system-level processes |
| 4 | Deploy/Orchestration | Infrastructure provisioning across cloud and local nodes |
| 5 | Branches/Environments | Management of ephemeral environments and git-branching logic for agentic code tests |
| 6 | Lucidia Core/Memory | Long-term context storage, state transitions, and simulation data |
| 7 | Orchestration | High-level task distribution logic powering the @blackroad-agents scaffold |
| 8 | Network/API | External interface providing REST and GraphQL endpoints for the @BlackRoadBot matrix |

A failure in Layer 8 (Network) does not affect the persistence of state in Layer 6 (Memory). The architecture is designed for resilience through this layered isolation.

---

## roadchain — Witnessing Architecture

The BlackRoad ecosystem incorporates a cryptographic strategy called **roadchain**. Unlike traditional blockchains focused on consensus-based proof, roadchain functions as a "witnessing" architecture.

Every state transition—whether an agent committing code or a bot routing a task—is hashed using SHA-256 and appended to a non-terminating ledger. This creates an immutable record of **what happened** rather than "what is true."

---

## Rate Limit Mitigation

Managing a distributed enterprise requires robust error handling. When an agent hits a rate limit, the system employs mitigation protocols:

| Provider | Observed Limit | Mitigation Protocol |
|----------|----------------|---------------------|
| GitHub Copilot | RPM / Token Exhaustion | Redirect to local Raspberry Pi LiteLLM proxy |
| Hugging Face Hub | IP-based Rate Limit | Rotate HF_TOKEN or use authenticated SSH keys |
| Google Drive | Individual User Quota | Use Shared Drives with GSA "Content Manager" role |
| DigitalOcean API | Concurrent Build Limits | Queue tasks via Layer 7 Orchestration |
| Salesforce API | Daily API Request Cap | Batch updates via Data Cloud Streaming Transforms |

If a task fails at any scaffold layer, the system creates a GitHub Issue containing detailed logs from Layer 6 (Lucidia Core). A human reviewer or a "Reflect and Retry" plugin assesses the failure and resolves it by adjusting agent prompts or switching inference models.

---

## Future Scaling: 30k Agents

The ecosystem is designed for massive scale, with active development on the `blackroad-30k-agents` repository. This project aims to orchestrate 30,000 autonomous agents using Kubernetes auto-scaling and self-healing, transitioning from Raspberry Pi clusters to larger, more power-efficient ARM-based data centers mirroring the decentralized witnessing architecture of roadchain.

---

## Implementation Guide

The FastAPI pattern is the starting point:

1. **Expose endpoints** (`/physics/hydrogen`, `/relativity/time-dilation`)
2. **Operator routes** (keyword matching → right function)
3. **Log everything** (JSON audit trail → future ledger)

This is the Operator pattern in miniature. Start with physics, extend to every domain.

---

## Verification

- **Source of Truth:** GitHub (BlackRoad-OS) + Cloudflare
- **Hash Verification:** PS-SHA-∞ (infinite cascade hashing)
- **Authorization:** Alexa's pattern via Claude/ChatGPT

---

*Last Updated: 2026-02-27*
*BlackRoad OS, Inc. - Proprietary and Confidential*
