<!--
**Note:** When your enhancement is complete, all of these comment blocks should be removed.

To get started with this template:

- [ ] **Create an issue in keylime/enhancements**
  When filing an enhancement tracking issue, please ensure to complete all
  fields in that template.  One of the fields asks for a link to the enhancement.  You
  can leave that blank until this enhancement is made a pull request, and then
  go back to the enhancement and add the link.
- [ ] **Make a copy of this template.**
 name it `NNNN-short-descriptive-title`, where `NNNN` is the issue number (with no
  leading-zero padding) assigned to your enhancement above.
- [ ] **Fill out this file as best you can.**
  At minimum, you should fill in the "Summary", and "Motivation" sections.
  These should be easy if you've preflighted the idea of the enhancement with the
  appropriate SIG(s).
- [ ] **Merge early and iterate.**
  Avoid getting hung up on specific details and instead aim to get the goals of
  the enhancement clarified and merged quickly.  The best way to do this is to just
  start with the high-level sections and fill out details incrementally in
  subsequent PRs.
-->
# enhancement-120: Plugin Architecture for Modular Attestation and Verification

<!--
This is the title of your enhancement.  Keep it short, simple, and descriptive.  A good
title can help communicate what the enhancement is and should be considered as part of
any review.
-->

<!--
A table of contents is helpful for quickly jumping to sections of a enhancement and for
highlighting any additional information provided beyond the standard enhancement
template.
-->

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Story 1: Post-boot TEE Attestation](#story-1-post-boot-tee-attestation)
    - [Story 2: Pre-boot TEE Attestation](#story-2-pre-boot-tee-attestation)
    - [Story 3: eBPF Attestation](#story-3-ebpf-attestation)
  - [Notes/Constraints/Caveats (optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Pre-existing Components](#pre-existing-components)
  - [Extensible Attestation Protocol](#extensible-attestation-protocol)
  - [Verification Engines](#verification-engines)
- [New Components](#new-components)
  - [Evidence Collectors](#evidence-collectors)
  - [Extensible Registration Protocol](#extensible-registration-protocol)
    - [Decision Engines](#decision-engines)
    - [Identity Collectors](#identity-collectors)
  - [Extensible Token Issuance Protocol](#extensible-token-issuance-protocol)
- [Additional Design Details](#additional-design-details)
  - [Test Plan](#test-plan)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed (optional)](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

<!--
**ACTION REQUIRED:** In order to merge code into a release, there must be an
issue in [keylime/enhancements] referencing this enhancement and targeting a release**.

For enhancements that make changes to code or processes/procedures in core
Keylime i.e., [keylime/keylime], we require the following Release
Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These
checklist items _must_ be updated for the enhancement to be released.
-->

- [ ] Enhancement issue in release milestone, which links to pull request in [keylime/enhancements]
- [ ] Core members have approved the issue with the label `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place
- [ ] User-facing documentation has been created in [keylime/keylime-docs]

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

## Summary

<!--
This section is incredibly important for producing high quality user-focused
documentation such as release notes or a development roadmap.  It should be
possible to collect this information before implementation begins in order to
avoid requiring implementers to split their attention between writing release
notes and implementing the feature itself. Reviewers
should help to ensure that the tone and content of the `Summary` section is
useful for a wide audience.

A good summary is probably at least a paragraph in length.
-->

This enhancement proposal describes an approach to supporting plugins within Keylime so that the base components (the 
registrar, verifier and agent) can be easily extended. The principal goal of this effort is to allow new forms of
attestation evidence to be sent, received and verified by Keylime. An example of such evidence would be the attestation
reports produced by a processor to attest the launch state of a trusted execution environment (TEE) which is hosting
a confidential VM (CVM).

## Motivation

<!--
This section is for explicitly listing the motivation, goals and non-goals of
this enhancement.  Describe why the change is important and the benefits to users.
-->

The current Keylime architecture is monolithic with each form of evidence&mdash;TPM quotes, IMA logs and UEFI 
logs&mdash;intermingled at nearly every layer of the application stack. On the server side, for example, adding support 
for a new  evidence type requires not only a new policy parser and verification logic, but also changes to multiple 
different request handlers and the HTTP client which retrieves evidence from the Keylime agent. And if the evidence is 
tied to a TPM quote, further changes are likely needed to Keylime's TPM libraries as well.

In contrast, other attestation/verification software (Veraison, SPIFFE/SPIRE, Trustee\*\*, etc.) is modularised such 
that each evidence form is handled by a different plugin. This enforces separation of concerns throughout the codebase,
making improvements to the core application easier, and allows third parties to easily extend core functionality with
their own capabilities. Additionally, features can be easily turned on or off, minimising the attack surface.

To unlock these benefits for Keylime, we propose an architecture for plugins which considers the historical context and
evolves the current offering to align with the present trajectory of the industry.

**\*\* Question for Tyler:** Is this true? Does Trustee have a plugin architecture?

### Goals

<!--
List the specific goals of the enhancement.  What is it trying to achieve?  How will we
know that this has succeeded?
-->

### Non-Goals

<!--
What is out of scope for this enhancement?  Listing non-goals helps to focus discussion
and make progress.
-->

<!-- - Fully decouple TPM, IMA and UEFI verification (e.g., in `tpm_main.py`)
- Provide a generic mechanism for managing policies and reference measurements
- Back porting the plugin architecture to the pull model
- Implement support for new attestation types within the proposed plugin architecture -->


## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation.  The "Design Details" section below is for the real
nitty-gritty.
-->

Our plan for comprehensive plugin support within Keylime requires, at a high level, the following components:

1. An extensible **attestation protocol** which can support arbitrary evidence types. In addition:
    1. On the verifier side, a common interface for requesting and processing relevant evidence from agents
    2. On the agent side, a common interface for reporting available evidence and submitting that which is requested
2. An extensible **registration protocol** which can support arbitrary identities. In addition:
    1. On the registrar side, a common interface for processing identities received and preparing appropriate challenges
    2. On the agent side, a common interface for reporting identities and preparing appropriate responses
3. An extensible **token issuance protocol** by which an agent may obtain tokens of arbitrary type upon successful 
   verification. In addition:
    1. On the verifier side, a common interface for receiving and fulfilling a token request
    2. On the agent side, a common interface for requesting tokens

Progress towards this architecture has already been made with components 1 and 1(i) already largely complete. The current
state of affairs is discussed in the subsequent [Pre-existing Components](#pre-existing-components) section.

The outstanding pieces are covered in the [New Components](#new-components) section.

> [!NOTE]
> While the need for the first two protocols should be apparent, the same may not be true for the third. This is required
> to support all use cases contemplated by this proposal, in particular integration with key brokers in confidential
> computing scenarios.

### User Stories

<!--
Detail the things that people will be able to do if this enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system.  The goal here is to make this feel real for users without getting
bogged down.
-->

#### Story 1: Post-boot TEE Attestation

TODO: Post-boot CVM attestation story

#### Story 2: Pre-boot TEE Attestation

TODO: Pre-boot CVM attestation story

#### Story 3: eBPF Attestation

TODO: Some other non-CVM attestation story

<!-- Network traffic? Files? -->

### Notes/Constraints/Caveats (optional)

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above.
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

### Risks and Mitigations

<!--
What are the risks of this proposal and how do we mitigate.  Think broadly.
For example, consider both security and how this will impact the larger
enhancement ecosystem.

How will security be reviewed and by whom?
-->

## Pre-existing Components

[PR #1693](https://github.com/keylime/keylime/pull/1693) to implement a new agent-driven (push) attestation protocol
([Enhancement #103](https://github.com/keylime/enhancements/blob/master/103_agent-driven-attestation.md)) has already
made progress towards increased extensibility within Keylime. The new protocol has been designed so that additional 
evidence types can easily be supported within a common set of semantics which aims to avoid namespace collisions.

Additionally, verification logic is now contained within classes which implement the `VerifierEngine` abstract class. 
This allows additional engines to modify the attestation conversation to negotiate the appropriate parameters which the 
agent should use when collecting and preparing evidence and actually process that evidence once it is received by the 
verifier. As a result, the request handlers and database tables are agnostic to evidence type and should not require 
changes for most new forms of attestation.

### Extensible Attestation Protocol

The new attestation protocol operates in two phases: _capabilities negotiation_ and _evidence handling_. Each phase
consists of a single HTTP request and response:

```
                          Agent                                           Verifier
                          -----                                           --------
                            │                                                 │
                        ┬   │                 1. Capabilities                 │
                        │   │ ----------------------------------------------> │
            1st Phase:  │   │         e.g., supported TPM algorithms          │
           ------------ │   │                                                 │
           CAPABILITIES │   │                                                 │
            NEGOTIATION │   │            2. Attestation Parameters            │
                        │   │ <---------------------------------------------- │
                        ┴   │    e.g., ima offset, chosen algorithms, etc.    │
                            │                                                 │
                            │                                                 │
                            │                   3. Evidence                   │  ┬
                            │ ----------------------------------------------> │  │
                            │       e.g., quote, UEFI log, IMA log, etc.      │  │ 2nd Phase:
                            │                                                 │  │ ----------
                            │                                                 │  │  EVIDENCE
                            │                   4. Response                   │  │  HANDLING
                            │ <---------------------------------------------- │  │
                            │  i.e., whether the request appears well-formed  │  ┴
                            │                                                 │
```

The messages sent and received in each phase are constructed from one or more _evidence items_. Each of these JSON 
objects has an `evidence_class` and an `evidence_type`. The class indicates what fields are available within the object, 
the semantics of those fields and the data types allowable for each. Classes are defined by the Keylime project and 
meant to be generic. At present there are two: `certification` (for evidence which certifies claims, like a TPM quote or
TEE attestation report) and `log` (for event logs).

Conversely, an evidence item's type is the domain of one or more plugins. It may further constrain the allowable fields
for the evidence item and further specify the semantics of a field's values. Additional evidence types can be added at
any time by Keylime itself, or by plugin authors, so long as they do not conflict with the name of an existing type. As
the allowable fields and their semantics may change over time for a given evidence type and do so independently of the 
API version understood by the agent and verifier, evidence types themselves are also versioned.

[Examples of the JSON structures used to represent evidence can be found here.](https://gist.github.com/stringlytyped/ab76c6c83e4630be9446576fbeb7534e)

### Verification Engines

When push mode is turned on for the verifier, attestations are processed by one or more verification engines. The list
of engines are currently hard-coded, but these could be discovered dynamically from a configuration option. Each
engine has predefined set of methods which are called by the `EngineDriver` class while processing an attestation to
receive data, prepare responses and, ultimately, verify evidence. This is mainly done by reading from and updating
`self.attestation` which represents the state of the currently processed attestation.

A verification engine made to handle AMD's SEV-SNP attestation reports might look something like this:

```python
from keylime.verification.base import EngineDriver, VerificationEngine

class SNPEngine(VerificationEngine):

    @classmethod
    def register(cls, attestation):
        # This method is called automatically by Keylime to register the verification engine and ensure the below
        # callback methods are invoked at the appropriate time in the attestation lifecycle
        EngineDriver.register_support(cls, "snp_report", "certification", ["1.0"])

    def mutate_engine_selection(self, engine_selection):
        # Modify what verification engines are invoked when processing an attestation for a given agent
        if self.agent.snp_policy:
            engine_selection.append(self)

    def process_capabilities(self, evidence_selection):
        # Given an agent's reported capabilities in `self.attestation`, amend the list of evidence to request
        pass

    def process_evidence(self):
        # Receive the list of evidence sent by the agent and perform validation as appropriate
        pass

    def verify_evidence(self):
        # Perform the steps necessary to verify the evidence against policy and update the attestation's status 
        # (this method is called asynchronously after the attestation protocol has concluded)
        pass
```

Understand that multiple verification engines may modify a single evidence item (these are found in 
`self.attestation.evidence`). For example, verifying an AMD-SNP report may require checking that the same UEFI firmware
blob which is part of the TEE launch measurement has also been measured into the relevant TPM platform configuration
register (PCR). In such case, the `SNPEngine.process_capabilities(…)` method could modify the PCRs selected by the
in-built `TPMEngine`.

<!-- TODO: move -->
<!-- In so far as is possible, verification engines should be implemented such that the order in which they are invoked does
not matter. When it does  -->

## New Components

### Evidence Collectors

With support for pluggable verification of received evidence on the verifier side, a corresponding way of collecting 
that evidence in the first place is needed. If we were to conceptualise this as pluggable _evidence collectors_ to
complement the _verification engines_ from the previous section, the interface might look something like this:

- `register(…)`: called to register the evidence types and versions supported by the evidence collector
- `prepare_capabilities(…)`: called when it is time to submit an attestation to obtain the capabilities which are 
  supported by the attested node
- `collect_evidence(…)`: called on receipt of the attestation parameters chosen by the verifier to obtain evidence
  according to those parameters
- `process_final_response(…)`: called when the final response for a given attestation is received from the verifier to
  report whether the submitted evidence was accepted by the verifier (but not whether it successfully verified)

However, calling an evidence collector which might implement this interface is not as simple in Rust as it is in Python,
specifically when the collector is part of a plugin external to the main agent codebase and compiled separately.

Loading a dynamically linked library in Rust is difficult due to the lack of a stable ABI. The 
[`abi_stable` crate](https://github.com/rodrimati1992/abi_stable_crates) aims to address this by providing a stable 
foreign function interface, but we hesitate to recommend this solution as there is not much evidence of active 
maintenance.

A better approach may be to execute evidence collectors as separate processes and use pipes to pass data back and forth
inter-process. This requires Keylime to specify an ABI or other common format. The 
[`bincode` crate](https://github.com/bincode-org/bincode) or similar could be used to serialise and deserialise data
structures on either end.

On the verifier, common functionality is made available across verification engines to simplify their implementation.
For instance, evidence items are presented to engines as a `RecordSet` which allows the collection to be filtered by the
properties of the evidence item. Achieving the same on the agent side with evidence collectors implemented with pipes is 
more difficult as such collectors would have their own independent memory space. A possible solution may be to make 
available a crate which can be imported by evidence collectors.

### Extensible Registration Protocol

To bring extensibility to the registration process, we propose a modernised protocol modelled partly after the push
attestation protocol. Such a protocol would be simpler as it would not include a capabilities negotiation phase. This is
because, before a challenge&ndash;response has completed, the registering agent has not been authenticated. Submitting
a list of available identities, for instance, and letting the registrar select a subset would allow a malicious party to
discover information about the registrar's configuration.

TODO: Describe high-level protocol

Identities would be discovered on the agent side by an _identity collector_ and processed on the registrar side by a
_decision engine_.

#### Decision Engines

TODO: describe high-level interface

#### Identity Collectors

TODO: describe high-level interface

### Extensible Token Issuance Protocol

TODO: Describe a protocol for the issuance of tokens of various types. A "token" can be a simple string, a JWT, or any
arbitrary JSON object, CBOR structure or binary blob. As such, a token could consist of an EAT or similar structure,
which could be presented to a key broker to obtain decryption keys, or otherwise itself directly contain such decryption
keys. I realise using "token" for the later case feels a bit odd but I could not think of a more generic name.



## Additional Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable.  This may include API specs (though not always
required) or even code snippets.  If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

### Test Plan

<!--
**Note:** *Not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy.  Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).
-->

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this enhancement
-->

### Dependency requirements

<!--
If your new change requires new dependencies, please outline and demonstrate that your selected dependency 
is well maintained and packaged in Keylime's supported Operating Systems (currently Debian Stable
and as of time writing Fedora 32/33). 

During code implementation you will also be expected to add the package to CI , the keylime ansible role and 
keylimes main installer (`keylime/installers.sh`).

If the package is not available in the supported Operated systems, the PR will not be merged into master. 

Adding the package in `requirements.txt` is not sufficent for master which is where we tag releases from. 

You may however be able to work within an experimental branch until a package is made available. If this is
the case, please outline it in this enhancement.

-->

## Drawbacks

<!--
Why should this enhancement _not_ be implemented?
-->

## Alternatives

<!--
What other approaches did you consider and why did you rule them out?  These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

## Infrastructure Needed (optional)

<!--
Use this section if you need things infrastructure related specific to your enhancement.  Examples include a
new subproject, repos requested, github webhook, changes to CI (travis).
-->
