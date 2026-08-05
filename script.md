# Script - "Cilium Network Policy Anti-Patterns I Learned the Hard Way"

## Slide 1 - Title

I everyone, thank you for joining this session, Let me first start with introducing myself.

## Slide 2 - About Myself

my name is Shubham, I am an Site Reliability Engineer at Obmondo, I am a Certified Kubernetes Administrator and have been contributing to various CNCF Projects since  - I also help maintain project KubeAid and KubeAid CLI - which is a ---

So lets get started.

---

## Slide 3 - Agenda

## Slide 4 - What is Cilium?

So I would assume most of us here know what Cilium is, for those who havent heard of cilium - It is an eBPF-based networking, observability, and security layer for Kubernetes. It replaces traditional iptables-based CNI + kube-proxy approach with an eBPF datapath.

Since its based on eBPF - programs run directly in the Linux kernel on every packet, replacing linear iptables rule chains with O(1) hash lookups.

It has Identity-Based Network Security - so policies are enforced on workload identity.

and because it is inspecting every packet at the kernel level to route and secure it, it gives us a lot of visibility on those packets and then you can use Hubble to export those flow data as logs, metrics, and a UI, without any extra instrumentation.

---

## Slide 5 - Why CiliumNetworkPolicy?

So why would you want to  use CiliumNetworkPolicy instead of native Kubernetes NetworkPolicy? A few real reasons.

- native NetworkPolicy is L3/L4 only, IPs and ports. CiliumNetworkPolicy gives you security at application layer L7 as well - so you can select which actual HTTP methods to allow or reject, and which which path to allow or reject.

- you can also apply CiliumClusterwideNetworkPolicy to define baseline policies across the whole cluster, native netpols aer ns scoped. even cilium netpols are ns scoped but there is a separate CR for cluster wide policies.

- Native netpols has no concept of hostnames at all — CIDR only. CNP can allow egress by domain name, dynamically resolved.

- native netpols has no concept of an explicit deny. We can do that with cilium netpols. and in that case - deny always wins over allow in this case.

- Native netpols can only reference Pods/namespaces via selectors, or raw CIDR blocks. cilium netpols also have predefined identities - like world, host, cluster, kube-apiserver, remote-node.

## Slide 6 - Section divider

Now lets move towards the antipatterns

## Slide 7 - Anti-Pattern 1: toServices vs toEndpoints



## Slide 9 - Anti-Pattern 3: Hardcoding kube-apiserver's ClusterIP

"Classic timeout-during-an-incident story: a pod needs to talk to the Kubernetes API server, someone hardcodes `toCIDR: 10.96.0.1/32` because that's the apiserver's ClusterIP they saw in `kubectl get svc -n default kubernetes`. Works in dev. Times out somewhere else - because that cluster reaches the apiserver through an external load balancer on `:6443`, not the in-cluster Service IP at all.

`toEntities: [kube-apiserver]` is the fix - it's a special entity Cilium resolves dynamically to wherever the apiserver is actually reachable from, so you stop guessing IP and port entirely."

> Extra - when do pods even need this? kube-apiserver is the source of truth for everything - secrets, configmaps, CRDs, pod status. But if your pod only *consumes* Kubernetes resources passively - mounted ConfigMaps/Secrets, env vars from a Secret - it never talks to the apiserver directly. Kubelet does that on the pod's behalf and writes the result to the node's filesystem or injects it at pod start. You only need `kube-apiserver` egress when a process **inside** the pod is making live API calls at runtime - client-go, kubectl, an SDK doing GET/WATCH/PATCH. Good context if someone asks "how do I even know if I need this rule."
>
> Extra: the other built-in entities worth knowing - `world` (everything outside the cluster), `host` (the local node), `remote-node` (other cluster nodes), `cluster` (all cluster entities). `world` is what Anti-Pattern 10 (LoadBalancer traffic) needs.

---

## Slide 10 - Anti-Pattern 4: Selecting on Unique-per-Pod Labels

"This one is sneaky because it looks totally reasonable: select a specific StatefulSet pod by `statefulset.kubernetes.io/pod-name: oncall-pgsql-1`. Except Cilium maintains an internal list of label patterns it deliberately **excludes** from identity calculation - specifically to avoid identity explosion, because if every pod got its own identity, you'd lose the whole point of identity-based policy grouping. Pod-name-style labels are on that exclusion list.

Practical effect: the endpoint literally never carries that label as part of its security identity, so a selector built around it matches nothing - forever, silently.

Fix: select on a label that's shared across all replicas of the thing you actually care about - `cnpg.io/cluster: oncall-pgsql` plus the namespace - one stable identity for the whole set."

> Extra: the exact exclusion list lives in Cilium's docs under "Limiting Identity-Relevant Labels" - worth linking if someone wants the authoritative list rather than trial-and-error.

---

## Slide 11 - Anti-Pattern 5: toFQDNs Without a DNS Proxy Rule

"This is the one that burned the most debugging hours for us, because it doesn't fail consistently - it fails *intermittently*, which is the worst kind of bug.

Here's the actual mechanism: when your pod does a DNS lookup, Cilium's eBPF checks - does *this specific pod's own policy* have a `rules.dns` block? If yes, the DNS proxy activates for that endpoint, intercepts the DNS response, and records the IP-to-FQDN mapping in that pod's own FQDN cache. If no - the DNS query and response still go through fine, but Cilium never learns that this particular IP belongs to `api.telegram.org`. Nothing in that pod's cache gets updated.

Then later, when the pod's traffic actually hits the `toFQDNs` rule, Cilium checks its cache for that pod: 'is this destination IP associated with the FQDN I'm allowed to reach?' If the cache is empty or expired - blocked. If it happens to have a cached mapping - maybe from a lucky global-cache hit from another pod - allowed. Same policy, same traffic, different outcome depending on cache state. That's why it looks flaky instead of just broken.

Critical detail: DNS proxy activation is strictly per-endpoint. Having a DNS rule in some other policy - like your cluster's default-deny policy - does **not** activate the proxy for this pod. Every policy that uses `toFQDNs` needs to ship its own `rules.dns` block for the pods it selects."

---

## Slide 12 - Anti-Pattern 6: Default-Deny Surprises

"The mental model most people bring from native NetworkPolicy is 'I'm only ever adding allow rules, nothing gets more restrictive.' Cilium breaks that assumption on day one: the very first CiliumNetworkPolicy that selects a given pod flips that pod's enforcement mode from 'never' to 'always' - for *both* directions, not just the one you were thinking about.

So a team ships a policy that's only trying to lock down egress to a database, and suddenly DNS breaks, kubelet health-check probes get dropped, the pod starts restarting - and none of that was explicitly denied by anything. It's an emergent side effect of enforcement mode flipping on.

The fix is really a sequencing discipline: treat default-deny as day-one design, not something you back into. Allow DNS egress with its DNS rule, allow kubelet's health-check probes, allow kube-apiserver where it's actually needed, *then* layer in your app-specific rules - and roll the whole thing out in audit/monitor mode first so you see what would have been dropped before you commit to enforcing it."

---

## Slide 13 - Anti-Pattern 7: Hardcoded Namespaces & Domains

"This is specifically a scaling problem for us, because we don't write one CiliumNetworkPolicy - we ship these as centralized Helm-based addons across every cluster and every team we support. A hardcoded namespace like `prod`, or a hardcoded FQDN like `api.telegram.org` baked directly into the YAML, means the moment that changes anywhere downstream, someone is hand-editing every copy of that chart across every cluster instead of changing one line in `values.yaml`.

The fix is boring but non-negotiable at our scale: template it. `{{ .Release.Namespace }}` for namespace, `{{ .Values.cnp.external.fqdns.telegram }}` for external domains, with the actual value living in `values.yaml`. One place to change, everywhere picks it up."

---

## Slide 14 - Anti-Pattern 8: toServices + toPorts, and Silent L7 No-ops

"Two separate landmines bundled into one slide because they're both 'the policy does less than it appears to, with no error telling you so.'

First: combining `toServices` with `toPorts` in the same rule isn't supported by Cilium - it produces confusing egress denies that look like a normal policy misconfiguration but are actually just an unsupported field combination.

Second, and this one's sneakier: if you write L7 rules - HTTP methods, Kafka topics, whatever - but the L7 proxy isn't actually enabled cluster-wide, Cilium doesn't error out. It just silently falls back to enforcing L3/L4 only. Your policy YAML says 'only allow GET requests to this path' and what you actually get is 'allow all traffic on this port.' You won't find that out from a `kubectl apply` - you find it out from a security review, or worse.

Takeaway: verify the L7 proxy is actually enabled cluster-wide *before* you rely on any http/gRPC/Kafka rule doing what it says."

---

## Slide 15 - Anti-Pattern 9: Datapath Mode Mismatch Across Environments

"This one's not a YAML mistake at all - it's an environment-parity mistake that *looks* like a policy bug.

We run staging on VXLAN tunnel mode - Cilium's default - and production on native routing with BGP peering. The policy YAML is byte-for-byte identical between the two. But MTU headroom, how encryption interacts with the datapath, even what your troubleshooting signal looks like - all of that differs by datapath mode. So you get the classic 'works in staging, breaks in prod' bug report, someone spends an hour re-reading the policy line by line, and the policy was never the problem.

If you can, keep datapath mode identical across environments and pin it explicitly rather than relying on the default. If you can't - for cost or infra reasons - document the difference and test encryption/MTU behavior for each mode deliberately, before you're debugging it live during an incident."

---

## Slide 16 - Anti-Pattern 10: Assuming LoadBalancer Traffic Is Pre-Authorized

"Mental shortcut that trips people up: 'this traffic came in through our internet-facing LoadBalancer Service, so it's already been through some kind of gate - it must be fine.' It hasn't been through a gate. It's been through Service load-balancing, which is a completely separate step from policy enforcement.

A packet can pass Service DNAT successfully and still get dropped immediately afterward by policy - because the policy engine only ever evaluates *world → Pod* identity. It doesn't know, and doesn't care, that the traffic happened to arrive via an internet-facing Service versus, say, a random IP hitting a NodePort directly. If you want that path to work, you need an explicit `fromEntities: [world]` (or a scoped `fromCIDR`) - passing through the LB doesn't grant anything on its own."

---

## Slide 17 - Anti-Pattern 11: Expecting Instant Enforcement on Label Changes

"Incident-response instinct: relabel a compromised or misbehaving pod - say, slap `role: quarantined` on it - expecting that to immediately sever its existing connections to sensitive backends. That's not guaranteed.

Changing a pod's labels recomputes its Cilium identity, and that does trigger the endpoint to regenerate its policy. But already-established connections - think conntrack-tracked flows - aren't necessarily re-evaluated against the new policy instantly; enforcement is largely applied at connection-setup time. I want to be careful here - exact behavior is version- and configuration-dependent, so I'm not stating this as an absolute law, but as a real gap between what people *assume* relabeling does and what it's guaranteed to do.

Practical incident-response takeaway: if you need a hard, immediate cut, don't rely on a relabel alone - kill the pod or the connection directly."

---

## Slide 18 - Anti-Pattern 12: "Services Aren't Identities" - Don't Select on a ClusterIP

"This is really the conceptual root underneath Anti-Pattern 2, stated on its own because it's worth internalizing as a mental model, not just a specific gotcha.

DNAT resolves a Service IP to a real backend Pod IP *before* the identity/policy check ever happens. Cilium's policy engine never actually evaluates traffic against a Service - only against post-DNAT Pod identity, or `world`. So if you hardcode a Service's ClusterIP into a `toCIDR` rule, it'll work - right up until that Service gets recreated and picks up a new ClusterIP, at which point the rule silently stops matching anything.

To be precise, `toServices` itself is a legitimate, supported field - it's not wrong to reference a Service by name. The anti-pattern specifically is reaching for `toCIDR`/`fromCIDR` with a Service's IP baked in, instead of using the dedicated `toServices` selector, which resolves the Service dynamically and needs no IP kept in sync by hand."

---

## Slide 19 - Anti-Pattern 13: CVE-2026-33726 - Ingress Silently Bypassed, Same-Node L7

"Last one, and it's not a YAML mistake at all - it's a real, disclosed vulnerability, so I want to be precise about it: CVE-2026-33726, GitHub advisory GHSA-hxv8-4j4r-cqgv.

Under a specific combination - Per-Endpoint Routing enabled, BPF Host Routing disabled - Ingress NetworkPolicies were not enforced for pod traffic to L7 Services (Envoy, GAMMA) with a local backend on the same node. No log entry, no drop event, nothing - the traffic that should have been denied by your Ingress policy just went through.

Why this deserves a slide of its own: that combination isn't some obscure edge-case config - it's the *default* under several common cloud-IPAM setups. Cilium ENI mode on EKS, Azure IPAM, some GKE configurations all auto-enable Per-Endpoint Routing, meaning EKS-with-Cilium-ENI-mode is probably the single most common affected environment in the wild.

Fixed versions: 1.17.14, 1.18.8, 1.19.2, and anything after. If you're running an affected version on one of those cloud-IPAM setups, this isn't a policy-authoring problem you can fix in YAML - it's a 'go check your Cilium version' problem, today, after this talk."

---

## Slide 20 - Debugging: How We Actually Find the Drop

"When something's blocked and you don't know why, this is the actual workflow, not the theoretical one.

One - identify the failing pod. The application log almost always names the peer it couldn't reach; that's your starting point, not guesswork.

Two - find the Cilium pod running on the *same node* as the pod you're debugging, and query Hubble scoped to that namespace or pod, filtered to `DROPPED` verdicts:

```
kubectl exec -n cilium <cilium-pod> -- \
  hubble observe --namespace <ns> --verdict DROPPED --last 50
```

You can scope it further to one pod, or `--follow` it live while you reproduce the issue.

Three - read the verdict output. `policy-verdict:none` means no policy exists that allows this traffic at all - you're probably missing a rule, not fighting a wrong one. `EGRESS DENIED` vs `INGRESS DENIED` tells you which side needs the fix - don't waste time editing the wrong pod's policy.

Four - fix it, re-apply, re-test. Don't guess twice; let Hubble confirm the fix actually worked before you move on."

> Extra: `kubectl get cep -n <namespace>` lists every CiliumEndpoint in a namespace - handy for spot-checking which pods even *have* a resolved identity. `kubectl get ciliumnetworkpolicies -n <namespace>` lists every CNP in play, useful when you're not sure which of several policies is the one actually selecting your pod.

---

## Slide 21 - The Checklist

"Quick recap, eight things - steal this, put it in your policy PR template if you want:

Always add the namespace label to every selector. Prefer `toEndpoints` over `toServices` for pod-to-pod traffic. Use `toEntities: [kube-apiserver]` - never hardcode its IP. Never select on unique-per-pod labels. Every `toFQDNs` policy ships its own DNS proxy rule. Plan default-deny for DNS, health checks, and apiserver access *first*, before app rules. Template namespaces and domains via `values.yaml` - never hardcode. And confirm the L7 proxy is actually enabled before you rely on any http/kafka rule doing what it says."

---

## Slide 22 - Thank You / Q&A

"That's thirteen ways I've personally shipped a CiliumNetworkPolicy that looked correct and wasn't. Happy to take questions - and if you want the deeper writeup, `docs.cilium.io/security/policy` is the canonical reference, and `editor.networkpolicy.io` is a genuinely useful visual editor if you're prototyping a policy before committing it to a chart."

---

## Q&A prep - things that might come up but aren't on any slide

- **"Does the `k8s:` prefix matter?"** No, in the vast majority of real clusters - it defaults to `any:` if omitted, which matches all sources. Only matters if you're disambiguating label sources (e.g., container-runtime labels vs. Kubernetes labels).
- **"How do I check what identity a pod actually has?"** `kubectl get cep -n <ns> <pod> -o jsonpath='{.status.identity.labels}' | jq`
- **"What's the full list of labels Cilium excludes from identity?"** Point to Cilium's "Limiting Identity-Relevant Labels" doc - don't try to recite it from memory.
- **"Does my pod even need kube-apiserver egress?"** Only if something inside the pod is making live API calls at runtime (client-go, kubectl, an SDK). Mounted ConfigMaps/Secrets and env-var injection are handled by kubelet on the pod's behalf - no egress rule needed for those.
- **"Is `toServices` just wrong, then?"** No - it's a legitimate field for the cases it's built for. The anti-pattern is specifically hardcoding a ClusterIP into `toCIDR` instead of using it, and separately, expecting `toServices` alone to authorize pod-to-pod traffic after DNAT.
