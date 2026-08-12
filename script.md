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

### Slide 6 - Section divider

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

## 4: toFQDNs Without a DNS Proxy Rule

The next one is to add a toFQDNs without added l7 DNS rules. If you add a FQDNs egress to your pod without a rules.dns block allowing port 53, nothing permits DNS traffic at all. What internally happens is:

- pod would call a DNS lookup for example.org to CoreDNS at port 53. If you dont even allow egress to CoreDNS - this lookup never happens and your `toFQDNs` wont do anything. 
- if you allow traffic from your pod to CoreDNS, This packet hits the Pod's veth - eBPF checks the policy: is traffic allowed to coreDNS - it directly routes it to coreDNS.
- now this is imp, if you just wrtie a normal netpol without any l7 dns policy i.e. `rules.dns` cilium's eBPF cant do the DNS parsing itself (why ? because it can look at application level traffic only). so it will simply route it to CoreDNS and then CoreDNS answers directly back to the Pod the IP. since Cilium was never in this conversation at all, so nothing read this response, nothing got recorded anywhere. The Pod knows the IP now; Cilium does not.
    - Pod tries to connect to 93.184.216.34:443.
    - eBPF checks IPcache: is there a toFQDNs entry allowing this Pod to reach example.org? - No.
    - No matching policy map entry for this IP → blocked.
- only if you add the `rules.dns`,  it rewrites the packet's destination so that instead of going straight to CoreDNS, it goes to a local socket that the DNS proxy process is listening on, inside cilium-agent, on that same node.

- So we need to add l7 dns rule in the policy, if we add that, eBPF intercepts the packet, sees a L7 policy, it rewrites the packet so the kernel delivers it to the local socket where DNS proxy process is listening on, inside cilium-agent, on this same node - instead of CoreDNS.
- The DNS proxy process receives the packet on that socket. it reads the DNS query. The proxy now makes its own separate outbound connection to the CoreDNS, forwarding the same query on the Pod's behalf.
- CoreDNS answers, sending the response back to the proxy. The proxy reads this response - parses it - and writes this down in two places:
    - This Pod's own FQDN cache - "example.org → 93.184.216.34, valid until TTL expiry."
    - The IPCache (which is a the shared, cluster wide table)
- The proxy forwards the real DNS answer back to the Pod as well.
- now when pod tries to connect to exmaple.com - This new packet leaves the Pod, hits the veth, same eBPF hook again. now Pod's identity allowed to reach 93.184.216.34 on port 443? Yes - Allowed.

so the fix is to make sure a rules.dns block covering port 53 actually applies to every Pod that has a toFQDNs rule.

### Anti-Pattern 5: Default-Deny Surprises

when you apply  a CiliumNetworkPolicy that selects a Pod and you havent added any policy denying traffic - it automatically converts it to default deny, unless explicitly allowed. this happens per direction - ingress and egress - independently.

So never start with writing netpols for an app and just enforcing them, follow this step by step process - this always works

1. write a default deny policy that blocks everything in that ns, you can use NotIn operator to ignore applications if you have multiple apps running in one ns and you only want to write netpols for one app. allow DNS egress in same netpol
2. check if your app has probes - if yes - allow kubelet ingress
3. check if your app needs to talk to kube-api server - to query or watch live cluster state - if yes - then allow egress to kube-apiserver
4. If you're firewalling a resource managed by a controller or operator, check whether the operator talks directly to the workload Pod itself (not just to the Kubernetes API about it). If yes, allow ingress from the operator. In our case, someone didnt not add ingress from cnpg operator for a cnpg pod, now it was working fine, but then month or two later when cnpg pod was restarted - it could work - thats because cnpg operator talks to the cnpg pod at the startup - so these mistakes remain hindden sommetimes, thats one more good reason to be careful while writing netpols, and not just asking claude to one shot it, infact claude from our experience makes a lot of mistakes for network pollicies.
5. then roll out in audit mode and enforce the policies.
6. now go check your hubble it will show you what all communication your app does - and then using that write a proper netpol.

### Anti pattern 6: lb

If your pod is internet facing via a LoadBalancer - that doesn't mean traffic hitting it is already authorized since its a internet facing service or your loadbalancer service's DNAT will take care of it.

Traffic arrives from outside of cluster - gets DNAT'd by the loadbalancer Service to the Pod's IP. policy engine sits at the pod's veth so which it checks for policy for the packets, the Service is already out of the picture - all policy sees is an identity - that packter is coming from outside world to pod's IP.
Since the policy didne explicitly allow traffic from outside cluster - it blocks the traffic. so you will have to add a to a `fromEntities: [world]` explicitly in such cases. 

---

### Anti Pattern 7 - Expecting Instant Enforcement on Label Changes

"Incident-response instinct: relabel a compromised or misbehaving pod - say, slap `role: quarantined` on it - expecting that to immediately sever its existing connections to sensitive backends. That's not guaranteed.

Changing a pod's labels recomputes its Cilium identity, and that does trigger the endpoint to regenerate its policy. But already-established connections - think conntrack-tracked flows - aren't necessarily re-evaluated against the new policy instantly; enforcement is largely applied at connection-setup time. I want to be careful here - exact behavior is version- and configuration-dependent, so I'm not stating this as an absolute law, but as a real gap between what people *assume* relabeling does and what it's guaranteed to do.

Practical incident-response takeaway: if you need a hard, immediate cut, don't rely on a relabel alone - kill the pod or the connection directly."

### Anti-Pattern 8: Death by a Thousand NetPols

now this isn't a technical bug. It's an organizational iisue, and you can't fix it with better manifests.

your namespaces would not be one app per namespace. A shared namespace like `checkout` might run multiple services. Every service owner would add or update their own CiliumNetworkPolicy whenever they need a new path open. there is no shared owner, no central place where your netpols live - its scattered across projects or helm charts. nothing forces them to coordinate, because policies only ever compose additively. nothing breaks for now,  keep piling up.

but when something breaks, you have a pile of netpols in what not places and debugging is very difficult - becuase you know what traffic is blolcked - via hubble - you cant find the netpols or in some cases set of netpols causeing that. because the effective policy is the union of every rule from every policy that happens to select it.

The fix is centralizing policy under one place. but then theres another trap - If you generate a policy per app and each of those includes its own default-deny, you get several default-denies in a namespace, each written from the perspective of one app, each unaware of the others. That causes another problem.

- So this is how we fix this with our netpols, we have a centralized helm chart called KubeAid Addons , this is the place for all network policies we have (infact we have centralized our operator configs as well). For us and our open source users, so now network policies for all apps live in one chart, so it can be easily read, understood, debuggged or given to claude to debug, and these netpols rendered conditionally per namespace, one targeted policy per app. We make have put the netpols as a dependency chart while the default deny sits as part of main chart so that we can have one targeted policy per app, and exactly one default-deny per namespace - and thats the rule to follow. 

- so if a application level firewalling that is centralized and follows best pratices and avoids the mistakes we listed - you can checkout kubeaid-adons its open sourced and have firewalls for good number of open source applications SREs and platform engineers use. otherwise you can always follow all the rules I mentioned, or if you want to write it yourself you can at least refer those netpols - maybe copy paste if the netpols you need are already there. 

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

