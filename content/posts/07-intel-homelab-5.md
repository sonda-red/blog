+++
authors = ["Kalin Daskalov"]
title = "Lab Notes: Why llm-d pushed me out of Ingress (and into agentgateway)"
date = "2026-04-14"
description = "When LLM routing became a scheduling problem, ingress stopped being enough."
categories = ["lab notes"]
tags = ["k8s", "mlops", "intel", "homelab", "devops", "kubernetes", "k3s", "llm-d", "gateway-api", "agentgateway", "fluxcd", "openwebui", "vllm"]
+++

[![Why llm-d pushed me out of Ingress and into agentgateway](/images/post-07/ingress-gateway-api.jpg)](/images/post-07/ingress-gateway-api.jpg)

> Link to Part 4: [Intel AI Inference Platform MVP 2 with llm-d](https://blog.sonda.red/posts/06-intel-homelab-4/)

> Disclaimer: This is not production guidance and it is not sponsored. It documents what actually ran in my homelab. Please double-check before you roll it into your own setup.

---

## Table of Contents
- [Table of Contents](#table-of-contents)
- [00 I came back from vacation to a broken routing layer](#00-i-came-back-from-vacation-to-a-broken-routing-layer)
- [01 Why Ingress couldn't follow](#01-why-ingress-couldnt-follow)
- [02 The migration I didn't plan for](#02-the-migration-i-didnt-plan-for)
- [03 When timeouts become product behavior](#03-when-timeouts-become-product-behavior)
- [04 Where the stack landed](#04-where-the-stack-landed)
- [05 Three lessons from three migrations](#05-three-lessons-from-three-migrations)
- [References](#references)

---

## 00 I came back from vacation to a broken routing layer

So I went on vacation for a week. Before I left I had queued a minor version bump on the gateway provider. Came back, checked Flux, and nothing was routing. All inference traffic was dead. Turns out the provider had deprecated the AI Gateway path I was using and I just hadn't noticed the release notes.

That was fun to debug on a Monday morning. But it also made me realize this whole thing was not one migration but three, and I should probably write down how I got here.

In Part 4, I introduced `llm-d`. It was a solution after my original thought that inference on Kubernetes can follow a "standard" path of:

1. Get a model running locally
2. Run it in a container
3. Run it in Kubernetes
4. Expose it with Ingress

Step 4 was the original plan. I had Ingress in place for all my other apps, so it felt natural to just add another route.

With one model pod, Ingress looks fine. With two replicas serving the same model, the question changes from "which Service" to "which replica should take this request right now." Ingress only sees HTTP endpoints. It does not know anything about decode pods, cache locality, or inference-specific backends, while llm-d's scheduler does.

{{< mermaid >}}
flowchart LR

    classDef node fill:#f7f7fb,stroke:#3f3f58,color:#2b2b35,stroke-width:1.6px
    classDef blue fill:#f7f9ff,stroke:#5b7cfa,color:#2b2b35,stroke-width:2px
    classDef orange fill:#fff8ef,stroke:#c98717,color:#2b2b35,stroke-width:2px
    classDef purple fill:#f4efff,stroke:#7b61ff,color:#2b2b35,stroke-width:2px
    classDef pink fill:#faeff7,stroke:#b44b8a,color:#2b2b35,stroke-width:2px
    classDef note fill:#ececf2,stroke:#c8cad6,color:#44475a,stroke-width:1.2px

    client["Client"]
    ingress["ingress-nginx\n(kube-system)\nLoadBalancer\n192.168.1.240"]

    subgraph apps["Apps"]
        owui["OpenWebUI"]
        minio["MinIO"]
        grafana["Grafana"]
    end

    subgraph inference["Inference"]
        svc["vLLM Service\n(round-robin)"]
        vllm1["vLLM pod\n(GPU 1)"]
        vllm2["vLLM pod\n(GPU 2)"]
    end

    client -->|HTTPS| ingress
    ingress --> owui
    ingress --> minio
    ingress --> grafana
    ingress -->|"same as any\nother Service"| svc
    svc --> vllm1
    svc --> vllm2

    class client,ingress node
    class owui,minio,grafana blue
    class svc orange
    class vllm1,vllm2 pink
{{< /mermaid >}}
*Stage 1: Ingress NGINX*

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

## 01 Why Ingress couldn't follow

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

At that point I could have just left Ingress in place for the non-inference apps and only used Gateway API for `llm-d`. I thought about it for maybe a day. Running two routing stacks in parallel sounded like the kind of decision I'd regret every time something broke and I had to check both. Since I already had to learn Gateway API anyway, I just moved everything over.

The part I didn't plan for came next.

---

## 02 The migration I didn't plan for

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

## 03 When timeouts become product behavior

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

The current gateway shape is one shared `main-gateway` in `agentgateway-system`, with explicit HTTPS listeners per hostname. But to appreciate how it got here, here's what the kgateway stage looked like before it broke:

{{< mermaid >}}
flowchart LR

    classDef node fill:#f7f7fb,stroke:#3f3f58,color:#2b2b35,stroke-width:1.6px
    classDef blue fill:#f7f9ff,stroke:#5b7cfa,color:#2b2b35,stroke-width:2px
    classDef orange fill:#fff8ef,stroke:#c98717,color:#2b2b35,stroke-width:2px
    classDef purple fill:#f4efff,stroke:#7b61ff,color:#2b2b35,stroke-width:2px
    classDef pink fill:#faeff7,stroke:#b44b8a,color:#2b2b35,stroke-width:2px
    classDef note fill:#ececf2,stroke:#c8cad6,color:#44475a,stroke-width:1.2px

    client["Client"]

    subgraph kgw["kgateway-system"]
        gw["main-gateway\n(kgateway)\nHTTPS :443"]
    end

    subgraph llmd["llm-d"]
        int_gw["inference-gateway\n(kgateway)\n:80"]
        pool["InferencePool\n:8200"]
        epp["EPP"]
        vllm["vLLM decode pods"]
    end

    owui["OpenWebUI"]

    client -->|HTTPS| gw
    gw -->|"chat.sonda.red.intra"| owui
    gw -->|"infer.sonda.red.intra ❌"| int_gw
    int_gw --> pool
    pool --> epp
    epp --> vllm

    class client,gw node
    class owui blue
    class int_gw,pool,epp purple
    class vllm pink
{{< /mermaid >}}
*Stage 2: kgateway — the inference path (❌) broke when AI Gateway support was deprecated*

The red link is the one that broke. kgateway 2.1 deprecated AI Gateway and Inference Extension support on Envoy proxies, and 2.2 removed it. The inference path moved to `agentgateway`, and everything had to follow.

The clearest way to think about the current layout is to separate public endpoints from inference-internal routing. `chat.sonda.red.intra` is the browser UI. `infer.sonda.red.intra` is the single OpenAI-compatible API surface (`/v1/...`) used by OpenWebUI and direct clients. The hostname tells you which API surface you are hitting; the request body still carries `model=...`, and the llm-d layer still has to pick a concrete backend pod.

{{< mermaid >}}
flowchart TB

    classDef node fill:#f7f7fb,stroke:#3f3f58,color:#2b2b35,stroke-width:1.6px
    classDef blue fill:#f7f9ff,stroke:#5b7cfa,color:#2b2b35,stroke-width:2px
    classDef orange fill:#fff8ef,stroke:#c98717,color:#2b2b35,stroke-width:2px
    classDef purple fill:#f4efff,stroke:#7b61ff,color:#2b2b35,stroke-width:2px
    classDef pink fill:#faeff7,stroke:#b44b8a,color:#2b2b35,stroke-width:2px
    classDef note fill:#ececf2,stroke:#c8cad6,color:#44475a,stroke-width:1.2px

    subgraph clients["Clients"]
        direction LR
        browser["Browser user"]
        api["Direct API client"]
    end

    subgraph traffic["Live request path"]
        direction LR
        gw["1. Main gateway<br/>agentgateway<br/>HTTPS :443"]
        owui["2a. OpenWebUI<br/>chat.sonda.red.intra"]
        infer["2b. llm-d inference gateway<br/>Service :80"]
        epp["3. EPP<br/>cache/load-aware chooser"]

        subgraph pods["4. Candidate decode backends"]
            direction LR
            ds1["Pod A<br/>DeepSeek R1 Llama 8B"]
            ds2["Pod B<br/>DeepSeek R1 Llama 8B"]
            ds3["Pod C<br/>Qwen 1.5B"]
        end
    end

    subgraph controlplane["Kubernetes control objects"]
        direction LR
        ext["External HTTPRoute<br/>infer.sonda.red.intra"]
        route["Internal HTTPRoute"]
        pool["InferencePool<br/>selector: llm-d.ai/role=decode"]
    end

    browser -->|"chat.sonda.red.intra"| gw
    gw -->|"serve UI"| owui
    owui -->|"infer.sonda.red.intra<br/>POST /v1/...<br/>model=..."| gw
    api -->|"infer.sonda.red.intra<br/>POST /v1/...<br/>model=..."| gw
    gw -->|"forward infer traffic"| infer
    infer -.->|"consult EPP"| epp
    epp -.->|"selected backend"| infer
    infer --> ds1
    infer --> ds2
    infer --> ds3

    ext -.->|"bind hostname"| gw
    ext -.->|"forward"| infer
    route -.->|"attach"| infer
    route -.->|"backendRef"| pool
    pool -.->|"select by label"| ds1
    pool -.->|"select by label"| ds2
    pool -.->|"select by label"| ds3

    class browser,api,gw node
    class owui blue
    class infer,epp,ext,route,pool purple
    class ds1,ds2,ds3 pink
{{< /mermaid >}}
*Stage 3: agentgateway + llm-d. Solid arrows show request flow. Dashed arrows show configuration and backend selection. `chat.sonda.red.intra` serves the UI; `infer.sonda.red.intra` is the single OpenAI-compatible API surface; llm-d still chooses one concrete decode backend for each request.*

This was the mental model I was missing at first. `HTTPRoute` and `InferencePool` are not extra network hops in the same sense as Gateway -> pod. They are the objects that tell the llm-d inference gateway how to resolve a request. The actual runtime path is simpler: request hits `main-gateway`, inference traffic gets handed to the llm-d gateway, EPP picks one backend from the pool, and only then does a specific decode pod answer.

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

## 05 Three lessons from three migrations

What I expected to be one migration taught me three separate lessons:

1. Inference routing is not generic web routing. I think I'm spending the bulk of my time on these issues because of the unique semantics of LLM workloads, not just because of the newness of Gateway API.
2. Gateway provider lifecycle matters as much as application lifecycle. I missed one release, went on vacation, came back to a dead routing layer. Things are moving fast in this space and nobody is going to wait for you to catch up.
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
