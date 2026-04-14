+++
authors = ["Kalin Daskalov"]
title = "Lab Notes: Why llm-d pushed me out of Ingress (and into agentgateway)"
date = "2026-04-14"
description = "When LLM routing became a scheduling problem, ingress stopped being enough."
categories = ["lab notes"]
tags = ["k8s", "mlops", "intel", "homelab", "devops", "kubernetes", "k3s", "llm-d", "gateway-api", "agentgateway", "fluxcd", "openwebui", "vllm"]
+++

> Link to Part 4: [Intel AI Inference Platform MVP 2 with llm-d](https://blog.sonda.red/posts/06-intel-homelab-4/)

> Disclaimer: This is not production guidance and it is not sponsored. It documents what actually ran in my homelab. Please double-check before you roll it into your own setup.

---

## Table of Contents
- [Table of Contents](#table-of-contents)
- [00 This started as "just expose one more endpoint"](#00-this-started-as-just-expose-one-more-endpoint)
- [01 Why Ingress was no longer enough](#01-why-ingress-was-no-longer-enough)
- [02 The second migration](#02-the-second-migration)
- [03 Timeouts became part of architecture](#03-timeouts-became-part-of-architecture)
- [04 Where the stack landed](#04-where-the-stack-landed)
- [05 Closing notes](#05-closing-notes)

---

## 00 This started as "just expose one more endpoint"

In Part 4, I introduced `llm-d`. It was a solution after my original thought that inference on Kubernetes can follow a "standard" path of:

1. Get a model running locally
2. Run it in a container
3. Run it in Kubernetes
4. Expose it with Ingress

Step 4 was the original plan. I had Ingress in place for all my other apps, so it felt natural to just add another route.

With one model pod, Ingress looks fine. With two replicas serving the same model, the question changes from "which Service" to "which replica should take this request right now." Ingress only sees HTTP endpoints. It does not know anything about decode pods, cache locality, or inference-specific backends, while llm-d's scheduler does.

Once `llm-d` became the center of inference, Ingress stopped fitting naturally.

The routing question was no longer "which Service" but "which inference pool backend." Inference pools are a new kind of backend that encode model-serving semantics, and they are only supported in Gateway API with the Inference Extension.

Kubernetes docs now [explicitly recommend Gateway API over Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/#what-is-ingress), and the Ingress API is marked as frozen (stable, but no new feature development). Around the same time, `ingress-nginx` retirement was [announced on November 11, 2025](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/), with best-effort maintenance through March 2026.

So the path became clear: keep existing Ingress where needed, but invest new routing work in Gateway API.

One dependency bump later, I learned this was not one migration but three:

1. Ingress -> Gateway API because of `llm-d`
2. `kgateway` -> `agentgateway` because the provider path changed
3. Default timeouts -> explicit timeout policies because LLM traffic is long-lived

I originally chose `kgateway` for its early Gateway API support, but the provider ecosystem is still evolving. When `agentgateway` emerged with a more focused vision on AI workloads, it made sense to follow that path.

---

## 01 Why Ingress was no longer enough

`llm-d` depends on Gateway API Inference Extension CRDs, so Gateway API became a hard dependency in this repo.

```yaml
# infrastructure/gateway-api/gateway-api-inference-extension.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: gateway-api-inference-extension
  namespace: gateway-system
spec:
  url: https://github.com/kubernetes-sigs/gateway-api-inference-extension
  ref:
    tag: v1.4.0
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: gateway-api-inference-extension
  namespace: gateway-system
spec:
  path: ./config/crd
  dependsOn:
    - name: gateway-api
```

Then the route itself stopped targeting a plain Service and started targeting an `InferencePool`.

```yaml
# deployments/llm-d/inference-scheduling/httproute.yaml
rules:
  - backendRefs:
      - name: llm-d-inferencepool
        kind: InferencePool
        group: inference.networking.k8s.io
        port: 8200
```

And the pool encodes model-backend semantics (`v1` API, decode label selection, target port):

```yaml
# deployments/llm-d/inference-scheduling/inferencepool/inferencepool-values.yaml
inferencePool:
  apiVersion: inference.networking.k8s.io/v1
  targetPortNumber: 8200
  modelServerType: vllm
  modelServers:
    matchLabels:
      llm-d.ai/role: "decode"
```

At that point, I could have kept a mixed world: Ingress for apps, Gateway API for inference. I decided not to. Running two routing models in parallel was extra cognitive overhead for policy, debugging, and observability. Since `llm-d` already forced me to learn Gateway API, consolidating routes made more sense.

The unplanned part came next.

---

## 02 The second migration

A concrete timeline from my infra repo:

1. `f01d2cf` (2026-02-07): migrate app ingress routes to Gateway API
2. `0a1415d` (2026-02-07): migrate infra ingress routes (`Flux`, `MinIO`, `Grafana`, `Prometheus`, `VictoriaLogs`)
3. `829d7a4` (2026-02-13): split `OpenWebUI` route behavior and add long-stream policy
4. `7b9a137` -> `4fde0b6` (2026-04-07): Gateway API dependency bump to `v1.5.1`, then revert to `v1.4.1`
5. `1904628` + `8397a51` (2026-04-07): `kgateway` -> `agentgateway` refactor plus listener/route cleanup

The revert in step 4 was where I learned about the provider migration. I had been following `kgateway` releases, but I missed the deprecation notice for AI Gateway support. When I bumped to `v1.5.1`, all my inference routes stopped working and I had to dig into release notes and code to understand why.

The key nuance: `kgateway` was not dead. The AI/inference path moved. In `kgateway` 2.1 release notes, AI Gateway and Gateway API Inference Extension support on Envoy-based proxies was marked deprecated in favor of `agentgateway` proxy support, with removal planned in 2.2. Then 2.2 introduced dedicated `agentgateway.dev` APIs and a separate chart/controller split ([release notes](https://kgateway.dev/docs/envoy/latest/reference/release-notes/), [2.2 breaking changes](https://kgateway.dev/docs/2.2.x/release-notes/breaking-changes/)).

The obvious diff looked small:

```diff
# commit 1904628 (llm-d infra values)
- provider: kgateway
+ provider: agentgateway

- gatewayClassName: kgateway
+ gatewayClassName: agentgateway
```

But there was more surface area:

```diff
# commit 1904628 (route parent refs)
- namespace: kgateway-system
+ namespace: agentgateway-system
```

```diff
# commit 8397a51 (listener + route cleanup)
- - name: infer-https
+ - name: inference-gateway-https

- - name: agtw-https
+ - name: agentgateway-admin-ui-https

- # HTTP to HTTPS redirect routes
+ # removed and consolidated around HTTPS listeners
```

After I stabilized these bindings, the next bottleneck was connection behavior.

---

## 03 Timeouts became part of architecture

The long-connection behavior of LLM inference and UI sessions forced me to treat timeout config as architecture, not optional tuning.

For external inference traffic:

```yaml
# deployments/llm-d/inference-scheduling/httproute-external.yaml
rules:
  - matches:
      - path:
          type: PathPrefix
          value: /
    timeouts:
      backendRequest: "3600s"
      request: "0s"
```

For OpenWebUI:

```yaml
# deployments/openwebui/httproute.yaml
rules:
  - matches:
      - path:
          type: PathPrefix
          value: /
    timeouts:
      backendRequest: "3600s"
      request: "0s"
```

If you serve short request/response APIs, defaults are often fine. If you serve token streams and slower model turns, default timeout assumptions can quietly wreck UX. I started receiving strange "connection reset" errors in OpenWebUI and llm-d, and it took a while to connect the dots that these were not random network issues but timeout policies kicking in.

---

## 04 Where the stack landed

The current gateway shape is one shared `main-gateway` in `agentgateway-system`, with explicit HTTPS listeners per hostname.

```yaml
# infrastructure/agentgateway/gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
  namespace: agentgateway-system
spec:
  gatewayClassName: agentgateway
  listeners:
    - name: openwebui-https
      hostname: chat.sonda.red.intra
      port: 443
      protocol: HTTPS
    - name: inference-gateway-https
      hostname: infer.sonda.red.intra
      port: 443
      protocol: HTTPS
```

This ended up cleaner than what I had before:

1. Gateway API is now a stable dependency of the inference stack
2. `llm-d` routing semantics are explicit in manifests
3. The provider migration is complete and aligned with the current control plane
4. Long-lived connection behavior is handled in route policy, not left to defaults

---

## 05 Closing notes

What I expected to be one migration taught me three separate lessons:

1. Inference routing is not generic web routing. I think I'm spending the bulk of my time on these issues because of the unique semantics of LLM workloads, not just because of the newness of Gateway API.
2. Gateway provider lifecycle matters as much as application lifecycle, especially in a fast-evolving space like AI inference. I had to follow the provider ecosystem and be ready to adapt when the path changed. Things really are moving too fast. I go on a vacation and miss a release, and suddenly my whole routing layer breaks because of a provider migration I didn't know was coming.
3. Timeout policy is part of product behavior when LLMs are in the loop. A connection is more akin to a session than a request, and the "request" can be arbitrarily long.

---

## References

- [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)
- [Gateway API Inference Extension](https://github.com/kubernetes-sigs/gateway-api-inference-extension)
- [llm-d](https://github.com/llm-d/llm-d)
- [agentgateway](https://agentgateway.dev/)
- [kgateway release notes](https://kgateway.dev/docs/envoy/latest/reference/release-notes/)
- [kgateway 2.2 breaking changes](https://kgateway.dev/docs/2.2.x/release-notes/breaking-changes/)
- [Kubernetes: What is Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/#what-is-ingress)
- [ingress-nginx retirement announcement](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)
- [Gateway API HTTP timeouts](https://gateway-api.sigs.k8s.io/guides/http-routing/#timeouts)
