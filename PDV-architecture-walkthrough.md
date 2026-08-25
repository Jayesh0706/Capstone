# Principal Diagnosis Validation — Architecture Walkthrough

Review notes + speaking script for the solution architecture review.

---

## Part 1 — Correctness review

### One real error (fixed in v3)

**Front Door's private endpoint is in *our* VNet, not Microsoft's.**

The original diagram drew Front Door arriving directly into `snet-containerApps`, with a guarantee line saying the endpoint "sits in Microsoft's network, not ours." For a Container Apps origin that is wrong on both counts:

- You create a private endpoint for the **Container Apps environment**, and it takes an IP in your VNet.
- Traffic arrives at that private endpoint IP, which connects to the internal environment, which reaches the ingress controller via an internal load balancer VIP.
- **Private endpoints cannot live in a delegated subnet.** `snet-containerApps` is delegated to `Microsoft.App/environments`, so the endpoint has to go in `snet-privateEndpoints`. Microsoft's own troubleshooting table lists "private endpoint is not in a separate, nondelegated subnet" as the cause of endpoint creation failure.

v3 adds `pe-cae-env` at `10.10.4.3` as the first NIC in the private endpoint subnet, routes Front Door into it, and draws the hop up to the internal VIP.

This is the single thing most likely to get caught in review, and it's better that you raise it than that they do.

### The private-endpoints-vs-services fix (from v2, retained)

Cosmos DB, Blob, Document Intelligence, AI Foundry, Key Vault, and ACR are multi-tenant PaaS. They are **not in your VNet**. What is in `snet-privateEndpoints` is a NIC per service consuming a `10.10.4.x` address. v3 keeps them in a separate band below the VNet with Private Link lines connecting each pair.

### Everything that checks out

| Element | Verdict |
|---|---|
| Front Door **Premium** for Private Link origin | Correct — Standard tier does not support it |
| WAF policy attached to Front Door | Correct placement; geo-filtering belongs here |
| `/23` for the Container Apps subnet | Matches Microsoft's recommendation — room for replica and node scaling |
| Delegation to `Microsoft.App/environments` | Correct |
| `/24` nondelegated for private endpoints | Matches the recommended pattern |
| Internal VIP + `publicNetworkAccess: Disabled` on the environment | Correct, and required for this design |
| NAT Gateway for deterministic egress | Correct |
| Managed identity everywhere, no keys | Correct |
| Neo4j Aura over Private Link | Real capability — see the caveat below |

### Caveats to know before you're asked

**Workload profiles are mandatory.** Private endpoints on a Container Apps environment are a workload-profiles-only feature. If anyone on the team assumed a Consumption-only environment, that assumption breaks the whole inbound path.

**Front Door Private Link is only offered in regions with availability zone support.** East US qualifies. Worth a sentence if someone asks about region strategy.

**Managed identity origin authentication is not supported on Private Link origins.** If someone proposes Front Door authenticating to the origin with a managed identity, that combination doesn't exist. The origin is protected by network isolation plus the app's own Entra sign-in, not by origin auth.

**Don't mix public and private origins in one origin group.** Microsoft explicitly warns this causes configuration and traffic errors. Keep private origins in their own group.

**Front Door sometimes creates duplicate private endpoint connection requests** — a documented quirk. Approve each one matching your request message. Mention it as an operational note if you're asked about deployment gotchas.

**Neo4j Aura Private Link needs the Virtual Dedicated Cloud tier.** It also differs from the Azure services in three ways worth saying out loud: it's a manual-approval connection against a Private Link Service alias, you create and manage the private DNS zone yourself with a wildcard A record rather than getting an Azure-managed `privatelink.*` zone, and you must separately disable public traffic on the Aura side. Confirm your Aura tier before the meeting — this is the one dependency that could be a surprise.

**ACR needs two DNS records, not one.** `privatelink.azurecr.io` plus the regional data endpoint. Miss the second and image pulls fail in a way that looks like a registry outage.

### The honest gap: telemetry and identity are not private

Your original guarantee said "nothing traverses the public internet." That claim does not survive contact with a reviewer:

- **App Insights ingestion** goes to public regional endpoints unless you deploy an **Azure Monitor Private Link Scope (AMPLS)**. Right now that telemetry leaves via the NAT Gateway.
- **Entra ID** is a public endpoint by design. User sign-in goes browser → `login.microsoftonline.com` directly.

Neither is a security defect — both ride the Microsoft backbone with TLS and neither carries PHI if configured properly. But the *wording* was overclaiming. v3 narrows the guarantee to "every data dependency is reached through a private endpoint," which is true and defensible. Have AMPLS ready as your "phase two" answer.

### Gaps worth naming before someone else does

1. **Break-glass access.** With `publicNetworkAccess: Disabled` on Cosmos and Key Vault, the portal's data plane will not work from a laptop. You need a jump box, Bastion, or point-to-site VPN into the VNet. Nothing on the diagram provides this today.
2. **Single region.** East US only. No DR story on the page.
3. **PHI controls not shown.** Clinical coders and principal diagnosis means HIPAA. Customer-managed keys in Key Vault, audit logging, retention, and the BAA aren't represented. Fine to say "out of scope for the network view."
4. **NSGs are drawn but not specified.** The shields are there; the rules aren't. Container Apps subnets have required outbound rules that must be preserved.

---

## Part 2 — The script

Roughly 8–10 minutes. Speak in the order the traffic flows. Text in *italics* is stage direction, not something to read out.

### Opening (30 seconds)

> "This is the target architecture for Principal Diagnosis Validation. The design goal was one sentence: a clinical coder can reach this application from a browser, and nothing else can reach it from anywhere.
>
> I'll walk the request path top to bottom, then the dependency path, then the one thing that's deliberately public. The bottom panel is what I'm committing to, so hold me to it."

### 1. The user and the edge

*Point at the users icon, then Front Door.*

> "On the left, a clinical coder in a browser. They hit Azure Front Door over HTTPS 443.
>
> **What Front Door is:** Microsoft's global edge network. The user connects to whichever of its points of presence is nearest, and from there traffic rides the Microsoft backbone rather than the public internet.
>
> **Why it has to be here:** two reasons. First, it terminates TLS at the edge and gives us a single global entry point we control. Second — and this is the design-critical one — it's the only Azure ingress service that can reach a private origin over Private Link without us running an Application Gateway or exposing anything publicly. That's why it's **Premium** tier specifically. Standard cannot do private origins."

*Point at the WAF policy.*

> "**The WAF policy** attaches to the Front Door profile. It's where we enforce three things: geo-filtering down to cleared countries, the OWASP managed rule set, and rate limiting.
>
> **Why it belongs here and not deeper:** WAF at the edge means malicious traffic is dropped before it ever reaches Azure's regional network, let alone our subscription. Filtering after traffic has entered our VNet would mean we're paying to receive attacks we're about to discard."

### 2. The inbound private path — *the part they'll test you on*

*Trace the line down the left side into the private endpoint subnet.*

> "Now the part that's easy to draw wrong, so let me be precise. There are **two** private endpoints forming this tunnel, and I want to name both.
>
> **Microsoft's side:** when we enable a Private Link origin, Front Door assigns a dedicated private endpoint inside its own managed virtual network. We don't provision it, we don't pay for it directly, and it doesn't touch our address space. We just approve the connection request.
>
> **Our side:** that endpoint connects over the Microsoft backbone to a private endpoint **we** own, at `10.10.4.3` in our private endpoint subnet. This one is a real resource in our subscription — our IP, our NSG, our cost line.
>
> So Front Door does **not** connect to our container apps over the internet, and it does **not** connect directly into the application subnet. Traffic lands on our endpoint first.
>
> **What a private endpoint actually is:** a network interface. A NIC holding one IP address that terminates a Private Link tunnel. It is not the service. The service lives elsewhere; the NIC is one end of a private tunnel to it.
>
> **Why this endpoint is in the private endpoint subnet and not the app subnet:** the app subnet is *delegated* to `Microsoft.App/environments` — we've handed control of it to the Container Apps service. Azure will not let a private endpoint be created in a delegated subnet. So it has to live in a separate, nondelegated subnet. That's not a preference, it's a platform constraint, and it's why we have two subnets rather than one."

*Trace the arrow up into the environment.*

> "From that endpoint, traffic reaches the Container Apps environment's **internal load balancer VIP** — an address inside the app subnet — and the environment's ingress controller routes it to the right app based on host header.
>
> The environment itself has **public network access disabled** and an **internal** virtual IP. That means even if somebody found the environment's default domain and resolved it, there's nothing listening on a public address. The isolation is a configuration state on the resource, not a firewall rule we hope is correct."

### 3. Inside the environment

*Point at the two container apps.*

> "**`ca-frontend`** is the Flask UI. It handles Entra ID sign-in, so the user is authenticated as a person before they see anything.
>
> **`ca-backend`** runs with **internal ingress only**. That's a setting on the container app: it's addressable from inside the environment and from nowhere else. There is no route to it from the internet — not a blocked route, no route.
>
> **Why split them at all:** it keeps the thing handling untrusted browser input separate from the thing that talks to the clinical data. If the frontend is compromised, the attacker still has to present a valid token to get anything out of the backend.
>
> The frontend calls the backend on its internal FQDN, carrying an **Entra token**. So the hop between our own two services is authenticated too. We don't treat 'inside the VNet' as 'trusted'."

### 4. The dependency path

*Point at the blue arrow going down into the private endpoint subnet.*

> "Now the outbound side. When the backend needs Cosmos, or Document Intelligence, or a model in AI Foundry, it makes an ordinary HTTPS call to the normal service hostname — nothing exotic in the application code.
>
> Two mechanisms make that call private, and this is the part worth understanding properly."

*Point at the private DNS zone box.*

> "**First, DNS.** We have private DNS zones linked to the VNet — `privatelink.documents.azure.com`, `privatelink.blob.core.windows.net`, and so on. When the app resolves the Cosmos hostname, it doesn't get Cosmos's public address. It gets `10.10.4.4` — the private endpoint's address. Same code, same hostname, different answer.
>
> This is the most fragile piece of the design and the piece that gets missed. Private endpoints without correct DNS linkage look fine in the portal and fail at runtime. A note for the runbook: ACR needs *two* records, the registry endpoint and the regional data endpoint — miss the second and image pulls fail while everything else looks healthy."

*Point at the gray band at the bottom.*

> "**Second, the services themselves.** And here's the honest framing that I want to correct from earlier versions of this diagram.
>
> These services are **not inside our VNet**. Cosmos DB, Blob Storage, Document Intelligence, AI Foundry, Key Vault, and the Container Registry are multi-tenant services running in Microsoft's fabric. You cannot put them in a VNet — that's not what they are.
>
> What we've done is put a NIC for each one in our subnet, and turn `publicNetworkAccess` to `Disabled` on every one of them. So the only path to any of these services is the tunnel that terminates on our NIC, and the tunnel runs on the Microsoft backbone. The service has no public front door left open.
>
> I've drawn them outside the VNet boundary deliberately, because the shortcut version — services drawn inside the subnet — implies a network property we don't actually have, and it's the first thing a network reviewer will challenge."

*Point at Neo4j.*

> "**Neo4j Aura** works the same way in principle but differs in three practical ways. It's a partner service outside Azure entirely, so it's a Private Link **Service** connection requiring manual approval on the Neo4j side. We create and manage its DNS zone ourselves rather than getting an Azure-managed one. And it requires their Virtual Dedicated Cloud tier — so it's the one dependency where the commercial arrangement is load-bearing on the architecture."

### 5. Egress

*Point at the NAT Gateway.*

> "**The NAT Gateway** is associated with the container apps subnet. Note the word — associated, not routed. There's no user-defined route here; attaching a NAT Gateway to a subnet makes it the outbound path for that subnet by default.
>
> **What it does:** anything genuinely internet-bound leaves through it and appears as one static IP.
>
> **What it doesn't do:** it does not touch private endpoint traffic. Cosmos, Key Vault, all of it stays internal and never sees the NAT Gateway.
>
> **Why we want it:** without it, outbound traffic uses ephemeral platform addresses that rotate. With it, we have one attributable address. Partners can allowlist us. Our egress shows up as a single identifiable source in anyone's logs, including ours.
>
> This is the only public IP we own anywhere in the design. That's the sentence I'd like you to remember from this slide."

### 6. The cross-cutting services

*Point at Entra and Monitor.*

> "Two things on the right, and I want to be straight about their network position rather than let the dotted lines imply something flattering.
>
> **Entra ID** issues the managed identities our apps use to authenticate to every dependency. This is why there are no keys or connection strings anywhere in this design — nothing to leak, nothing to rotate, nothing to find in a config file. It's also a **public endpoint by design**. User sign-in is the browser talking to Microsoft directly. That's normal and correct, and I'm calling it out so nobody thinks I'm claiming otherwise.
>
> **Azure Monitor and App Insights** collect telemetry. Today that ingestion also goes to a public regional endpoint, out through the NAT Gateway. If we want that path private too, the mechanism is an Azure Monitor Private Link Scope, and I'd propose that as a phase-two item rather than a blocker.
>
> So the precise claim is: **every data dependency is private.** Not 'no packet ever touches a public endpoint.' The second version would be a nicer sentence and it wouldn't be true."

### 7. Closing on the guarantees

*Point at the bottom panel.*

> "Six commitments, and each one maps to something specific on this page rather than to a policy document:
>
> 1. **The NAT Gateway holds the only public IP we own.** Everything else is private-addressed.
> 2. **Inbound lands on a private endpoint inside our VNet** — and the environment has public access disabled, so there's no second way in.
> 3. **The backend is internal-ingress only** — unreachable by configuration, not convention. Nobody can misconfigure a rule and expose it, because there is no rule; there's no path.
> 4. **Every data dependency goes through a private endpoint**, with the public endpoint switched off at each service.
> 5. **Private DNS resolves every service name to a `10.10.4.x` address** — there's no public resolution path for the app to accidentally take.
> 6. **Identity-based auth end to end** — no secrets exist to leak.
>
> The things I know are still open: break-glass admin access needs a Bastion or jump box, because with public access disabled nobody can reach Cosmos or Key Vault data planes from a laptop. We're single-region. And AMPLS for telemetry. I'd rather flag those now than have them found."

---

## Part 3 — Anticipated questions

**"Why not Application Gateway instead of Front Door?"**
App Gateway is regional and would need its own public IP and a WAF in our VNet. Front Door Premium gives us global edge, WAF at the edge, and Private Link to the origin. The classic Front Door → App Gateway → app stack exists specifically to keep backends private; Private Link origin support removes the need for the middle hop.

**"Why two subnets? Why not one?"**
Not a choice. `snet-containerApps` is delegated to `Microsoft.App/environments`, and Azure won't create private endpoints in a delegated subnet. `/23` for the apps gives replica and node headroom; `/24` for endpoints is far more than the eight we need.

**"What stops someone reaching Cosmos directly?"**
`publicNetworkAccess: Disabled` on the account. The public endpoint doesn't accept connections. Then RBAC via managed identity on top. Two independent layers — network and identity.

**"What if the private DNS zone is misconfigured?"**
The app resolves the public hostname, gets a public IP, connects, and gets refused because public access is disabled. It fails closed. Annoying to debug, but it fails in the safe direction.

**"Can Front Door authenticate to the origin?"**
Not with managed identity on a Private Link origin — that combination isn't supported. Origin protection here is network isolation plus the app's own Entra sign-in.

**"I thought Front Door's private endpoint was on Microsoft's side — why is one in our subnet?"**
Both are true. Front Door assigns a dedicated endpoint in its own managed VNet; that connects over the backbone to the endpoint we own at `10.10.4.3`. Only ours consumes our address space, which is why only ours appears on the diagram as a resource. And ours has to sit in the nondelegated subnet, because Azure won't create a private endpoint in a subnet delegated to `Microsoft.App/environments`.

**"Is the Front Door endpoint hostname discoverable?"**
Yes, and it's supposed to be — that's the front door. WAF geo-filtering, Entra sign-in, and rate limiting are what sit behind it.

**"How do we operate this? I can't reach Key Vault from the portal."**
Correct, and that's a real gap on this page. We need Bastion or a point-to-site VPN into the VNet for break-glass. I'd like to scope that today.

**"Cost?"**
Front Door Premium is the significant line item, plus per-endpoint charges for private endpoints on both the Consumption and Dedicated plans. Roughly eight endpoints here. Happy to bring numbers back.
