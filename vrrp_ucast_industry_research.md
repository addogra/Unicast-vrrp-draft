# VRRP Unicast Industry Research for a Potential Informational RFC

Research date: 2026-04-11

Assumption: the request mentioned "keepvid"; this note interprets that as **Keepalived**, which is the widely used Linux VRRP implementation with documented unicast support.

## 1. Executive Summary

The standards baseline and the industry reality are now clearly split:

- **RFC 9568 VRRPv3 is explicitly multicast-based and LAN-scoped.**
- Multiple implementations already ship **unicast VRRP extensions**, but they are **not behaviorally identical**.
- The market has not converged on a single interoperable unicast design. Some implementations preserve most classic VRRP semantics, while others use "VRRP-like" master/backup election for L3 or cloud-only scenarios and relax gateway/virtual-MAC assumptions.

Because of that, the strongest near-term IETF deliverable is likely an **Informational RFC** that:

- documents deployment drivers,
- describes the implementation patterns already in use,
- catalogs protocol and operational deltas from RFC 9568,
- identifies the interop gaps that block a clean Standards Track extension today.

Trying to jump directly to a Standards Track unicast VRRP specification would be risky unless the draft first narrows the scope and picks one transport/behavior model.

## 2. Standards Baseline

### 2.1 What RFC 9568 assumes

RFC 9568 defines VRRPv3 as:

- operating **on a LAN**,
- sending protocol messages using **IPv4 or IPv6 multicast**,
- using the standard VRRP multicast destinations:
  - IPv4 `224.0.0.18`
  - IPv6 `ff02::12`
- requiring **TTL/Hop Limit = 255** on received packets,
- relying on a **single well-known virtual MAC** per virtual router.

This matters because many unicast implementations change at least one of these assumptions:

- multicast destination becomes peer-specific unicast,
- peers may be across L3 hops,
- source/destination validation changes,
- virtual-MAC behavior becomes optional, modified, or irrelevant,
- packet format and transport can change.

### 2.2 Current IETF management-model gap

The active VRRP YANG work (`draft-ietf-rtgwg-vrrp-rfc8347bis`) still models **VRRP version 2 and version 3**, but I did not find unicast-specific configuration nodes in the current draft material I checked. That strongly suggests that **industry unicast practice is ahead of the standardized management model**.

### 2.3 Deployment-driver nuance

The cloud motivation is real, but it should be described carefully.

- **Azure Virtual Network** explicitly states that **multicast and broadcast are not supported**, and that only unicast is supported inside virtual networks.
- **AWS** does support multicast, but via **Transit Gateway multicast**, not as a blanket assumption for every VPC design.

So the draft should avoid saying "public cloud does not support multicast" as a universal truth. A more accurate statement is that **many cloud and overlay environments either do not support multicast at all, or do not make it available in the simple on-link form that classic VRRP assumes**.

## 3. Industry Survey

## 3.1 Cisco IOS XR

### What Cisco documents

Cisco IOS XR documentation for NCS/ASR platforms explicitly documents **Unicast VRRP**. Cisco positions it as an answer for **cloud network scenarios** where paired routers need a VIP but the environment does not support classic L2 multicast or broadcast behavior.

### Key characteristics

- Unicast transport is configured with `unicast-peer`.
- When unicast is enabled, the router **does not send or receive multicast VRRP packets**.
- Cisco documents only **one `unicast-peer`**, so the model is effectively **two physical routers only**.
- The examples still configure a **virtual IP address**, so Cisco keeps the first-hop redundancy flavor rather than turning VRRP into a generic election-only tool.
- Cisco frames the feature as **Layer 3 unicast transport** for cloud use cases.

### Draft-relevant implications

- Cisco's model is close to "standard VRRP, but replace multicast with point-to-point unicast".
- It is **not** a general multi-peer mesh model.
- It keeps the **VIP semantics**, which is useful if an informational RFC wants a conservative, minimally invasive extension model.

## 3.2 Huawei

### What Huawei documents

Huawei explicitly documents **unicast VRRP** as a **proprietary protocol based on VRRPv2** for devices that need master/backup negotiation **across a Layer 3 network**.

### Key characteristics

- Configuration is centered on `vrrp vrid <id> peer-ip <peer> [source-ip <src>]`.
- Huawei recommends configuring a **source IP** to limit what packets are accepted.
- Huawei documents a configurable **UDP port** for unicast VRRP, with default **UDP 3077**.
- Huawei documents optional **MD5** and **HMAC-SHA256** authentication for unicast VRRP.
- Huawei states that unicast VRRP is for **two devices** on an L3 network.
- Huawei explicitly says this mode **does not provide gateway redundancy using virtual IP addresses** in the same way as common VRRP and does **not periodically send gratuitous ARP**.
- Huawei positions it heavily for **master/backup role negotiation**, including BRAS-like scenarios.

### Draft-relevant implications

- Huawei diverges much more strongly from RFC 9568 than Cisco does.
- The biggest difference is transport: this is not just "VRRP over unicast IP protocol 112"; Huawei documents a **UDP-ported unicast transport**.
- The second major difference is semantics: Huawei's mode is partly a **VRRP-derived role election mechanism**, not just a first-hop gateway redundancy mechanism.
- This is a strong argument for an informational draft to separate:
  - **unicast VRRP for gateway redundancy**, and
  - **VRRP-derived unicast master/backup election**.

## 3.3 Keepalived

### What Keepalived documents

Keepalived has mature and flexible unicast support through `unicast_peer`, plus multiple guardrails around source validation and hop/TTL handling.

### Key characteristics

- `unicast_peer { ... }` supports **multiple IPv4 and/or IPv6 peers**.
- `unicast_ttl` is configurable.
- Per-peer `min_ttl` / `max_ttl` can be configured.
- `check_unicast_src` and the global `vrrp_check_unicast_src` add source validation.
- `unicast_fault_no_peer` avoids silently falling back to multicast when no peers are present.
- Keepalived's `vrrp_strict` mode explicitly notes that **unicast peers are not allowed** under strict RFC compliance.

### Draft-relevant implications

- Keepalived is the clearest evidence that real deployments want **multi-peer unicast**, not just Cisco-style 2-node pairing.
- Keepalived also exposes a key standards point: the implementation itself treats unicast mode as **non-RFC-strict behavior**.
- The TTL range feature is operationally useful, but it also shows that vendors/implementations are solving the "TTL 255 on routed paths" problem in different ways.

## 3.4 VyOS

### What VyOS documents

VyOS documents **Unicast VRRP** and configures it with:

- `peer-address`
- `hello-source-address`

VyOS also documents an optional `rfc3768-compatibility` mode that creates a virtual interface and virtual MAC semantics closer to classic VRRP behavior.

### Draft-relevant implications

- VyOS confirms that unicast VRRP is no longer just a niche Linux knob; it is exposed as a first-class product feature.
- The `rfc3768-compatibility` option is especially interesting because it suggests one possible IETF framing:
  - a **transport extension** that tries to preserve classic VRRP forwarding semantics as much as possible.

## 3.5 Juniper Cloud-Native Router

### What Juniper documents

Juniper Cloud-Native Router documentation explicitly says **VRRP unicast** can be used in cloud deployments. In EKS/AWS deployments, Juniper ties VRRP mastership to **copying prefixes into AWS VPC route tables**, noting that AWS VPC supports exactly one next hop for a prefix in that workflow.

### Draft-relevant implications

- This is another major semantic shift: VRRP state is being used to drive **cloud route-table ownership**, not just on-link host default-gateway behavior.
- That makes unicast VRRP attractive in cloud-native routing systems, but it also means implementations are stretching VRRP into a broader **active/standby control-plane role selector**.
- An informational RFC should probably acknowledge this cloud pattern without redefining all of VRRP around it.

## 3.6 MikroTik RouterOS

### What MikroTik documents

Current RouterOS documentation describes VRRP in standard terms:

- multicast packets,
- TTL 255,
- virtual MAC,
- same-network / shared-segment assumptions.

I did **not** find documented unicast VRRP support in the RouterOS material I checked.

### Draft-relevant implication

- Not all vendors appear to be moving toward unicast VRRP.
- That matters for scope: a future standards effort should not assume that all existing VRRP stacks need or want the extension.

## 3.7 FRRouting (FRR)

### What FRR documents

FRR documents VRRPv2/VRRPv3 support with Linux macvlan-based virtual-MAC handling and explicitly describes classic segment-based VRRP behavior.

I did **not** find documented unicast VRRP support in the FRR documentation reviewed.

### Draft-relevant implication

- FRR reinforces that the open-source routing ecosystem is not uniformly aligned with the Keepalived unicast model.
- A standards effort that wants broad open-source implementation buy-in would need to justify the new mode clearly.

## 4. Support Matrix

| Implementation | Unicast support found | Peer model | Transport behavior | VIP/gateway semantics | Notable deviations |
|---|---|---|---|---|---|
| RFC 9568 baseline | No | Multi-router on LAN | IP proto 112 to multicast | Yes | Multicast + virtual MAC + LAN scope |
| Cisco IOS XR | Yes | 2 nodes | Unicast peer instead of multicast | Yes | One peer only; cloud-oriented |
| Huawei VRP | Yes | 2 nodes | Proprietary unicast mode; documented UDP port 3077 | Often no, role negotiation focus | Based on VRRPv2; configurable auth; no periodic GARP in this mode |
| Keepalived | Yes | Multi-peer | Unicast peer list; TTL controls | Yes, but flexible | Explicitly non-strict-RFC behavior |
| VyOS | Yes | Peer-based | Unicast peer/source config | Yes | Optional RFC3768 compatibility behavior |
| Juniper Cloud-Native Router | Yes | Active/backup cloud pair patterns | Unicast in cloud deployments | Used with route-table ownership workflows | VRRP state drives AWS route-table behavior |
| MikroTik RouterOS | No documented unicast found | Standard VRRP | Multicast | Yes | Stays close to standard model |
| FRRouting | No documented unicast found | Standard VRRP | Multicast | Yes | Linux/macvlan standard VRRP model |

## 5. Main Technical Fault Lines Across Implementations

These are the areas where implementations do not currently line up.

### 5.1 What problem is being solved?

There are at least **two different problems** being solved under the label "unicast VRRP":

1. **Classic first-hop redundancy without multicast**
   - Cisco, Keepalived, VyOS mostly fit here.
2. **Master/backup control-plane election across L3**
   - Huawei and some cloud-native uses fit here.

If one draft tries to cover both equally, it will get muddy very quickly.

### 5.2 Packet transport

The industry has not converged on one unicast transport model:

- "VRRP packet semantics but unicast delivery" style
- vendor-defined or implementation-defined additional transport choices
- Huawei's documented **UDP 3077** is the clearest sign that not everyone is preserving the exact RFC 9568 packet/transport assumptions

This is probably the single biggest blocker to immediate interoperability.

### 5.3 Topology scope

Implementations split between:

- **pairwise only** designs,
- **multi-peer** designs,
- LAN-adjacent semantics,
- routed L3 adjacency,
- cloud route-owner semantics.

### 5.4 Virtual MAC and neighbor behavior

Classic VRRP assumes a shared virtual MAC and on-link host behavior. Unicast implementations vary on:

- whether the virtual MAC is preserved,
- whether gratuitous ARP / unsolicited ND still behaves the same,
- whether the feature is even intended to protect a host-facing gateway.

Huawei is especially important here because its documentation explicitly says the mode differs from common VRRP gateway protection behavior.

### 5.5 Security model

RFC 9568 leans on on-link scoping plus TTL/Hop-Limit 255. Unicast across routed paths changes the threat model.

Industry responses differ:

- source-IP pinning,
- configurable peer allow-lists,
- min/max TTL or hop validation,
- MD5 / HMAC-SHA256 in Huawei,
- "strict RFC" modes that disallow unicast entirely in Keepalived.

This is a strong topic for the Security Considerations section of any future RFC.

### 5.6 Management and YANG

There is no obvious standardized management model for:

- peer lists,
- unicast source address,
- transport choice,
- TTL/hop filters,
- unicast-specific authentication,
- gateway-vs-election mode distinctions.

That is another reason an Informational RFC is the right first step.

## 6. What This Means for an Informational RFC

## 6.1 Recommended document positioning

The draft should position itself as:

- **Informational**
- **descriptive rather than prescriptive**
- focused on **deployment motivations, implementation patterns, and interop gaps**

That is safer than claiming a de facto standard where one does not yet exist.

## 6.2 Recommended problem statement

The cleanest problem statement is:

"Operators increasingly need VRRP-like first-hop or active/standby behavior in environments where multicast delivery is unavailable, undesirable, or operationally constrained, especially in routed overlays and cloud-connected topologies."

That wording is broad enough for industry reality but still honest about the fact that not every deployment is classic LAN first-hop redundancy.

## 6.3 Recommended scope boundaries

The document should probably:

- describe **existing unicast implementation families**,
- compare them against RFC 9568 assumptions,
- highlight areas where behavior is common enough to consider future standardization,
- avoid pretending that all existing implementations interoperate.

The document should probably **not**:

- claim wire interoperability across Cisco, Huawei, Keepalived, and cloud-native variants,
- redefine Huawei-style role-election semantics as equivalent to standard first-hop gateway VRRP,
- prescribe one mandatory transport unless the authors are ready to exclude part of today's deployed base.

## 7. Suggested Draft Structure

1. Introduction
2. Terminology
3. Why unicast VRRP exists
4. Standards baseline from RFC 9568
5. Survey of deployed implementation models
6. Transport and encapsulation differences
7. First-hop gateway semantics versus election-only semantics
8. Security considerations for routed/unicast deployments
9. Management and YANG gaps
10. Interoperability considerations
11. Considerations for possible future standardization

## 8. Best Candidate Topics for a Future Standards Track Follow-On

If the informational document gains traction, the cleanest follow-on standards work would likely be one of these:

### Option A: Minimal unicast transport extension to VRRPv3

- Keep protocol 112 and VRRP packet format
- replace multicast destination with configured unicast peers
- keep TTL/Hop Limit 255
- keep VIP and virtual-MAC semantics wherever feasible
- likely limit the initial standard to **two-node** operation

This is the most Cisco-like path.

### Option B: Multi-peer unicast VRRP profile

- explicitly standardize peer lists
- define source validation and TTL/hop rules
- define behavior when peers disagree on advert interval or peer inventory

This is closer to Keepalived/VyOS practice.

### Option C: Separate "VRRP-derived active/standby election" work

- do not overload classic first-hop VRRP semantics
- acknowledge Huawei/cloud-native style role-election use cases as a related but distinct problem

This may ultimately be cleaner than forcing every use case under one umbrella.

## 9. Bottom-Line Recommendation

For the draft you want to curate, the strongest message is:

- **there is real industry demand**, and
- **there is real deployment experience**, but
- **there is not yet one interoperable unicast VRRP behavior model**.

That makes an **Informational RFC** both useful and defensible right now.

The draft should focus on:

- documenting why operators need the feature,
- surveying how vendors implemented it,
- showing exactly where implementations differ,
- preparing the ground for later standards work once the WG decides which behavior family is worth standardizing.

## 10. Source Set

- RFC 9568: https://www.rfc-editor.org/rfc/rfc9568.html
- Current VRRP YANG work: https://datatracker.ietf.org/doc/html/draft-ietf-rtgwg-vrrp-rfc8347bis-15
- Cisco IOS XR VRRP guide (example release family): https://www.cisco.com/c/en/us/td/docs/routers/asr9000/software/asr9k-r7-9/ip-addresses/configuration/guide/b-ip-addresses-cg-asr9000-79x/implementing-vrrp.html
- Cisco IOS XR PDF excerpt with unicast rationale: https://www.cisco.com/c/en/us/td/docs/iosxr/ncs5500/ip-addresses/76x/b-ip-addresses-cg-ncs5500-76x/m-implementing-vrrp-ncs5000-ncs5500.pdf
- Huawei unicast VRRP overview: https://info.support.huawei.com/hedex/api/pages/EDOC1100363264/AEN0403J/05/resources/vrp/feature_vrrp_0025.html
- Huawei unicast VRRP command: https://info.support.huawei.com/hedex/api/pages/EDOC1100277644/AEM10221/04/resources/command/yunshan/UNICAST-VRRP%28VRRPOM%29.html
- Huawei unicast VRRP config procedure: https://info.support.huawei.com/hedex/api/pages/EDOC1100363264/AEN0403J/05/resources/vrp/dc_vrp_vrrp_cfg_0144.html
- Huawei unicast VRRP UDP port: https://info.support.huawei.com/hedex/api/pages/EDOC1100277644/AEM10221/03/resources/command/yunshan/UNIVRRPPORT%28VRRPOM%29.html
- Keepalived man page: https://www.keepalived.org/manpage.html
- VyOS high availability docs: https://docs.vyos.io/en/1.4/configuration/highavailability/
- Juniper Cloud-Native Router VRRP concept: https://www.juniper.net/documentation/us/en/software/cloud-native-router23.4/cloud-native-router-user/topics/concept/l3-vrrp.html
- Juniper Cloud-Native Router EKS/AWS VRRP ConfigMap: https://www.juniper.net/documentation/us/en/software/cloud-native-router25.4/cloud-native-router-deployment-guide/topics/concept/system-resource-requirements-eks.html
- MikroTik RouterOS VRRP docs: https://help.mikrotik.com/docs/spaces/ROS/pages/81362945/VRRP
- FRRouting VRRP docs: https://docs.frrouting.org/en/stable-7.5/vrrp.html
- Azure VNet FAQ: https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-faq#do-virtual-networks-support-multicast-or-broadcast
- AWS Transit Gateway multicast overview: https://docs.aws.amazon.com/vpc/latest/tgw/tgw-multicast-overview.html
