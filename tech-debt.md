# Tech Debt for `draft-abinabraham-vrrp-unicast`

Date: 2026-04-26

## Current Situation

The unicast VRRP work is no longer just local research. It is already published as an Internet-Draft:

- [draft-abinabraham-vrrp-unicast-00](https://datatracker.ietf.org/doc/draft-abinabraham-vrrp-unicast/)

As of 2026-04-26, the published draft is:

- an **active individual Internet-Draft**,
- currently framed as **Informational** in the rendered draft text,
- already written as an update-oriented document for **RFC 9568**,
- already scoped around a narrow unicast mode that preserves the existing VRRP packet format and gateway semantics.

That means the work is no longer "start a draft". The tech debt is now about the **next revision**.

## The Two Real Gaps for the Next Version

There are two concrete gaps to close in the upcoming revision sequence:

1. **Protocol gap**:
   take the already-published unicast VRRP draft and evolve it from an informational individual draft into a **Standards Track update to RFC 9568**.
2. **YANG gap**:
   extend the VRRP YANG work in [draft-ietf-rtgwg-vrrp-rfc8347bis](https://datatracker.ietf.org/doc/draft-ietf-rtgwg-vrrp-rfc8347bis/) so the standardized unicast mode has a corresponding interoperable configuration and state model.

Everything else is secondary to those two items.

## Gap 1: Convert the Published Unicast VRRP Draft into a Standards Track RFC 9568 Update

### Why this is a gap

The published draft already makes the core protocol argument:

- multicast VRRP is the existing default behavior,
- some environments cannot use multicast for VRRP advertisements,
- operators still want standard VRRP gateway semantics,
- unicast mode can be introduced without changing the VRRP packet format.

However, the published draft is still positioned as an informational document. The next version needs to become a proper **Standards Track** document so that RFC 9568 explicitly supports **two modes**:

- `multicast` mode, which remains the default,
- `unicast` mode, which is enabled by explicit configuration.

### Intended protocol shape

The standards-track version should keep the current narrow approach:

- **No packet format change**
- **No new VRRP protocol number**
- **No change to virtual IP semantics**
- **No change to the virtual router MAC behavior**
- **Default mode remains multicast**
- **Unicast mode is selected explicitly by configuration**
- **The only normative change is advertisement delivery and peer-based validation**

### Why this direction is justified

The external documentation already shows that this exact operational shape exists in practice.

Publicly documented implementations that support the narrow model include:

- Cisco IOS XR:
  [Cisco IOS XR VRRP](https://www.cisco.com/c/en/us/td/docs/routers/asr9000/software/asr9k-r7-9/ip-addresses/configuration/guide/b-ip-addresses-cg-asr9000-79x/implementing-vrrp.html)
- Juniper Cloud-Native Router:
  [Juniper VRRP](https://www.juniper.net/documentation/us/en/software/cloud-native-router23.4/cloud-native-router-user/topics/concept/l3-vrrp.html)
- Keepalived:
  [Keepalived manpage](https://www.keepalived.org/manpage.html)
- VyOS:
  [VyOS High Availability](https://docs.vyos.io/en/latest/configuration/highavailability/)

These implementations are not identical, but they provide enough evidence that a conservative, interoperable unicast mode is worth standardizing.

### What the next draft revision needs to do

The next published revision should:

1. Change the draft framing from informational to **Standards Track**.
2. Present the document explicitly as a **normative update to RFC 9568**.
3. State clearly that VRRP now has **two modes of operation**:
   multicast by default, unicast when configured.
4. Keep the protocol delta intentionally narrow:
   advertisement destination changes from multicast group to configured unicast peers, while the rest of the VRRP model stays intact.
5. Tighten the standards language around:
   peer configuration, source validation, IPv4/IPv6 destination rules, and interoperability expectations.

### What must remain out of scope

The first standards-track version should not try to absorb every deployed "unicast VRRP" behavior.

Still out of scope:

- Huawei-style UDP-based transport variants,
- generic multi-hop routed active/backup election,
- cloud route-table ownership workflows as the primary protocol model,
- broad conversion of VRRP into a generic master/backup framework.

## Gap 2: Add the YANG Model Changes Based on `draft-ietf-rtgwg-vrrp-rfc8347bis`

### Why this is a separate gap

Even if the protocol draft becomes standards-track, the work will still be incomplete unless there is a standard YANG representation for unicast mode.

I reviewed:

- [draft-ietf-rtgwg-vrrp-rfc8347bis-15](https://datatracker.ietf.org/doc/draft-ietf-rtgwg-vrrp-rfc8347bis/)

The current YANG draft already models VRRP instances under the IPv4 and IPv6 interface trees, but it does **not** model unicast mode today.

### What the current YANG draft already provides

The current YANG draft already gives a strong base structure:

- it augments the IPv4 and IPv6 interface trees with a `vrrp` container,
- it models `vrrp-instance` lists keyed by `vrid`,
- it already contains the baseline per-instance attributes and state,
- it already defines protocol/global and virtual-router error reporting,
- it already includes VRRP notifications such as:
  - `vrrp-new-active-event`
  - `vrrp-protocol-error-event`
  - `vrrp-virtual-router-error-event`

This is important because it means we should **extend the existing YANG model**, not create a separate unicast-VRRP model.

### What is missing in the YANG draft

The current YANG draft does not currently expose:

- a mode selector for `multicast` versus `unicast`,
- a configured unicast peer list,
- address-family-aware peer typing,
- unicast-specific operational state,
- peer-validation-specific counters or notifications.

The word `unicast` does not currently appear in the draft, which is a simple but strong signal that this part is still absent.

### What the YANG update should add

The YANG follow-up should add unicast support in the existing per-instance model.

Minimum needed additions:

1. **Mode selection**
   A per-instance leaf or equivalent enumeration:
   - `multicast` as default
   - `unicast` as explicit configured alternative

2. **Unicast peer list**
   A per-instance unicast peer container/list:
   - mandatory when mode is `unicast`
   - keyed by peer address
   - IPv4 peers for IPv4 instances
   - IPv6 link-local peers for IPv6 instances

3. **Operational visibility**
   State that allows an operator to see:
   - effective mode,
   - configured peers,
   - whether advertisements are being rejected because the source is not a configured peer

4. **Error/statistics support**
   Counters and/or event identities for:
   - packets dropped because the source is not a configured peer,
   - unicast-specific consistency failures

### YANG design principles for this work

The YANG update should follow the same narrow philosophy as the protocol draft:

- extend the current VRRP instance tree,
- keep multicast as the default,
- make unicast peer configuration explicit,
- reflect peer validation as normative behavior of unicast mode,
- avoid vendor-specific transport knobs in the first revision.

### What should not go into the first YANG revision

Do not model vendor-specific extras in revision 1:

- proprietary UDP transport selection,
- vendor-specific authentication controls for non-standard unicast variants,
- implementation-specific TTL range knobs,
- cloud-specific route-table programming extensions,
- generic multi-hop election controls.

## Relationship Between the Two Gaps

These two gaps are related, but they should stay distinct in planning:

- **Gap 1** is the protocol standards-track gap.
- **Gap 2** is the management-model gap.

The standards-track protocol text can move first, but the work is not really complete until the YANG model also reflects the standardized unicast mode.

## Repo Material That Already Supports This Work

The following files already contain most of the input needed for these next steps:

- [draft-abinabraham-vrrp-unicast-00.xml](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/draft-abinabraham-vrrp-unicast-00.xml)
- [draft-abinabraham-vrrp-unicast-deployment-00.xml](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/draft-abinabraham-vrrp-unicast-deployment-00.xml)
- [vrrp_ucast_industry_research.md](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/vrrp_ucast_industry_research.md)
- [lessons.md](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/lessons.md)

## Exit Criteria

This tech debt is paid down when:

- the next revision of `draft-abinabraham-vrrp-unicast` is repositioned as a **Standards Track** RFC 9568 update,
- that draft explicitly standardizes the **two-mode model**:
  multicast by default, unicast when configured,
- the protocol remains packet-format compatible with existing VRRP,
- the corresponding YANG work extends `draft-ietf-rtgwg-vrrp-rfc8347bis` with mode selection, peer configuration, and unicast-specific observability,
- the standardized model remains narrow enough to map cleanly to Cisco/Juniper/Keepalived/VyOS-like deployments without pulling in broader vendor-proprietary behaviors.

## One-Line Summary

The next-phase tech debt is now very clear: **promote the already-published unicast VRRP draft into a Standards Track RFC 9568 update, and add the matching YANG model extensions in the existing VRRP YANG draft.**
