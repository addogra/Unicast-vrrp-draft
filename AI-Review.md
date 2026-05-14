# AI Review: `draft-abinabraham-vrrp-unicast`

Date: 2026-05-14

Reviewer perspective: senior routing-area reviewer with protocol design, implementation, and YANG review expectations similar to what the RTGWG/LSR community applies to mature routing drafts.

## Executive Review

The -00 draft had the right technical instinct: define unicast VRRP as a narrow extension that preserves VRRP packet format, protocol number, virtual IP behavior, and Virtual Router MAC behavior while replacing multicast advertisement delivery with configured unicast peers.

The main weakness was positioning. The draft was informational while simultaneously saying it updates RFC 9568. That is a mixed signal for IETF reviewers. If the document updates RFC 9568 and introduces normative behavior, it should be standards track.

The second weakness was management-plane completeness. A standards update that adds configured unicast peers should also say how that configuration and state are represented. The right base is `draft-ietf-rtgwg-vrrp-rfc8347bis-15`, which already defines the `ietf-vrrp` model.

There are now two local draft packages:

- [draft-abinabraham-vrrp-unicast-01.xml](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/draft-abinabraham-vrrp-unicast-01.xml) is the main Standards Track protocol draft. It keeps the protocol improvements but does not embed a YANG module.
- [draft-abinabraham-vrrp-unicast-01-with-yang.xml](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/draft-abinabraham-vrrp-unicast-01-with-yang.xml) is the retained working variant that includes the proposed YANG augmentation.

The main -01 draft defers YANG augmentation until the VRRP YANG bis work is stable:

> This document does not define a YANG module. The VRRP YANG data model is currently being revised by `draft-ietf-rtgwg-vrrp-rfc8347bis`, which is intended to obsolete RFC 8347. Deferring the unicast VRRP YANG augmentation until that work is approved or published allows the augmentation to be based on the stable successor to RFC 8347, avoids unnecessary churn against a moving base model, and helps ensure alignment with the final VRRP YANG schema.

## What IETF Reviewers Are Likely to Care About

Based on the existing RTGWG discussion around `draft-ietf-rtgwg-vrrp-rfc8347bis`, the review pattern is predictable:

- References must be current and properly classified.
- A draft that claims to update an RFC must be precise about exactly what text or behavior it updates.
- A YANG module must validate and must not create migration surprises.
- Operational state and notifications need to be useful, not decorative.
- Deprecated or legacy behavior should not leak into new work unless there is a strong reason.
- The document should separate normative protocol behavior from implementation survey material.

Relevant public discussion:

- RTGDIR early review of `draft-ietf-rtgwg-vrrp-rfc8347bis-01` called out obsolete references and withdrawn external references.
- Acee Lindem replied that the references would be updated.
- Acee later asked for substantive YANG review before IESG processing and noted the opportunity to improve operational state while doing the bis.

Useful links:

- [Published unicast VRRP draft](https://datatracker.ietf.org/doc/draft-abinabraham-vrrp-unicast/)
- [VRRP YANG bis draft](https://datatracker.ietf.org/doc/draft-ietf-rtgwg-vrrp-rfc8347bis/)
- [RTGDIR early review thread](https://mailarchive.ietf.org/arch/msg/rtgwg/A2qhgy0GzS7CWQ46gL1vGWePDjE/)
- [Acee response to RTGDIR review](https://mailarchive.ietf.org/arch/msg/rtgwg/xH6t-iPVipzc0opzZ0HG053bnDw/)
- [Acee request for substantive YANG review](https://mailarchive.ietf.org/arch/msg/rtgwg/mr1kAJ5Dpnen4CdYYnQG7llBH4U/)

As of 2026-05-14, the datatracker lists `draft-ietf-rtgwg-vrrp-rfc8347bis-15` as an active RTGWG Internet-Draft, submitted to the IESG for publication, in AD Evaluation, with YANG validation reporting 0 errors and 0 warnings.

## Changes Made in the Proposed -01

The -01 revision makes the draft look more like a standards-track RFC 9568 update:

- Changed intended status from Informational to Standards Track.
- Bumped the main draft name from `draft-abinabraham-vrrp-unicast-00` to `draft-abinabraham-vrrp-unicast-01`.
- Updated the date to 2026-05-14.
- Clarified that the document updates VRRP Version 3 and does not update VRRP Version 2.
- Made multicast mode explicitly the default.
- Made unicast mode explicitly configured.
- Clarified mode mismatch handling.
- Removed the implementation-status survey from the normative body.
- Added YANG considerations that defer the YANG augmentation until `draft-ietf-rtgwg-vrrp-rfc8347bis` has been approved or published.
- Kept a separate local `-01-with-yang` artifact with a candidate YANG module for later reuse.

## YANG Review

The current `draft-ietf-rtgwg-vrrp-rfc8347bis-15` model already has a clean per-interface and per-address-family structure:

- IPv4 VRRP augments `/if:interfaces/if:interface/ip:ipv4`
- IPv6 VRRP augments `/if:interfaces/if:interface/ip:ipv6`
- each address family contains `vrrp/vrrp-instance`
- `vrrp-instance` is keyed by `vrid`
- existing operational state includes `state`, `last-adv-source`, `last-event`, and per-instance statistics
- existing notifications include `vrrp-new-active-event`, `vrrp-protocol-error-event`, and `vrrp-virtual-router-error-event`

The right approach is to augment the existing model, not define a parallel unicast VRRP model.

The retained `-01-with-yang` working artifact adds:

- feature `unicast-vrrp`
- typedef `vrrp-advertisement-mode`
- per-instance leaf `advertisement-mode`
- IPv4 unicast peer list
- IPv6 unicast peer list
- unicast peer-source error counter
- unicast mode-mismatch error counter
- virtual-router error identities for peer-source and mode-mismatch errors

I initially modeled the peer list with `min-elements 1`, but `pyang` correctly rejected that because an augment cannot add a mandatory node into an existing tree. The corrected version uses a `must` expression on `advertisement-mode` to require at least one peer when `advertisement-mode = unicast`.

YANG validation performed:

- extracted `ietf-vrrp@2026-02-13.yang` from `draft-ietf-rtgwg-vrrp-rfc8347bis-15`
- extracted `ietf-vrrp-unicast@2026-05-14.yang` from the -01 XML
- validated both with `pyang`
- final `pyang` result: clean

## Review Findings

### Finding 1: Standards-track status is the right direction

Severity: High

The draft says it updates RFC 9568 and uses normative requirements. Keeping it informational would create process friction. Reviewers will ask whether this is really documenting deployment experience or changing protocol behavior.

Recommendation:

Keep -01 as Standards Track and preserve the `updates="9568"` metadata.

### Finding 2: Explicit default mode is essential

Severity: High

The two-mode model needs to be unambiguous. Existing VRRP behavior must remain multicast unless the operator explicitly configures unicast mode.

Recommendation:

Keep the text that says multicast mode is default and unicast mode is explicit configuration.

### Finding 3: The draft should be VRRPv3-only

Severity: High

RFC 9568 is VRRPv3. Huawei's public unicast VRRP material is VRRPv2-based and transport-divergent, but pulling that into this draft would make the work much harder to standardize.

Recommendation:

Keep the text that says this document updates VRRPv3 and does not update VRRPv2.

### Finding 4: Vendor survey material belongs outside the standards body

Severity: Medium

The implementation evidence is useful for WG adoption, but it makes the protocol document longer and easier to argue with. Standards text should define behavior, not compare products.

Recommendation:

Keep detailed implementation evidence in `vrrp_ucast_industry_research.md`, `lessons.md`, and adoption slides. Keep the standards draft concise.

### Finding 5: YANG belongs in the same revision plan

Severity: High

A configured peer list is central to unicast mode. If the protocol draft does not include YANG or a concrete companion YANG plan, reviewers will likely call the management model incomplete.

Recommendation:

Keep the main -01 draft focused on protocol behavior and include a YANG Considerations section that explicitly defers the augmentation until `draft-ietf-rtgwg-vrrp-rfc8347bis` is approved or published. Retain `-01-with-yang` as implementation-ready input for a future revision or companion YANG draft.

### Finding 6: YANG peer validation needs careful modeling

Severity: Medium

The model should require peers in unicast mode, but YANG augment rules prohibit adding mandatory nodes into an existing tree. The current `must` expression is the correct direction.

Recommendation:

Ask a YANG doctor or RTGWG YANG reviewer to confirm the final modeling pattern before submission.

### Finding 7: Future YANG security text needs to be explicit about management risk

Severity: Medium

Changing `advertisement-mode` or the peer list can break first-hop redundancy. This is not just operational state; it is safety-critical configuration. Since the main no-YANG -01 defers the YANG module, this risk should be carried into the future YANG revision or companion YANG draft.

Recommendation:

Keep the YANG security text in the retained `-01-with-yang` artifact and consider adding NACM-specific language when the YANG augmentation is submitted.

## Remaining Open Questions

The next human review should decide:

- Should the deferred YANG work return in a later revision of this protocol draft, or become `draft-abinabraham-vrrp-unicast-yang`?
- Should unicast mode be explicitly limited to VRRPv3 in the YANG feature, or is the surrounding protocol text enough?
- Should reviewers prefer a shorter leaf name later, the current recommendation is to keep `advertisement-mode` because it is explicit and follows IETF naming patterns such as `label-advertisement-mode`.
- Should `unicast-mode-mismatch-error` be a virtual-router error only, or should there also be a global protocol error identity?
- Should the draft keep the `Use Cases and Deployment Drivers` section, or collapse it into Scope and Applicability for concision?

## Two AI Review Passes

### Pass 1: Protocol and WG-Adoption Review

The first AI review pass checked whether the main no-YANG -01 is ready to be read as a Standards Track update to RFC 9568.

Result:

- Confirmed that the draft is Standards Track and retains `updates="9568"`.
- Confirmed that multicast mode remains the default and unicast mode is explicitly configured.
- Confirmed that the draft is scoped to VRRPv3 and does not update VRRPv2.
- Confirmed that the packet format, IP protocol number, state machine, Virtual Router MAC, and host-facing forwarding behavior remain unchanged.
- Found and fixed one issue: the abstract still said the draft added "YANG data model extensions" after the no-YANG split. The abstract now says YANG augmentation is deferred until the VRRP YANG bis work is stable.

### Pass 2: Process, Editorial, and Artifact Review

The second AI review pass checked the document as a submission package and looked for stale text after the YANG split.

Result:

- Confirmed that the main -01 no longer embeds a YANG module or requests YANG-related IANA registrations.
- Confirmed that `draft-ietf-rtgwg-vrrp-rfc8347bis-15` and RFC 8347 are informative references in the main no-YANG -01.
- Found and fixed one consistency issue: the YANG deferral text now says "approved or published" consistently.
- Confirmed that the YANG-inclusive `-01-with-yang` package is retained separately for later reuse.
- Regenerated review diffs for the full -00 to -01 change, the YANG-inclusive to no-YANG split, and the focused two-pass review fixes.

## Generated Artifacts

Generated from the main no-YANG -01:

- [draft-abinabraham-vrrp-unicast-01.xml](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/draft-abinabraham-vrrp-unicast-01.xml)
- [draft-abinabraham-vrrp-unicast-01.txt](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/draft-abinabraham-vrrp-unicast-01.txt)
- [draft-abinabraham-vrrp-unicast-01.html](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/draft-abinabraham-vrrp-unicast-01.html)
- [draft-abinabraham-vrrp-unicast-01.txt.html](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/draft-abinabraham-vrrp-unicast-01.txt.html)
- [draft-abinabraham-vrrp-unicast-01.idnits.txt](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/draft-abinabraham-vrrp-unicast-01.idnits.txt)
- [draft-abinabraham-vrrp-unicast-00-to-01.iddiff.html](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/draft-abinabraham-vrrp-unicast-00-to-01.iddiff.html)
- [draft-abinabraham-vrrp-unicast-00-to-01.diff](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/draft-abinabraham-vrrp-unicast-00-to-01.diff)
- [draft-abinabraham-vrrp-unicast-01-with-yang-to-01.iddiff.html](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/draft-abinabraham-vrrp-unicast-01-with-yang-to-01.iddiff.html)
- [draft-abinabraham-vrrp-unicast-01-with-yang-to-01.diff](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/draft-abinabraham-vrrp-unicast-01-with-yang-to-01.diff)
- [draft-abinabraham-vrrp-unicast-01.two-ai-reviews.iddiff.html](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/draft-abinabraham-vrrp-unicast-01.two-ai-reviews.iddiff.html)
- [draft-abinabraham-vrrp-unicast-01.two-ai-reviews.diff](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/draft-abinabraham-vrrp-unicast-01.two-ai-reviews.diff)
- [draft-abinabraham-vrrp-unicast-01.remove-changes-section.iddiff.html](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/draft-abinabraham-vrrp-unicast-01.remove-changes-section.iddiff.html)
- [draft-abinabraham-vrrp-unicast-01.remove-changes-section.diff](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/draft-abinabraham-vrrp-unicast-01.remove-changes-section.diff)

Retained YANG-inclusive working artifacts:

- [draft-abinabraham-vrrp-unicast-01-with-yang.xml](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/draft-abinabraham-vrrp-unicast-01-with-yang.xml)
- [draft-abinabraham-vrrp-unicast-01-with-yang.txt](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/draft-abinabraham-vrrp-unicast-01-with-yang.txt)
- [draft-abinabraham-vrrp-unicast-01-with-yang.html](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/draft-abinabraham-vrrp-unicast-01-with-yang.html)
- [draft-abinabraham-vrrp-unicast-01-with-yang.txt.html](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/draft-abinabraham-vrrp-unicast-01-with-yang.txt.html)
- [draft-abinabraham-vrrp-unicast-01-with-yang.idnits.txt](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/draft-abinabraham-vrrp-unicast-01-with-yang.idnits.txt)
- [draft-abinabraham-vrrp-unicast-00-to-01-with-yang.iddiff.html](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/draft-abinabraham-vrrp-unicast-00-to-01-with-yang.iddiff.html)
- [draft-abinabraham-vrrp-unicast-01-with-yang.advertisement-mode.iddiff.html](/Users/addogra/Desktop/code-repos/Unicast-vrrp-draft/draft-abinabraham-vrrp-unicast-01-with-yang.advertisement-mode.iddiff.html)

Validation summary for the main no-YANG -01:

- `xmllint`: pass
- `xml2rfc --text --html`: pass
- `idnits`: boilerplate and checklist pass; remaining findings are local reference-status noise for RFC 9568, normal downref/status noise from the local idnits database, and an informative work-in-progress reference to `draft-ietf-rtgwg-vrrp-rfc8347bis`

Validation summary for `-01-with-yang`:

- `xmllint`: pass
- `xml2rfc --text --html`: pass
- `pyang`: pass against extracted `ietf-vrrp@2026-02-13.yang`
- `idnits`: includes the expected naming warning because `draft-abinabraham-vrrp-unicast-01-with-yang` is a local working name and not a datatracker submission name ending in `-NN`

## Recommended WG Strategy

The best WG adoption story is:

"This draft updates RFC 9568 to add one narrowly scoped advertisement mode. Multicast remains default. Unicast is configured explicitly. The VRRP packet format, state machine, virtual IP behavior, and Virtual Router MAC behavior are unchanged. The companion YANG extension is derived from the current VRRP YANG bis structure."

That story gives reviewers fewer places to push back. It also separates this work from broader vendor-specific unicast VRRP variants that use different transport or non-gateway election semantics.

## Next Best Edit Before Submission

Submit or circulate the main no-YANG -01 first, using the YANG Considerations text to show that the management model has not been ignored. Keep `-01-with-yang` available as the candidate augmentation once `draft-ietf-rtgwg-vrrp-rfc8347bis` is approved or published.
