# Agentic Internet Constitution Completeness Review

| Field | Value |
|---|---|
| Review date | 2026-08-25 |
| Reviewed document | [The Agentic Internet Constitution — TOS Founding Edition](AGENTIC_INTERNET_CONSTITUTION.md) |
| Reviewed version | Founding Draft 0.5 |
| Review type | Three-pass internal constitutional and architecture completeness review |
| Repository base | `tosnetwork/doc` `main` at `9481f45` |
| Review result | Accepted for public proposal review after remediation; not ratified |

## 1. Scope and Evidence Boundary

This review asks whether the proposed Constitution is sufficiently complete to
guide both:

1. an open Agentic Internet that can be implemented by independent parties;
   and
2. TOS as a founding proposer, reference ecosystem, and Constitutional Adopter.

The review covers institutional scope, authority, human and Agent rights,
protocol openness, independent implementation, cross-domain security, economic
integrity, privacy, governance, emergency powers, repository boundaries,
compliance, and ratification.

This is a review of constitutional text and architecture boundaries. It is not:

- ratification;
- legal advice or jurisdiction-specific legal approval;
- an audit of deployed TOS code, contracts, keys, networks, or organizations;
- proof that a protocol specification is complete or independently
  implementable;
- a security assessment of a production implementation; or
- evidence of deployment, adoption, demand, revenue, or economic settlement.

## 2. Review Method

The review used four tests:

1. **Internal closure:** every right, duty, authority, exception, amendment,
   and emergency action must identify a responsible subject and review path.
2. **Agentic Internet neutrality:** a protocol implementer must be able to
   interoperate without becoming subordinate to TOS or reusing TOS code.
3. **TOS authority integrity:** constitutional direction, normative protocol,
   implementation behavior, finalized chain state, Agreements, and derived
   projections must remain distinct.
4. **Adversarial interpretation:** ambiguous clauses were tested for silent
   centralization, self-granted Agent authority, replay across domains,
   indefinite emergency control, false conformance, and evidence inflation.

The review also compared the document's direction with its stated social
foundations and with established Internet engineering principles concerning
end users, end-to-end intelligence, open process, protocol responsibility,
interoperability, and implementation evidence.

## 3. Findings and Remediation

| ID | Severity | Finding | Resolution |
|---|---|---|---|
| C-01 | High | Protocol implementation could be confused with constitutional adoption, allowing TOS governance to appear to bind independent implementers. | Added constitutional definitions that separate Constitutional Adopter, Protocol Implementer, Participant, and Affected Party. Protocol conformance alone creates no constitutional subordination. |
| C-02 | High | The document did not define how constitutional direction, normative specifications, source code, finalized state, Agreements, and AI or indexer claims interact when they conflict. | Added an authority-question matrix and a fail-closed conflict process. The Constitution cannot rewrite finalized state, protocol versions, or accepted Agreements. |
| C-03 | High | Emergency powers lacked an identified invoker, maximum duration, automatic expiry, renewal rule, and explicit capability record. | Replaced the emergency section with bounded authorization, declaration contents, multi-party review, automatic expiry, renewal, evidence preservation, and prohibited-purpose rules. |
| C-04 | High | No compliance mechanism converted constitutional principles into repository, protocol, release, and institutional duties. | Added mandatory compliance records, review triggers, exception rules, violation handling, and separation of constitutional, protocol, security, deployment, and legal claims. |
| C-05 | Medium | Cross-domain authorization and replay protection were not constitutional requirements despite the multi-network and multi-service scope. | Added domain separation and cross-domain trust requirements covering version, purpose, parties, revisions, semantic identity, freshness, bridges, oracles, relayers, and wrapped representations. |
| C-06 | Medium | Open implementation was protected, but protocol evolution, downgrade resistance, deprecation, confidential security periods, and fork freedom were incomplete. | Added an append-only protocol lifecycle, public migration evidence, equal implementer information, bounded vulnerability confidentiality, and truthful fork rules. |
| C-07 | Medium | TOS lacked a constitutional authority map connecting documentation, TIPs, normative service specifications, base-layer code and state, reference libraries, Gateways, and OpenFox. | Added an explicit repository and institutional authority map. Cached types, SDKs, databases, and reference implementations cannot appropriate normative or finalized authority. |
| C-08 | Medium | Agent authority controls did not cover disclosure, lifecycle, controller rotation, runtime and model separation, transfer, recovery, and termination. | Extended human-primacy and authority articles with automated-status disclosure and a complete persistent-Agent authority lifecycle. |
| C-09 | Medium | The Constitution centered participants but did not sufficiently protect non-participants affected by financial, identity, privacy, access, or physical actions. | Added accountability to affected parties, proportional safeguards, evidence and remedy paths, and explicit rejection of autonomy or payment as excuses for externalized harm. |
| C-10 | Medium | Governance records did not require conflict disclosure, recusals, affected-party notice, dissent, implementation ownership, or expiry conditions. | Added institutional transparency, anti-capture rules, steward conflict handling, and a complete governance decision record. |
| C-11 | Medium | The canonical constitutional artifact, translation authority, and amendment history were not defined. | Bound authority to the ratified commit and digest, made the TOS Founding Edition's English text authoritative unless co-ratified, and required append-only interpretations and translations. |
| C-12 | Medium | Economic integrity did not explicitly classify related-party activity, test transfers, custody segregation, or Sybil-distorted identity and demand. | Added related-party economic classification, custody reconciliation, and a prohibition on treating accounts, Agents, tasks, votes, or transfers as unique humans or independent demand without an appropriate model. |
| C-13 | Medium | Privacy rules did not explicitly constrain model-training reuse or explain immutable-state deletion limits. | Added authority requirements for training and secondary use, plus disclosure and data-minimization duties before immutable publication. |
| C-14 | Medium | Internet-specific end-user and end-to-end design principles were implicit rather than constitutional. | Added end-to-end agency, transport pluralism, intermediary burden of proof, resilience duties, and authoritative Internet engineering references. |
| C-15 | Low | The institutional-incentive section implemented Douglass North's idea but did not name him symmetrically with Ostrom, Hayek, Coase, and Simon. | Named North and assigned the incentive and institutional-evolution rule explicitly. |
| C-16 | High | Intent discovery, AI interpretation, negotiation, and Gateway state were not explicitly prohibited from becoming accepted commercial or payment authority. | Required exact protocol acceptance, typed Agreement commitments, and equality across escrow, execution gates, Receipts, recovery, and settlement. External and direct settlement must identify evidence and cannot be presented as TOS-finalized state. |
| C-17 | Medium | Capability-market metadata and task-controlled content could be read as trusted instructions during Skill or MCP admission. | Classified descriptions, retrieved content, model output, task instructions, and tool responses as untrusted authorization inputs and required revocable, pinned, least-privileged, risk-isolated admission. |

All Round 1 findings above were remediated in Founding Draft 0.3. Remediation
means the text contains the necessary rule; it does not prove that TOS software
or operations implement that rule.

## 4. Round 2 Adversarial Re-review

Round 2 reviewed the interaction between the Round 1 additions rather than
repeating the same checklist. It tested whether definitions, privacy,
licensing, commerce, emergency action, and compliance rules could contradict
one another or preserve a discretionary bypass.

| ID | Severity | Finding | Resolution or gate |
|---|---|---|---|
| R2-01 | High | `TOS` was defined only as the eventual ratified scope, making pre-ratification references ambiguous and leaving `major` and `critical` open to discretionary interpretation. | Separated the broader TOS ecosystem from the exact TOS Adoption Scope and defined Owner, Controller, Major Surface, and Critical Violation. Required an append-only scope register with accountable owners. |
| R2-02 | Medium | The transaction-cost rule could be misread as putting legal ownership of physical or off-chain assets under blockchain consensus, while the Constitution lacked an explicit legal-advice boundary. | Limited the rule to canonical digital facts, required an identified adopted authority, and added a legal applicability and accountable-entity duty. |
| R2-03 | Medium | Permanent retention of every failed experiment could conflict with privacy, confidentiality, minimization, and deletion duties. Pseudonymous participation was also undefined. | Limited historical retention to required decision and evidence commitments under applicable retention rules; prohibited retaining raw sensitive data merely for history; added bounded pseudonymity and selective disclosure. |
| R2-04 | High | Copyright and source-code licensing alone did not prevent essential-patent, certification-mark, or defective-test capture of an allegedly open protocol. | Added royalty-free, worldwide, non-discriminatory terms for controlled essential claims, disclosure of unresolved third-party claims, self-declared conformance rights, certification separation, and test-suite correction and appeal. |
| R2-05 | Medium | An authority-aware interface could still present a signed, submitted, or locally observed action as finalized. | Required explicit proposed-through-ambiguous lifecycle states and prohibited optimistic finality presentation. |
| R2-06 | High | Accepted commercial binding used `SHOULD`, allowing implementations to omit a material task, evidence, custody, dispute, or recovery term while still claiming constitutional compliance. | Changed the rule to `MUST` for every material term while retaining an applicability test for irrelevant fields. |
| R2-07 | Medium | Mandatory public governance records could expose personal data, live vulnerabilities, credentials, or confidential Agreement material. | Added bounded redaction with mandatory disclosure of the decision's existence, authority, scope, evidence class, accountable custodian, and later-disclosure conditions. |
| R2-08 | High | Emergency declarations required automatic expiry even where it was not technically enforceable, while multi-party approval remained only recommended for high-impact power. | Added procedural expiry when technical expiry is unavailable, made multi-party approval mandatory except for demonstrated imminent-harm delay, and added separate, logged, tested, and post-incident-revoked emergency access. |
| R2-09 | High | A Critical Violation blocked a conformance claim but only `SHOULD` block production, preserving an unsafe release bypass. | Required it to block dependent production activation; allowed only narrowly scoped containment or remediation through the recorded emergency or exception process. |
| R2-10 | Medium | Compliance records did not identify who issued the claim, the assessment method, evidence time, or exact evaluated artifact or deployment. | Added claim provenance and exact artifact or deployment identifiers. |
| R2-11 | Medium | The TOS open-protocol gate was abstract and did not record current repository evidence. | The reviewed local snapshots found GPLv3 in this proposed `doc` working tree, `tos@377fbf9c0782`, and `tos-service-protocol@4fe4342a39e0`; MIT in `openfox@6b997b638b7a`; and CC0 in `TIP@d7ffebdf842c`. No root license was found in `tos-service-spec@5ad6ad741d33` or `tos-service-gateway@98dfe8a3b00e`. Open-protocol or reference-implementation claims for those unlicensed surfaces remain blocked pending an explicit repository license and fresh verification. |
| R2-12 | Medium | The privacy rule that public consensus contain only information requiring public verification was too absolute for voluntary public publication and programmable applications. | Replaced it with data minimization and a prohibition on required raw sensitive or unrelated data. Allowed intentional non-confidential publication after permanence disclosure while clarifying that inclusion does not prove content truth or endorsement. |
| R2-13 | Medium | The strengthened Agreement binding still omitted governing protocol/profile versions and Provider Offer quantity, capacity, and single- or multi-acceptance semantics. | Added protocol/profile versions, quantity, capacity, and acceptance cardinality to the materially binding Agreement terms enforced across escrow, execution, Receipt, recovery, and settlement. |

All constitutional-text defects identified in Round 2 were remediated in
Founding Draft 0.4. R2-11 is an external repository-readiness gate, not a defect
that this document can license on another repository's behalf.

## 5. Round 3 Anti-circumvention Re-review

Round 3 treated the Constitution itself as an attack surface. It tested whether
a founder, maintainer, Agent, protocol body, emergency operator, or future key
holder could comply with the form of one clause while defeating its purpose
through a lower-level label, repository split, repeated exception, credential
handover, or unsupported stability claim.

| ID | Severity | Finding | Resolution or gate |
|---|---|---|---|
| R3-01 | High | The persistent-Agent lifecycle was advisory, and key rotation or recovery lacked a forward-effective rule. A controller could claim that new credentials silently authorized past actions or invalidated inconvenient history. | Made a complete lifecycle mandatory for persistent Agent authority capable of Consequential Actions. Added forward-effective rotation, revocation, suspension, recovery, and compromise rules; contested prior actions now require an adopted dispute, recovery, restitution, or governance record. |
| R3-02 | Medium | The right to export and migrate data did not define a portable evidence format or prevent migration from silently copying credentials, transferring authority, changing an Agreement, or elevating local state to canonical fact. | Required a public, versioned portability format that preserves provenance, integrity, revision, revocation, and authority-domain context without exporting secrets or another party's protected data. Added authority-preserving migration constraints. |
| R3-03 | Medium | Bounded adaptation did not require a predeclared hypothesis, control, harm metric, budget, affected-party analysis, stop condition, rollback path, or cumulative treatment of repeated experiments. | Required those controls before an experiment can affect a Consequential Action. Prohibited success metrics that omit failures, disputes, subsidies, shadow costs, or harm, and required cumulative review and fresh promotion decisions. |
| R3-04 | High | Valid remote authorization, governance, Agreement, or payment could still be interpreted as sufficient authority for a dangerous physical-world action. | Made the independent local safety interlock non-overridable by remote, task, model, payment, or governance authority and required it to fail to a declared safe state. |
| R3-05 | High | Stable-protocol status required test materials but not independently reproduced acceptance and rejection behavior. A reference implementation could remain the hidden specification. | Made independent interoperability evidence mandatory before canonical or stable status. Required an independent implementation, clean-room verifier, or documented independent reproduction plus positive and applicable negative vectors, including replay, downgrade, ambiguity, and recovery cases. |
| R3-06 | High | Enhanced review did not create an entrenched constitutional boundary. An ordinary amendment, exception, scope decision, or operational implementation could remove human primacy, Agent self-grant limits, open implementation, evidence integrity, meaningful exit, or emergency constraints. | Defined an Entrenched Core. Lower-level actions cannot derogate from it; a change requires a separately ratified major constitutional version, enhanced public and independent review, and explicit compatibility, migration, remedy, and exit consequences. Certain evidence, authority, history, replay, and false-conformance abuses cannot be authorized by constitutional process. |
| R3-07 | High | Related changes could be split among repositories, releases, exceptions, or successive emergency periods to avoid the classification and review required for their combined effect. | Required cumulative classification of related, staged, repeated, and mutually dependent changes. Repeated exceptions and emergencies with substantially the same purpose or effect become a proposed permanent change once they exceed temporary containment. |
| R3-08 | Medium | The text defined an exact canonical artifact but not how post-ratification spelling, link, interpretation, or errata changes affect its bytes. In-place edits could create two different documents under one apparent version. | Added an editorial-candidate class and required every post-ratification textual change to create a new candidate artifact, commit, digest, and version. The old version remains canonical until explicit ratification; interpretations and errata remain append-only. |
| R3-09 | Medium | Founder succession lacked incapacity handling, verifiable handover, credential rotation or revocation, and a distinction between possession of an operational key and constitutional authority. | Made succession, incapacity, and recovery paths mandatory. Required predecessor and successor identity, effective time, scope, obligations, conflicts, handover evidence, and credential treatment; key possession alone cannot confer stewardship. |
| R3-10 | Medium | Patent promises referred to an adopted contribution policy, but protocol governance was not required to publish that policy before accepting normative contributions. | Required a contribution and intellectual-property policy covering contributor authority, license compatibility, custody, essential-claim disclosure, and later ownership or licensing disputes. A contribution or merge cannot create a private implementation veto. |
| R3-11 | Medium | The ratification record did not require the canonical document path and version, verifiable decision signatories, stewardship and credential-custody continuity, an initial compliance baseline, or the contribution/IPR policy. | Added each field to the mandatory ratification record and strengthened the ratification clause to bind the exact path, version, signatory or decision-verification method, commit, and digest. |
| R3-12 | Medium | The constitutional decision test did not directly ask reviewers to detect cumulative evasion, Entrenched Core changes, unsupported stability claims, historical effects of key recovery, or bypass of physical safety interlocks. | Added explicit decision-test questions for all five conditions so they become routine review evidence rather than implicit interpretation. |
| R3-13 | High | The existing constitutional-presumption clause said even history rewriting, Agent self-grant, hidden authority, or evidence misrepresentation might overcome rejection with exceptional evidence. That contradicted the new Entrenched Core and preserved a textual bypass. | Made those abuses non-ratifiable under this Constitution. Kept a rebuttable presumption only for proposals that weaken safeguards without crossing the absolute prohibition, and explicitly subordinated Entrenched Core changes to the major-version process. |
| R3-14 | High | The founding steward could retain unilateral constitutional authority indefinitely even though polycentric governance is the governing core. Progressive distribution was advisory and had no mandatory review event or capture assessment. | Required the founding ratification record to set a public authority-review date and transition criteria. Each review must publish the authority and concentration map, delegation and succession readiness, capture risks, decision, and next trigger; missing review cannot expand authority. |
| R3-15 | Medium | A future ratifying body could acquire constitutional power without defined constituencies, membership, removal, quorum, conflicts, public procedure, challenge rights, or a Sybil-resistant representation model. | Required those institutional rules before any future body exercises ratification authority and prohibited treating tokens, accounts, validators, Agents, or protocol identities as unique human mandates without an adopted model. |
| R3-16 | High | Stable semantic identity was only recommended even for consequential or irreversible effects that can be retried, relayed, resumed, or submitted by multiple actors. An implementation could omit idempotent identity while still claiming constitutional compliance. | Made stable semantic identity and exact request commitment mandatory for every Consequential Action or irreversible economically or operationally meaningful effect exposed to those duplicate-execution paths. |
| R3-17 | High | Meaningful exit was declared part of the Entrenched Core, but the operative data export, revocation, provider-change, cessation, and public portability-format clauses remained advisory. | Made the responsible service or protocol provide practical exit mechanisms and made the public, versioned, authority-preserving portability format mandatory, subject to counterparty rights, retention duties, and existing obligations. |
| R3-18 | High | Avoiding exposure of an unrestricted owner key or root authority to an Agent, model, Skill, MCP server, task-controlled browser, or other untrusted process was only recommended. | Made usable-key exposure to those contexts prohibited. A user-controlled custody or signing component may hold the key only inside a documented protected boundary; Agents receive bounded capabilities instead of unrestricted root authority. |

All constitutional-text defects identified in Round 3 were remediated in
Founding Draft 0.5. The repository-license evidence gate identified as R2-11
remains external and unresolved.

## 6. Completeness Matrix

| Review domain | Result | Remaining condition |
|---|---|---|
| Agentic Internet purpose and scope | Pass | Independent adopters must declare their own adopted scope. |
| Polycentric and subsidiarity model | Pass | Operational governance bodies are not yet constituted. |
| Human purpose and Agent accountability | Pass | Implementations must provide actual controls, evidence, and redress. |
| Meaningful exit and portability | Pass at constitutional level | Services and protocols must implement public authority-preserving formats and practical export, revocation, provider-change, and cessation paths. |
| Open protocol and independent implementation | Conditional | Each stable protocol needs its own explicit license, contribution/IPR and essential-claims policy, normative version, public positive and negative conformance suite, and independent implementation or clean-room reproduction. The reviewed `tos-service-spec` and `tos-service-gateway` snapshots have no root license. |
| End-to-end and transport neutrality | Pass | Availability and anti-abuse behavior remain protocol-specific. |
| Authority and evidence separation | Pass | Cross-repository compliance records are not yet created. |
| Identity, delegation, revocation, recovery, and succession | Pass at constitutional level | Exact state machines, credential-custody controls, and operational recovery tests remain implementation work. |
| Domain separation and cross-domain trust | Pass at constitutional level | Every signing profile and bridge requires implementation and adversarial tests. |
| Intent Commerce and institutional routing | Pass | Accepted-work, evidence, settlement, and recovery bindings remain protocol-specific gates. |
| Skills, MCP, model, and supply-chain governance | Pass | Admission implementations and trusted-market evidence remain product work. |
| Privacy and data governance | Pass at constitutional level | Jurisdictional requirements and implementation controls need separate review. |
| Root authority and physical safety | Pass at constitutional level | Implementations must prove bounded delegation, no usable root-key exposure to untrusted execution, and non-bypassable local safety interlocks. |
| Economic integrity and custody | Pass at constitutional level | Accounting controls, solvency evidence, and live settlement remain unverified. |
| Amendment and decision governance | Pass at constitutional level | The Entrenched Core, cumulative classification, version integrity, founding-authority review, and minimum rules for future ratifying bodies are defined; the bodies themselves are not yet constituted. |
| Emergency powers | Conditional | The ratification record must name the emergency authority and adopted incident-response rule. |
| TOS repository and institutional authority map | Pass | Current repository roadmaps and deployment evidence still control implementation status. |
| Constitutional compliance and remedies | Conditional | TOS must assign an owner, review cycle, formats, and initial repository assessments. |
| Ratification readiness | Blocked | Exact path, version, signatory evidence, canonical commit and digest, public review, independent review, adopted scope, initial compliance baseline, contribution/IPR policy, transition plan, emergency rule, succession rule, and compliance owner remain pending. |

## 7. Ratification Gates

The Constitution MUST remain a proposal until the ratification record contains:

1. a named ratifying authority, verifiable decision signatories or equivalent
   verification method, and explicit adoption decision;
2. the constitutional version, canonical document path, repository commit, and
   document SHA-256;
3. the adopted organizational, repository, network, and protocol scope;
4. a public review record, review period, objections, and responses;
5. independent institutional and security review evidence;
6. a transition plan for known conflicts with existing documents or systems;
7. an emergency authority and adopted incident-response rule;
8. a constitutional compliance owner and review cycle;
9. protocol-specific licensing and conformance policy where open-protocol
   status is claimed, including contribution/IPR, essential-patent,
   certification, independent-implementation, and positive and negative test
   evidence;
10. the TOS Adoption Scope register, Major Surface owners, and legal
    applicability statement;
11. a stewardship succession, incapacity, recovery, and credential-custody
    rule;
12. a founding-authority review date, transition criteria, and initial
    authority and concentration map;
13. an initial compliance baseline for every included Major Surface; and
14. any separate validator, contract, legal, or organizational approvals needed
    to activate dependent changes.

## 8. Final Assessment

Founding Draft 0.5 is sufficiently complete to serve as a public constitutional
proposal and as the governing design framework for subsequent Agentic Internet
and TOS specifications. Its core doctrine is coherent:

> Local judgment, protocol coordination; minimum authority, verifiable
> commitments; polycentric governance, meaningful exit; bounded experiments,
> no automatic expansion of power.

The document now constrains TOS while preserving independent implementation
and external governance. It identifies the authority and evidence boundaries
needed to prevent a blockchain, Gateway, marketplace, model, founder, or
reference repository from becoming universal authority by convenience. It also
prevents lower-level labels, chained exceptions, emergency renewal, credential
possession, and in-place editorial changes from silently acquiring
constitutional effect.

The review result is **accepted for public proposal review, not ratified**.
Production, deployment, economic, legal, and independent-security claims remain
separate and unverified until their respective evidence gates are satisfied.
