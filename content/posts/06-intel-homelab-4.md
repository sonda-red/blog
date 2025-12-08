+++
authors = ["Kalin Daskalov"]
title = "Lab Notes: Intel AI Inference Platform MVP 2 with llm-d"
date = "2025-12-08"
description = "With the cluster in place, I built the second layer of the stack."
categories = ["lab notes"]
tags = ["k8s", "mlops", "intel", "homelab", "devops", "gpu", "arc", "a770", "intel-arc", "kubernetes", "k3s"]
+++

# Lab Notes: Intel AI Inference Platform MVP 2

## 00 How llm-d became the routing brain of my Intel stack

In [Part 3](https://blog.sonda.red/posts/05-intel-homelab-3/) I ended with a working Intel only inference MVP:

* Models packaged as ModelKits via KitOps and stored in Harbor
* A shared PVC populated by `kitops-init` for fast local model loading
* Intel Arc A770 GPUs allocated through Dynamic Resource Allocation
* vLLM with IPEX as the serving engine behind an OpenAI compatible API
* Open WebUI on top, with xpumanager and Prometheus watching everything

That stack did the job. DeepSeek R1 ran on my GPUs, the configuration was reproducible and stable.

Something was bugging me however. In Kubernetes we're used `12 factor apps`, or atleast close enough to that philosphy so that you treat scale, replication and stateless behavior as default and somewhat straightforward to manage.

The vLLM deployments I made previously are one pod, with one model, loaded in one GPU. Tensor parallelism enables this same one pod to house its model on two or more GPUs but the single replica point of failure remains. You don't just scale a vLLM or similar deployment to 2 replicas. You'll get another instance on another GPU you can put a load balancer in front of two separate inference workloads but it's a mistake to consider them basic http services:

* Requests have wildly different prompt and response lengths
* Each replica holds its own KV and prefix cache
* Long prompts and agent loops can pin a GPU for quite a while

I feel lucky to be honest. I feel lucky I chose vLLM as my main inference engine of choice by gut feeling, didn't really get deep into until later. I'm also lucky I decided to take a listen to the Kubernetes Podcast while I was loading the dishwasher one night and stumbled upon the llm-d episode mid October: https://kubernetespodcast.com/episode/258-llmd/

You can try to imagine me holding a dirty dish and just listening, forgetting about the dishwaser at all. Says lots about the talk but yeah, in short:

This post is an extenstion my at-home inference MVP: the same hardware and the same vLLM engine, but with llm-d and the Gateway API Inference Extension sitting in the middle as a real routing and scheduling layer.

---

## 00 What made llm-d click for me

I loaded the dishasher and instantly set to search what the hell is llm-d and can I install it. The docs at [llm-d.ai](https://llm-d.ai/) were somewhat alright, but I also found an interesting, though rather short [Hacker News thread](https://news.ycombinator.com/item?id=44040883) where I also learned about Nvidia's Dynamo inference solution but that's for another time.

In that thread, one of the maintainers described llm-d as **three clean layers**. ([Hacker News][1])

1. Balance and schedule incoming requests to the right backend
2. Run model server replicas on different hardware topologies
3. Provide a prefix caching hierarchy with tuned variants for different use cases

So llm-d is not “yet another AI platform.” It is a very opinionated three tier architecture:

* A routing and scheduling plane
* A pool of model servers (vLLM in my case)
* A cache hierarchy for tokens and prefixes

It uses the **Gateway API Inference Extension** to define routing, request priorities and flow control in Kubernetes owned APIs, rather than hiding them in a controller.

I did not need a pipeline SDK. I already had vLLM and my own GitOps opinions. I needed a routing brain that understands “LLM server” as a special kind of backend and in an area I'm comfortable - Kubernetes.

---

## 02 LLMs as CPUs, and what Kubernetes does not know yet

The Podcast episode I mentioned in the beginning made me see the stack with a different analogy: an LLM is **a new kind of CPU**.

* vLLM is the microarchitecture
* The GPU plus context window plus KV cache is the “core”
* Each request is a process whose “cost” depends on token shapes
* The prefix cache is a special memory that can dramatically change cost

Kubernetes today is very good at:

* CPU and memory allocation
* Container scheduling and bin packing
* Generic load balancing through Services and Ingress

It has no built in concept of “LLM core capacity” or cache locality.

That missing layer is exactly where llm-d and the Gateway API Inference Extension fit. They sit between “L4 networking” and “model server,” and they give Kubernetes some language for this new CPU.

---

## 03 How llm-d slots into the Intel MVP

The nice part: I did not have to throw away anything from Part 3.

* vLLM stays
* Intel GPUs and DRA stay

Interestingly I implemented llm-d before they had DRA support and still used GPU plugins as default. I had to Frankenstein a solution between their helmfile(which I love with my whole heart) approach and my new love of FluxCD. Thankfully, the PR enabling DRA support got merged quite quickly https://github.com/llm-d-incubation/llm-d-modelservice/pull/144 and now I can run vLLM with DRA natively through llm-d's modelservice chart.

* ModelKit and kitops-init stay

However with a small but important change: I now use `kitops-init` to populate a PVC that is then mounted into the vLLM pods declared by llm-d's `modelservice` chart. This way I can keep my existing model packaging and loading flow, however this time not as sidecar but as a volume mount.

* GitOps with Flux and Kustomize stays

The updated flow looks like this:

1. A client or Open WebUI sends a request to an OpenAI compatible endpoint
2. `kgateway` receives it on port 80 and matches an `HTTPRoute`
3. The `HTTPRoute` forwards to an `InferencePool` instead of a Service
4. Envoy inside the gateway calls the Endpoint Picker (EPP) via External Processing
5. The EPP looks at pod load and cache hints and returns a single backend
6. Envoy forwards the request to the chosen vLLM pod

From the vLLM pod’s perspective, nothing changed. It is still just serving tokens over HTTP. From the cluster’s perspective, a whole new set of decisions became explicit and observable.

---

## 04 The three llm-d components in this cluster

In my repo, the integration lives under `deployments/llm-d/inference-scheduling`. It breaks into three pieces.

### 04.1 modelservice: vLLM on Intel GPUs with DRA and ModelKit

The llm-d `modelservice` chart is where I declare “how many vLLM replicas, which image, which GPUs.”

The important bits from `modelservice-values.yaml`:

```yaml
modelArtifacts:
  name: "deepseek-ai/DeepSeek-R1-Distill-Llama-8B"
  uri: "hf://deepseek-ai/DeepSeek-R1-Distill-Llama-8B"  # placeholder, overridden by ModelKit PVC via post-render

dra:
  enabled: true
  type: "intel-a770-single"
  claimTemplates:
    - name: "intel-a770-single"
      class: "gpu.intel.com"
      count: 1

decode:
  replicas: 2
  containers:
    - name: vllm
      image: intelanalytics/ipex-llm-serving-xpu:latest
      args: |
        python -m ipex_llm.vllm.xpu.entrypoints.openai.api_server \
          --model /data/ds-r1-llama-8 \
          --tensor-parallel-size 1 \
          --enable-prefix-caching
      volumeMounts:
        - name: modelkit
          mountPath: /data  # PVC populated by kitops-init
```

This is basically Part 3, wrapped in a chart:

* vLLM with Intel IPEX
* one A770 per decode pod from DRA
* prefix cache enabled
* models pulled from Harbor into `/data` by kitops-init

llm-d does not replace this. It assumes you know how you want to serve the model.

### 04.2 InferencePool: an LLM aware backend instead of a Service

The **Gateway API Inference Extension** adds `InferencePool` as a new kind of backend. It is like a Service that knows its pods are LLM servers.

Conceptually, my pool looks like this:

```yaml
apiVersion: inference.networking.k8s.io/v1alpha1
kind: InferencePool
metadata:
  name: llm-d-inferencepool
  namespace: llm-d
spec:
  selector:
    matchLabels:
      llm-d.ai/role: decode
  targetPort: 8000   # vLLM HTTP port
  extensionRef:
    kind: Service
    name: llm-d-epp   # Endpoint Picker service
    namespace: llm-d
```

Three key ideas:

* `selector` ties the pool to vLLM decode pods that serve the same model
* `targetPort` indicates which port speaks the model protocol
* `extensionRef` points at the Endpoint Picker that must be consulted before routing

This is the Kubernetes owned description of “these pods form a single logical LLM backend, and this is the brain that decides which one gets each request.” ([Hacker News][1])

### 04.3 HTTPRoute: Gateway to InferencePool

The last piece of the path is a regular `HTTPRoute` that targets the InferencePool instead of a Service:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: llm-d-route
  namespace: llm-d
spec:
  parentRefs:
    - name: llm-d-infra-inference-gateway
      namespace: llm-d
      kind: Gateway
  rules:
    - backendRefs:
        - name: llm-d-inferencepool
          kind: InferencePool
          group: inference.networking.k8s.io
          port: 8000
      matches:
        - path:
            type: PathPrefix
            value: /
      timeouts:
        backendRequest: 3600s
        request: 3600s
```

Clients do not see any of this. They still send `POST /v1/chat/completions` with an OpenAI style payload. The only difference is that the backend reference is an InferencePool that has an Endpoint Picker attached.

---

### 05.1 Routing is no longer blind

Instead of “Service sprays requests evenly,” I now have a component whose whole job is to decide which pod should handle which request.

The Endpoint Picker:

* Tracks per pod load and queue depth
* Uses cache hints and conversation metadata
* Keeps multi turn flows on the same backend when that makes sense
* vLLM exposes metrics about cache hits and token latency
* EPP uses hints and past routing decisions to keep similar prompts together

The llm-d maintainer even called out that their main users are large inference deployers who already run Kubernetes across providers, often mixing serving, batch and training on the same fleet. ([Hacker News][2])
I'm not a H100 datacenter, for which this tooling can generally solve a lot more problems, but even on my two A770s, you can see benefit and learnings from having this layer in place.

llm-d calls it something like paved ways. Well lit paths: https://llm-d.ai/docs/guide#well-lit-path-guides

### 05.3 GPU allocation is part of a story, not a trick

In Part 3, DRA was mostly a cleaner way to say “give this pod one Arc A770” instead of abusing node labels.

With llm-d in place:

* DRA sets which pods have which GPUs
* InferencePool defines which pods form a logical backend
* EPP knows which backends exist and how busy they are

It feels less like a trick and more like a small, coherent “LLM scheduler” built out of parts Kubernetes understands.

---

## 07 A personal checkpoint

Since I started this homelab project, the surface area has grown, but the core activity has not changed much:

* Run DeepSeek R1 locally
* Containerise it
* Schedule it on Intel GPUs in Kubernetes
* Scale it across more pods or devices

The outcome from the outside is trivial: a prompt goes in, tokens come out, one GPU is busy.

The interesting part has moved from “can I make this run at all” to “what does the cluster think this workload is.”

MVP 1 treated the LLM as a slightly exotic web service.

MVP 2 with llm-d treats it as its own kind of CPU, with:

* a scheduler that speaks Kubernetes
* a routing brain that sees cache and load
* a serving engine that knows about tokens and KV cache

The hardware is the same. The model is the same. The difference is that I can now look at an `InferencePool` object, EPP metrics and vLLM stats and say “this is how my cluster is thinking about this model.”

That is why llm-d does not feel like vaporware to me. It is not trying to be a platform. It is trying to be a better brain for a very specific kind of workload. On my small Intel homelab, that is exactly what I needed for MVP 2.

[1]: https://news.ycombinator.com/item?id=44040883&utm_source=chatgpt.com "LLM-D: Kubernetes-Native Distributed Inference"
[2]: https://news.ycombinator.com/item?id=44043135&utm_source=chatgpt.com "This is really interesting. For SOTA inference systems, I've ..."
