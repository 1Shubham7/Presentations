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

### Anti-Pattern 1: toServices vs toEndpoints

If you are using toServices as selector - the port number must match targetPort, not the Service's exposed port. which is a bit confusing but After 
DNAT, the packet's destination port becomes the Pod's actuall port. If you use Service's exposed port - it will silently never matches.

- What actually happens is that Pod A sends a request to the Service's ClusterIP, on service's port since thats the only address it knows. it doesnt know about the pod B's IP or port.

- eBPF intercepts this at Pod A's veth - eBPF program looks up the Service in the service map, picks Pod B, and rewrites the packet: destination IP becomes Pod B's real IP, and destination port becomes pod B's port.

- eBPF resolves identities - Pod A's IP → Pod A's identity, Pod B's IP → Pod B's identity.

- then it does policy map lookup: (Pod A identity → Pod B identity, port 80, TCP). if the toPorts is correctly written then it matches and the packet proceeds to Pod B.
  
- If the pod b also has ingress policy, that Pod B side gets checked too

### Anti Pattern 2: 

Also you should always prefer toEndpoints on a stable label - then you can sidestep from all of these edge cases, and also that survives Helm templated Service names changing across releases.

### Anti Pattern 3: Unique-per-Pod Labels

Cilium ignores "unique per Pod" or "unique per deployment revision" labels for identity - so while using selectors - dont use such labels. Cilium maintains this list of label patterns excluded from identity computation - this done to prevent every single Pod from getting its own unique identity - for eg. if there are 2 pods in a deployment - and cilium were to consider unique per pod labels nw if we allow ingress to that deployment - the label we use in "toEndpoints matchLabels" may match will one pod but not with another - which would defeat the purpose of identity-based policy grouping. If you write a CiliumNetworkPolicy selector against one of these excluded labels, it silently matches nothing. you can scan this QR to checkout the list of excluded labels. and you can also configure your cilium to append labels to it for your specific cluster.

## 4: toFQDNs Without a DNS Proxy Rule (wip)

client-1 wants to visit api.telegram.org. First step, always: it has to do a DNS lookup — ask "what IP is api.telegram.org?"
This DNS lookup is itself just network traffic — a request going out on port 53, to CoreDNS.
Before this traffic leaves client-1's veth, eBPF checks: does any policy that selects client-1 have a rules.dns block covering this traffic? In our correct case — yes, it does.
Because that check passed, eBPF redirects this specific DNS query into Cilium's DNS proxy (instead of just letting it go straight to CoreDNS untouched).
The DNS proxy forwards the query to CoreDNS, gets the real answer back (api.telegram.org = 1.2.3.4), and — because it's sitting in the middle of this conversation — it sees this answer.
The DNS proxy records this: 1.2.3.4 is api.telegram.org, and updates a shared table Cilium keeps for the whole node/cluster (we'll call this "the big lookup table" for now — more on it in a second).
client-1 receives the DNS answer normally, now tries to actually connect to 1.2.3.4.
eBPF checks the policy: client-1 has a toFQDNs: api.telegram.org rule. Cilium checks the big lookup table: "is 1.2.3.4 labeled as api.telegram.org?" — yes, because step 6 just wrote that in.
Allowed. Connection proceeds.

Watching a DNS lookup (so Cilium learns the IP) only happens for a Pod if that Pod's own policy has the DNS rule. But once any Pod's lookup gets watched and written to the shared table, any other Pod can benefit from that entry — until it expires. That's why a missing rules.dns block doesn't fail every time — it fails only when nobody else happened to refresh that entry recently.

## Anti-Pattern 5: Default-Deny Surprises

"The mental model most people bring from native NetworkPolicy is 'I'm only ever adding allow rules, nothing gets more restrictive.' Cilium breaks that assumption on day one: the very first CiliumNetworkPolicy that selects a given pod flips that pod's enforcement mode from 'never' to 'always' - for *both* directions, not just the one you were thinking about.

So a team ships a policy that's only trying to lock down egress to a database, and suddenly DNS breaks, kubelet health-check probes get dropped, the pod starts restarting - and none of that was explicitly denied by anything. It's an emergent side effect of enforcement mode flipping on.

The fix is really a sequencing discipline: treat default-deny as day-one design, not something you back into. Allow DNS egress with its DNS rule, allow kubelet's health-check probes, allow kube-apiserver where it's actually needed, *then* layer in your app-specific rules - and roll the whole thing out in audit/monitor mode first so you see what would have been dropped before you commit to enforcing it."

---

## Anti-Pattern 6: Hardcoded Namespaces & Domains

when you apply  a CiliumNetworkPolicy that selects a Pod and you havent added any policy denying traffic - it automatically converts it to default deny, unless explicitly allowed. this happens per direction - ingress and egress - independently.

So never start with writing netpols for an app and just enforcing them, follow this step by step process - this always works

1. write a default deny policy that blocks everything in that ns, you can use NotIn operator to ignore applications if you have multiple apps running in one ns and you only want to write netpols for one app. allow DNS egress in same netpol
2. check if your app has probes - if yes - allow kubelet ingress
3. check if your app needs to talk to kube-api server - to query or watch live cluster state - if yes - then allow egress to kube-apiserver
4. If you're firewalling a resource managed by a controller or operator, check whether the operator talks directly to the workload Pod itself (not just to the Kubernetes API about it). If yes, allow ingress from the operator.
5. then roll out in audit mode and enforce the policies.
6. now go check your hubble it will show you what all communication your app does - and then using that write a proper netpol.

## Anti-Pattern 7: toServices + toPorts, and Silent L7 No-ops

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
