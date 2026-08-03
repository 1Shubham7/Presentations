# Script - "Cilium Network Policy Anti-Patterns I Learned the Hard Way"

Full script, slide by slide. Plain paragraphs are what to *say*. Boxed `> Extra:` callouts are knowledge that didn't fit on the slide but is worth keeping in your back pocket for Q&A - don't feel obligated to say these out loud unless someone asks or you have time to spare.

---

## Slide 1 - Title

(Yours - intro yourself, the talk title is already on screen.)

## Slide 2 - About Myself

(Yours.)

---

## Slide 3 - Agenda

"Quick roadmap: I'll spend a minute on what Cilium actually is, a minute on why we chose CiliumNetworkPolicy over the vanilla Kubernetes NetworkPolicy, and then the bulk of the talk - thirteen anti-patterns I either wrote myself or watched a teammate write, each with the wrong version, the fix, and the *why*. I'll close with how we actually debug a policy drop with Hubble, and a checklist you can steal on your way out."

---

## Slide 4 - What is Cilium?

"Cilium is an eBPF-based networking, observability, and security layer for Kubernetes. Instead of kube-proxy and iptables rules, it programs the kernel directly with eBPF - so service routing and policy enforcement happen in-kernel, not as a chain of iptables rules being walked packet by packet.

The part that matters most for this talk: Cilium's security model is *identity-based*, not IP-based. A policy is written against pod labels - a set of labels resolves to a numeric security identity, and policy is enforced between identities, not IPs. That single design choice is the root cause of most of the anti-patterns coming up.

It also ships Hubble for flow-level observability, and CiliumNetworkPolicy - CNP - as a superset of the native NetworkPolicy API with L7 awareness (HTTP, gRPC, Kafka), DNS-aware rules, and explicit entities like `kube-apiserver` and `world`."

> Extra: internally, every unique *set* of identity-relevant labels gets a numeric Security Identity. Cilium deliberately excludes certain high-cardinality label patterns (like `statefulset.kubernetes.io/pod-name`) from that calculation - that list is what causes Anti-Pattern 4 later.

---

## Slide 5 - Why CiliumNetworkPolicy?

"We centralize operator configs and NetworkPolicies for multiple teams across multiple clusters - that's KubeAid Addons. The practical consequence: one wrong pattern doesn't cost one team a bad afternoon, it gets copy-pasted into every cluster we manage.

Native NetworkPolicy can only *allow* - there's no explicit deny, no priority, and no DNS or L7 awareness. CNP gives us explicit deny, `toFQDNs`, `toServices`, `toEntities`, and both namespace-scoped and cluster-wide policies (CNP vs CCNP) so we're not duplicating the same rule per namespace.

The tradeoff: because enforcement is in-kernel via eBPF, the failure modes look genuinely different from what you're used to with iptables. That's basically the rest of this talk."

---

## Slide 6 - Section divider

"Thirteen mistakes. Wrong versus correct, every time - let's go."

---

## Slide 7 - Anti-Pattern 1: Forgetting Namespace Scoping

"CiliumNetworkPolicy resources are namespace-scoped by their own `metadata.namespace` - but that's just where the *policy object* lives. Every selector inside it - `endpointSelector`, `fromEndpoints`, `toEndpoints` - matches on **labels only**. There is no implicit 'stay within my namespace' behavior.

So here: a policy in `test-oncall` tries to allow traffic from `cloudnative-pg` pods by label - but those pods actually live in the `cloudnative-pg` namespace. No namespace label on the selector means it silently matches nothing. No error, no warning - traffic just gets denied and you're left wondering why.

Fix: always add the namespace label explicitly - `k8s:io.kubernetes.pod.namespace: cloudnative-pg` - alongside the app label."

> Extra: the `k8s:` prefix itself is actually **optional**. Cilium labels carry a source prefix internally (`k8s:`, `container:`, etc.), and if you omit the prefix in your selector it defaults to `any:`, which matches labels from any source. So `io.kubernetes.pod.namespace: traefik` and `k8s:io.kubernetes.pod.namespace: traefik` behave identically in almost every real cluster. You only need the explicit prefix if you need to disambiguate label sources. Good one to know if someone in the audience asks "wait, do I need that k8s: prefix?"
>
> Extra: you can see exactly how Cilium stored a pod's labels with `kubectl get cep -n <namespace> <pod-name> -o jsonpath='{.status.identity.labels}' | jq` - `cep` = CiliumEndpoint. Great live-debugging move if you want to demo this.

---

## Slide 8 - Anti-Pattern 2: toServices Instead of toEndpoints

"`toServices` looks like the obviously-correct way to say 'let this pod talk to my database service.' It isn't, for pod-to-pod traffic.

`toServices` only authorizes traffic to the Service's ClusterIP. But by the time that packet is actually delivered, kube-proxy or Cilium's own eBPF datapath has already DNAT'd it straight to a backend pod IP - and policy enforcement happens *after* that translation, against the pod's identity, not the Service's. So the rule that looks like it should work, doesn't.

`toEndpoints`, selecting on the backend pod's labels directly, is what we settled on and recommend by default - it matches the real destination identity, and it also survives a Helm release name changing, since we're selecting on a stable label like `cnpg.io/cluster`, not a name that's templated per release."

---

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
