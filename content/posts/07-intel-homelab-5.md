+++
authors = ["Kalin Daskalov"]
title = "Lab Notes: Why llm-d pushed me out of Ingress (and into agentgateway)"
date = "2026-04-08"
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
- [02 The second migration I did not plan](#02-the-second-migration-i-did-not-plan)
- [03 Timeouts became part of architecture](#03-timeouts-became-part-of-architecture)
- [04 Where the stack landed](#04-where-the-stack-landed)
- [05 Closing notes](#05-closing-notes)

---

## 00 This started as "just expose one more endpoint"

I thought this would be a small networking change.

At first, my mental model was simple: I already had UI traffic working, so I just needed one more route for inference and we were done.

That model was wrong.

Once `llm-d` became the center of inference, routing stopped being "send traffic to a Service" and became "send traffic to a scheduler that understands LLM backends." That subtle shift is where Ingress stopped fitting naturally.

At the same time, the gateway implementation itself changed under me: while I was moving away from Ingress, `kgateway` was being deprecated and I had to migrate to `agentgateway` too.

So this post is the bigger picture behind that migration path:

1. Ingress -> Gateway API because of `llm-d`
2. `kgateway` -> `agentgateway` because the provider changed
3. Timeout and connection handling changes because LLM traffic is long lived by default

---

## 01 Why Ingress was no longer enough

`llm-d` depends on Gateway API Inference Extension CRDs, so Gateway API became a hard dependency, not a preference.

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

And the pool defines the model backend semantics (`v1` API, decode label matching, target port):

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

So this was the key realization for me: Ingress was still useful for web apps, but my inference path was no longer just web routing.

---

## 02 The second migration I did not plan

I started the first migration with `kgateway` and in the middle of that work, the direction shifted to `agentgateway`.

The base swap looked small:

```diff
# commit 1904628 (llm-d infra values)
- provider: kgateway
+ provider: agentgateway

- gatewayClassName: kgateway
+ gatewayClassName: agentgateway
```

But it was not just two lines. Listener names, route section references, and route shape had to move together.

```diff
# commit 8397a51 (gateway/listener + route cleanup)
- - name: infer-https
+ - name: inference-gateway-https

- - name: agtw-https
+ - name: agentgateway-admin-ui-https

- # HTTP to HTTPS redirect routes
+ # removed and consolidated around HTTPS listeners
```

The lesson for me was practical: in Gateway API setups, `sectionName` is a real contract. Renaming listeners is easy. Renaming listeners without breaking every dependent `HTTPRoute` is the real work.

---

## 03 Timeouts became part of architecture

The long-connection behavior of LLM inference and UI sessions forced me to treat timeout config as core architecture, not optional tuning.

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

If you are serving short APIs, default timeouts can be perfectly fine. If you are serving long streams and slower model turns, default assumptions can quietly wreck UX and make failures look random.

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
3. The provider migration is complete and aligned with the current gateway control plane
4. Long-lived connection behavior is handled where it should be: in route policy

---

## 05 Closing notes

What I expected to be one migration taught me three separate lessons:

1. Inference routing is not generic web routing
2. Gateway provider lifecycle matters as much as application lifecycle
3. Timeout policy is part of product behavior when LLMs are in the loop

I still think of this as MVP territory, but it is a more honest MVP now. The manifests encode the real behavior instead of relying on defaults and luck.

In the next notes, I want to go deeper on request-level behavior through the pool itself: what gets routed where, and why.
