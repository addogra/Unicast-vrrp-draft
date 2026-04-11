# Lessons From Unicast VRRP Research and Drafting

Date: 2026-04-11

This note captures the main lessons from the unicast VRRP research, the industry survey, and the work to turn that material into Internet-Drafts.

## 1. The biggest lesson: "unicast VRRP" is real, but it is not one thing

Multiple vendors and open-source projects already support something they call unicast VRRP, but they do not all mean the same thing.

There are at least two distinct models in the industry:

- Classic first-hop redundancy without multicast
- VRRP-derived active/backup role election across L3 or in cloud control-plane workflows

This distinction matters a lot. A standards-track draft cannot safely standardize every deployed behavior under one umbrella.

## 2. Why an Informational draft made sense first

The industry survey showed enough deployment reality to justify documenting the space, but not enough convergence to jump directly into a broad interoperable standard.

The main differences across implementations are:

- transport choice,
- topology assumptions,
- pairwise versus multi-peer behavior,
- security validation,
- virtual MAC and host-facing behavior,
- whether the feature is still really about a virtual default gateway.

That is why the first draft was framed as an overview / deployment-experience document, and the second draft was intentionally narrow.

## 3. The right narrow scope for a protocol draft

The cleanest standards path is a conservative unicast extension that stays close to RFC 9568:

- keep the VRRP packet format,
- keep IP protocol 112,
- keep virtual IP semantics,
- keep TTL / Hop Limit 255 validation,
- keep the standard VRRP virtual MAC behavior,
- replace multicast delivery with configured unicast peers,
- require explicit peer-list source validation,
- do not try to cover multi-hop routed election or vendor-proprietary UDP-based designs.

This is the safest way to define "unicast VRRP" without breaking the mental model of VRRP itself.

## 4. The MAC address lesson is subtle but important

Classic VRRP has two different MAC-related behaviors:

- VRRP advertisements are sent to a multicast destination.
- The VRRP virtual router MAC used as the host-facing gateway MAC is a unicast virtual MAC.

For standard VRRP that means:

- IPv4 advertisements use `224.0.0.18` and Ethernet multicast MAC `01:00:5e:00:00:12`
- IPv6 advertisements use `ff02::12` and Ethernet multicast MAC `33:33:00:00:00:12`
- The VRRP virtual gateway MAC remains unicast:
  - IPv4: `00:00:5e:00:01:<VRID>`
  - IPv6: `00:00:5e:00:02:<VRID>`

The key takeaway for unicast VRRP is this:

- the control-packet delivery method may change,
- the virtual gateway MAC does not need to become multicast,
- the standards-track draft should explicitly say that unicast mode changes advertisement delivery, not the Virtual Router MAC definition.

## 5. What we learned from vendors

### Cisco IOS XR

Cisco IOS XR documents unicast VRRP with `unicast-peer`.

Main lessons:

- It stays close to classic VRRP semantics.
- It is effectively a two-node model.
- It keeps virtual IP behavior.
- Published operational output still shows the standard VRRP virtual MAC while unicast transport is enabled.

This is strong evidence for the "transport-only change" model.

Important nuance:

- We found strong public IOS XR documentation.
- We did not independently confirm XRd-specific public documentation, so XRd should not be claimed unless separately verified.

### Keepalived

Keepalived is one of the strongest examples of mature unicast VRRP support.

Main lessons:

- It supports explicit `unicast_peer` lists.
- It supports source checking and TTL controls.
- It treats unicast as non-strict-RFC behavior.
- It supports VMAC usage, but when `use_vmac` is combined with unicast peers it requires `vmac_xmit_base`.

The design lesson is important:

- Keepalived does not redefine the VRRP virtual MAC for unicast mode.
- Instead, it may change which interface sends and receives the control traffic.

### VyOS

VyOS exposes unicast VRRP as a product feature with peer-based configuration.

Main lessons:

- It supports `peer-address` and `hello-source-address`.
- It also has `rfc3768-compatibility`.
- That compatibility mode creates a VRRP interface and assigns the virtual MAC and virtual IP.

This suggests that some implementations see "classic VRRP VMAC/interface behavior" as something that may need to be made explicit in unicast deployments.

### Huawei VRP

Huawei documents unicast VRRP, but it is much more divergent from RFC 9568.

Main lessons:

- It is documented as a proprietary VRRPv2-based unicast mechanism.
- It uses a configurable UDP port, with public docs pointing to UDP 3077.
- It supports additional authentication mechanisms.
- It is framed for L3 master/backup negotiation across a routed network.
- It does not align cleanly with classic virtual gateway semantics.
- Huawei documentation also states that this mode does not periodically send gratuitous ARP in the same way as common VRRP.

This is a strong reason not to over-generalize all deployed unicast VRRP under one protocol model.

### Juniper Cloud-Native Router

Juniper shows that cloud deployments use VRRP unicast for more than just on-link first-hop gateway protection.

Main lessons:

- VRRP state can be tied to route-table ownership.
- In those deployments, the active router role drives cloud route programming.

This is a real deployment driver, but it stretches VRRP toward broader active/standby control-plane use.

### FRRouting

FRR was useful as a negative control.

Main lessons:

- We did not find public unicast VRRP support in the FRR docs we reviewed.
- The code path we inspected remains aligned with classic multicast VRRP.
- FRR still programs the standard RFC virtual MAC and sends advertisements to the VRRP multicast group.

That means FRR currently supports the classic model, not the unicast extension.

### MikroTik RouterOS

We did not find documented unicast VRRP support in the RouterOS material reviewed.

That matters because it shows not all vendors are moving in the same direction.

## 6. Code-backed lessons from Keepalived and FRR

### Keepalived source findings

From the reviewed source:

- the default VMAC is still built from the standard VRRP pattern,
- IPv4 VMAC remains `00:00:5e:00:01:<VRID>`,
- IPv6 VMAC remains `00:00:5e:00:02:<VRID>`,
- `use_vmac` with unicast peers forces `vmac_xmit_base`,
- unicast advertisements are sent per peer,
- the interesting change is transmit/receive path selection, not redefinition of the VRRP virtual MAC.

This is one of the most useful implementation lessons from the session.

### FRR source findings

From the reviewed source:

- `vrrp_mac_set()` still programs the RFC VMAC values,
- the router object stores that VMAC in the normal way,
- `vrrp_send_advertisement()` still sends to the IPv4 or IPv6 VRRP multicast group,
- we did not find a unicast-peer transport path in the code reviewed.

So FRR currently reinforces the standard multicast model rather than the unicast extension.

## 7. Cloud language must be precise

One wording trap came up early and is worth preserving for others:

- Do not say "public cloud does not support multicast" as a universal statement.

The more accurate lesson is:

- many cloud, overlay, and virtualized environments do not support multicast at all, or do not expose it in the simple on-link form that classic VRRP expects.

This matters because broad cloud claims are easy for reviewers to challenge.

## 8. Draft-writing lessons

A few drafting lessons became clear while iterating:

- Put the use-case motivation into the introduction in a vendor-agnostic way.
- Keep the protocol draft narrowly scoped.
- Use an `Implementation Status` section to show why the work matters.
- Be explicit that the standards-track proposal is not trying to capture every vendor-specific behavior.
- State clearly that unicast mode changes advertisement delivery, not the definition of the VRRP Virtual Router MAC.

That last point is especially important because readers can easily confuse:

- multicast destination addressing for control packets, and
- the unicast virtual MAC used by hosts for forwarding.

## 9. The cleanest standards message

If this work is socialized with others, the cleanest message is:

"Industry already uses unicast VRRP, but implementations diverge. The right first standardization target is a narrow mode that preserves RFC 9568 VRRP behavior and changes only advertisement delivery and peer validation."

That message is both technically defensible and easy for reviewers to reason about.

## 10. Suggested talking points for sharing

- Unicast VRRP is already deployed in the industry.
- The industry has not converged on one interoperable model.
- The biggest design split is between gateway redundancy and generic active/backup election.
- The VRRP virtual router MAC remains a unicast MAC in the classic model.
- A unicast extension should not redefine that MAC behavior.
- Keepalived and Cisco IOS XR are strong examples of "preserve VRRP, change transport".
- Huawei is a strong example of why scope must be controlled.
- FRR is a useful example of the current multicast-only baseline.

## 11. Files produced in this work

- `vrrp_ucast_industry_research.md`
- `draft-abinabraham-vrrp-unicast-overview-00.xml`
- `draft-abinabraham-vrrp-unicast-00.xml`

## 12. Recommended next step

The best next step is to keep sharpening the protocol draft as a narrow update to RFC 9568, while keeping the overview draft available as supporting evidence for why the narrower specification is justified.
