---
title: Attestation Event Stream Subscription
abbrev: RATS Subscription
docname: draft-ietf-rats-network-device-subscription-latest
wg: RATS Working Group
stand_alone: true
ipr: trust200902
area: Security
kw: Internet-Draft
cat: std
pi:
  toc: 'yes'
  sortrefs: 'yes'
  symrefs: 'yes'

author:
- ins: H. Birkholz
  name: Henk Birkholz
  org: Fraunhofer SIT
  abbrev: Fraunhofer SIT
  email: henk.birkholz@sit.fraunhofer.de
  street: Rheinstrasse 75
  code: '64295'
  city: Darmstadt
  country: Germany
- ins: E. Voit
  name: Eric Voit
  org: Cisco Systems, Inc.
  abbrev: Cisco
  email: evoit@cisco.com
- ins: W. Pan
  name: Wei Pan
  org: Huawei Technologies
  abbrev: Huawei
  street: 101 Software Avenue, Yuhuatai District
  city: Nanjing, Jiangsu
  region: ''
  code: '210012'
  country: China
  phone: ''
  email: william.panwei@huawei.com

normative:
  I-D.ietf-rats-architecture: rats-arch
  I-D.ietf-rats-tpm-based-network-device-attest: rats-riv
  I-D.ietf-rats-yang-tpm-charra: rats-yang-tpm-charra
  I-D.ietf-rats-reference-interaction-models: rats-models
  RFC8639:
  TPM2.0:
    author:
      org: TCG
    title: "TPM 2.0 Library Specification"
    target: https://trustedcomputinggroup.org/resource/tpm-library-specification/
    seriesinfo:

informative:
  I-D.birkholz-rats-tuda: TUDA
  KGV:
    author:
      org: TCG
    title: "KGV"
    date: 2003-10
    target: https://trustedcomputinggroup.org/wp-content/uploads/TCG-NetEq-Attestation-Workflow-Outline_v1r9b_pubrev.pdf
    seriesinfo:

--- abstract

This memo defines how to subscribe to YANG Event Streams for Remote Attestation Procedures (RATS). In RATS, Conceptional Messages, are defined. Analogously, the YANG module defined in this memo augments the YANG module for TPM-based Challenge-Response based Remote Attestation (CHARRA) to allow for subscription to remote attestation Evidence. Additionally, this memo provides the methods and means to define additional Event Streams for other Conceptual Messages as illustrated in the RATS Architecture, e.g. Attestation Results, Endorsements, or Event Logs.

--- middle

# Introduction

{{-rats-riv}} and {{-rats-yang-tpm-charra}} define the operational prerequisites and a YANG Model for the acquisition of Evidence and other Conceptional Messages from a TPM-based network device. However, there are limitations inherent in the challenge-response based conceptual interaction model (CHARRA {{-rats-models}}) upon which these documents are based. One of these limitation is that it is up to a Verifier to request signed Evidence as provided by {{-rats-yang-tpm-charra}}, from a separate Attester which contains a TPM. The result is that the interval between the occurrence of a security-relevant change event, and the event's visibility within the interested RATS entity, such as a Verifier or a Relying Party, can be unacceptably long. It is common to convey Conceptual Messages ad-hoc or periodically via requests. As new technologies emerge, some of these solutions require Conceptual Messages to be conveyed from one RATS entity to another without the need of continuous polling. Subscription to YANG Notifications {{RFC8639}} provides a set of standardized tools to facilitate these emerging requirements. This memo specifies a YANG augment to subscribe to YANG modeled remote attestation Evidence as defined in {{-rats-yang-tpm-charra}}. Additionally, this memo provides the means to define further Event Streams to convey Conceptional Messages other than Evidence, such as Attestation Results, Endorsements, or Event Logs.

In essence, the limitation of poll-based interactions results in two adverse effects:

1. Conceptual Messages are not streamed to an interested consumer of information, e.g., Verifiers or Relying Parties, as soon as they are generated.

2. If they were to be streamed, Conceptual Messages are not appraisable for their freshness in every scenario. This becomes more important with Conceptional Messages that have a strong dependency on freshness, such as Evidence and corresponding Attestation Results.

This specification addresses the first adverse effect by enabling a consumer of Conceptual Messages (the subscriber) to request a continuous stream of new or updated Conceptual Messages via an {{RFC8639}} subscription to an \<attestation\> Event Stream. This new Event Stream is defined in this document and exists upon the producer of Conceptual Messages (the publisher). In the case of a Verifier's subscription to an Attester's Evidence, the Attester will continuously stream a requested set of freshly generated Evidence to the subscribing Verifier.

The second adverse effect results from the use of nonces in the challenge-response interaction model {{-rats-models}} realized in {{-rats-yang-tpm-charra}}. In {{-rats-yang-tpm-charra}}, an Attester must wait for a new nonce from a Verifier before it generates a new TPM Quote. To address delays resulting from such a wait, this specification enables freshness to be asserted asynchronously via the streaming attestation interaction model {{-rats-models}}. To convey a RATS Conceptual Message, an initial nonce is provided during the subscription to an Event Stream.

There are several options to refresh a nonce provided by the initial subscription or its freshness characteristics. All of these methods are out-of-band of an established subscription to YANG Notifications. Two complementary methods are taken into account by this memo:

1. a central provider supplies new fresh nonces, e.g. via a Handle Provider that distributes Epoch IDs to all entities in a domain as described in {{-rats-arch}} and as facilitated by the Uni-Directional Remote Attestation described in {{-rats-models}} or

2. the freshness characteristics of a received nonce are updated by -- potentially periodic or ad-hoc -- out-of-band TPM Quote requests as facilitated by {{-rats-yang-tpm-charra}}.

Both approaches to update the freshness characteristics of the Conceptual Messages conveyed via subscription to YANG Notification that are taken into account by this memo assume that clock drift between involved entities can occur. In consequence, in some usage scenarios the timing considerations for freshness {{-rats-arch}} might have to be updated in some regular interval. Analogously, there are can be additional methods that are not describe by but nevertheless supported by this memo.

This memo enables to remove the two adverse effects described by using the YANG augment specified. The YANG augment supports, for example, a RATS Verifier to maintain a continuous appraisal procedure of verifiably fresh Attester Evidence without relying on continuous polling.

# Terminology

The following terms are imported from {{-rats-arch}}: Attester, Conceptual Message, Evidence, Relying Party, and Verifier.  Also imported are the time definitions time(VG), time(NS), time(EG), time(RG), and time(RA) from that document's Appendix A.  The following terms are imported from {{RFC8639}}: Event Stream, Subscription, Event Stream Filter, Dynamic Subscription.

## Requirements Notation

{::boilerplate bcp14-tagged}

# Operational Model

{{-rats-riv}} describes the conveyance of TPM-based Evidence from a Verifier to an Attester using the CHARRA interaction model {{-rats-models}}. The operational model and corresponding sequence diagram described in this section is based on {{-rats-yang-tpm-charra}}. The basis for interoperability required for additional types of Event Streams is covered in {{otherstreams}}. The following sub-section focuses on subscription to YANG Notifications to the \<attestation\> Event Stream.

## Sequence Diagram

{{sequence}} below is a sequence diagram which updates Figure 5 of {{-rats-riv}}. This sequence diagram adapts {{-rats-riv}} by replacing the TPM-specific challenge-response interaction model with a {{RFC8639}} Dynamic Subscription to an \<attestation\> Event Stream. The contents of the \<attestation\> Event Stream are defined below within {{attestationstream}}.

~~~~
.----------.                            .--------------------------.
| Attester |                            | Relying Party / Verifier |
'----------'                            '--------------------------'
   time(VG)                                                    |
generateClaims(targetEnvironment)                              |
     | => claims, eventLogs                                    |
     |                                                         |
     |<---------establish-subscription(<attestation>)------time(NS)
     |                                                         |
   time(EG)                                                    |
generateEvidence(nonce, PcrSelection, collectedClaims)         |
     | => SignedPcrEvidence(nonce, PcrSelection)               |
     | => LogEvidence(collectedClaims)                         |
     |                                                         |
     |--filter(<pcr-extend>)---------------------------------->|
     |--<tpm12-attestation> or <tpm20-attestation>------------>|
     |                                                         |
     |                                                  time(RG,RA)
     |     appraiseEvidence(SignedPcrEvidence, eventLog, refClaims)
     |                                    attestationResult <= |
     |                                                         |
     ~                                                         ~
   time(VG')                                                   |
generateClaimes(targetEnvironment)                             |
     | => claims                                               |
     |                                                         |
   time(EG')                                                   |
generateEvidence(handle, PcrSelection, collectedClaims)        |
     | => SignedPcrEvidence(nonce, PcrSelection)               |
     | => LogEvidence(collectedClaims)                         |
     |                                                         |
     |--filter(<pcr-extend>)---------------------------------->|
     |--<tpm12-attestation> or <tpm20-attestation>------------>|
     |                                                         |
     |                                                 time(RG',RA')
     |    appraiseEvidence(SignedPcrEvidence, eventLog, refClaims)
     |                                    attestationResult <= |
     |                                                         |
~~~~
{: #sequence title="YANG Subscription Model for Remote Attestation"}

* time(VG,RG,RA) are identical to the corresponding time definitions from {{-rats-riv}}.

* time(VG',RG',RA') are subsequent instances of the corresponding times from Figure 5 in {{-rats-riv}}.

* time(NS) – the subscriber generates a nonce and makes an {{RFC8639}} \<establish-subscription\> request based on a nonce. This request also includes the augmentations defined in this document's YANG model. Key subscription RPC parameters include:
  * the nonce,
  * a set of PCRs of interest which the  wants to appraise, and
  * an optional filter which can reduce the logged events on the \<attestation\> stream pushed to the Verifier.

* time(EG) – an initial response of Evidence is returned to the Verifier. This includes:
  * a replay of filtered log entries which have extended into a PCR of interest since boot are sent in the \<pcr-extend\> notification, and
  * a signed TPM quote that contains at least the PCRs from the \<establish-subscription\> RPC are included in a \<tpm12-attestation\> or \<tpm20-attestation\>). This quote must have included the nonce provided at time(NS).

* time(VG',EG') – this occurs when a PCR is extended subsequent to time(EG). Immediately after the extension, the following information needs to be pushed to the Verifier:
  * any values extended into a PCR of interest,
  * a signed TPM Quote showing the result the PCR extension, and
  * and a handle (see Section 6. in {{-rats-models}}, which is either the initially received nonce or a more recently received Epoch ID (see Section 10.3. in {{-rats-arch}} that contains a new nonce or equivalent qualified data.

One way to acquire a new time synchronisation that allows for the reuse of the initially received nonce as a fresh handle is elaborated on in the follow section {{freshness-handles}}.

{: #freshness-handles "Continuously Verifying Freshness"}
## Continuously Verifying Freshness

As there is no new Verifier nonce provided at time(EG'), it is important to validate the freshness of TPM Quotes which are delivered at that time.  The method of doing this verification will vary based on the capabilities of the TPM cryptoprocessor used.

### TPM 1.2 Quote

The {{RFC8639}} notification format includes the \<eventTime\> object.  This can be used to determine the amount of time subsequent to the initial subscription each notification was sent.  However this time is not part of the signed results which are returned from the Quote, and therefore is not trustworthy as objects returned in the Quote.  Therefore a Verifier MUST periodically issue a new nonce, and receive this nonce within a TPM quote response in order to ensure the freshness of the results.  This can be done using the \<tpm12-challenge-response-attestation\> RPC from {{-rats-yang-tpm-charra}}.

### TPM 2 Quote

When the Attester includes a TPM2 compliant cryptoprocessor, internal time-related counters are included within the signed TPM Quote.  By including a initial nonce in the {{RFC8639}} subscription request, fresh values for these counters are pushed as part of the first TPM Quote returned to the Verifier. And then as shown by {{-TUDA}}, subsequent TPM Quotes delivered to the Verifier can the be appraised for freshness based on the predictable incrementing of these time-related countersr.

The relevant internal time-related counters defined within {{TPM2.0}} can be seen within \<tpms-clock-info\>.   These counters include the \<clock\>, \<reset-counter\>, and \<restart-counter\> objects.  The rules for appraising these objects are as follows:

* If the \<clock\> has incremented for no more than the same duration as both the \<eventTime\> and the Verifier's internal time since the initial time(EG) and any previous time(EG'), then the TPM Quote may be considered fresh. Note that {{TPM2.0}} allows for +/- 15% clock drift.  However many chips significantly improve on this maximum drift.  If available, chip specific maximum drifts SHOULD be considered during the appraisal process.

* If the \<reset-counter\>, \<restart-counter\> has incremented.  The existing subscription MUST be terminated, and a new \<establish-subscription\> SHOULD be generated.

* If a TPM Quote on any subscribed PCR has not been pushed to the Verifier for a duration of an Attester defined heartbeat interval, then a new TPM Quote notification should be sent to the Verifier.  This may often be the case, as certain PCRs might be infrequently updated.

~~~~
.----------.                        .--------------------------.
| Attester |                        | Relying Party / Verifier |
'----------'                        '--------------------------'
   time(VG',EG')                                         |
     |-<tpm20-attestation>------------------------------>|
     |                                    :              |
     ~                           Heartbeat interval      ~
     |                                    :              |
   time(EG')                              :              |
     |-<tpm20-attestation>------------------------------>|
     |                                                   |
~~~~


{: #attestationstream}
# Remote Attestation Event Stream

The \<attestation\> Event Stream is an {{RFC8639}} compliant Event Stream which is defined within this section and within the YANG Module of {{-rats-yang-tpm-charra}}. This Event Stream contains YANG notifications which carry Evidence to assists a Verifier in appraising the Trustworthiness Level of an Attester. Data Nodes within {{configuring}} allow the configuration of this Event Stream’s contents on an Attester.

This \<attestation\> Event Stream may only be exposed on Attesters supporting {{-rats-riv}}. As with {{-rats-riv}}, it is up to the Verifier to understand which types of cryptoprocessors and keys are acceptable.

## Subscription to the \<attestation\> Event Stream

To establish a subscription to an Attester in a way which provides provably fresh Evidence, initial randomness must be provided to the Attester. This is done via the augmentation of a \<nonce-value\> into {{RFC8639}} the \<establish-subscription\> RPC. Additionally, a Verifier must ask for PCRs of interest from a platform.

~~~~
  augment /sn:establish-subscription/sn:input:
    +---w nonce-value    binary
    +---w pcr-index*     tpm:pcr
~~~~

The result of the subscription will be that passing of the following information:

1. \<tpm12-attestation\> and \<tpm20-attestation\> notifications which include the provided \<nonce-value\>.  These attestation notifications MUST at least include all the \<pcr-indicies\> requested in the RPC.

2. a series of \<pcr-extend\> notifications which reference the requested PCRs on all TPM based cryptoprocessors on the Attester.

3. \<tpm12-attestation\> and \<tpm20-attestation\> notifications generated within a few seconds of the \<pcr-extend\> notifications.  These attestation notifications MUST at least include any PCRs extended.

If the Verifier does not want to see the logged extend operations for all PCRs available from an Attester, an Event Stream Filter should be applied.  This filter will remove Evidence from any PCRs which are not interesting to the Verifier.

## Replaying a history of previous TPM extend operations

Unless it is relying on Known Good Values, a Verifier will need to acquire a history of PCR extensions since the Attester has been booted.  This history may be requested from the Attester as part of the \<establish-subscription\> RPC.  This request is accomplished by placing a very old \<replay-start-time\> within the original RPC request.  As the very old \<replay-start-time\> will pre-date the time of Attester boot, a \<replay-start-time-revision\> will be returned in the \<establish-subscription\> RPC response, indicating when the Attester booted.  Immediately following the response (and before the notifications above)  one or more \<pcr-extend\> notifications which document all extend operations which have occurred for the requested PCRs since boot will be sent.  Many extend operations to a single PCR index on a single TPM SHOULD be included within a single notification.

Note that if a Verifier has a partial history of extensions, the \<replay-start-time\> can be adjusted so that known extensions are not forwarded.

The end of this history replay will be indicated with the {{RFC8639}} \<replay-completed\> notification.  For more on this sequence, see Section 2.4.2.1 of {{RFC8639}}.

After the \<replay-complete\> notification is provided, a TPM Quote will be requested and the result passed to the Verifier via a \<tpm12-attestation\> and \<tpm20-attestation\> notification.  If there have been any additional extend operations which have changed a subscribed PCR value in this quote, these MUST be pushed to the Verifier before the \<tpm12-attestation\> and \<tpm20-attestation\> notification.

At this point the Verifier has sufficient Evidence appraise the reported extend operations for each PCR, as well compare the expected value of the PCR value against that signed by the TPM.

### TPM2 Heartbeat

For TPM2, make sure that every requested PCR is sent within an \<tpm20-attestation\> no less frequently than once per heartbeat interval.   This MAY be done with a single \<tpm20-attestation\> notification that includes all requested PCRs every heartbeat interval.  This MAY be done with several \<tpm20-attestation\> notifications at different times during that heartbeat interval.

## YANG notifications placed on the \<attestation\> Event Stream

### pcr-extend

This notification documents when a subscribed PCR is extended within a single TPM cryptoprocessor.  It SHOULD be emmitted no less than the \<marshalling-period\> after an the PCR is first extended.  (The reason for the marshalling is that it is quite possible that multiple extensions to the same PCR have been made in quick succession, and these should be reflected in the same notification.)  This notification MUST be emmitted prior to a \<tpm12-attestation\> or \<tpm20-attestation\> notification which has included and signed the results of any specific PCR extension.   If pcr extending events occur during the generation of the \<tpm12-attestation\> or \<tpm20-attestation\> notification, the marshalling period MUST be extended so that a new \<pcr-extend\> is not sent until the corresponding notifications have been sent.

~~~~
{::include ietf-tpm-remote-attestation-stream_pcr-extend.tree}
~~~~

Each \<pcr-extend\> MUST include one or more values being extended into the PCR.   These are passed within the \<extended-with\> object.  For each extension, details of the event SHOULD be provided within the \<event-details\> object.
The format of any included \<event-details\> is identified by the \<event-type\>.  This document includes two YANG structures which may be inserted into the \<event-details\>.  These two structures are: \<ima-event-log\> and \<bios-event-log\>.  Implementations wanting to provide additional documentation of a type of PCR extension may choose to define additional YANG structures which can be placed into \<event-details\>.

### tpm12-attestation

This notification contains an instance of a TPM1.2 style signed cryptoprocessor measurement. It is supplemented by Attester information which is not signed. This notification is generated and emitted from an Attester when at least one PCR identified within the subscribed \<pcr-indices\> has changed from the previous \<tpm12-attestation\> notification.  This notification MUST NOT include the results of any PCR extensions not previously reported by a \<pcr-extend\>.  This notification SHOULD be emitted as soon as a TPM Quote can extract the latest PCR hashed values.  This notification MUST be emitted prior to a subsequent \<pcr-extend\>.

~~~~
{::include ietf-tpm-remote-attestation-stream_tpm12-attestation.tree}
~~~~

All YANG objects above are defined within {{-rats-yang-tpm-charra}}.  The \<tpm12-attestation\> is not replayable.

### tpm20-attestation

This notification contains an instance of TPM2 style signed cryptoprocessor measurements. It is supplemented by Attester information which is not signed. This notification is generated at two points in time:

* every time at least one PCR has changed from a previous tpm20-attestation. In this case, the notification SHOULD be emitted within 10 seconds of the corresponding \<pcr-extend\> being sent:

* after a locally configurable minimum heartbeat period since a previous tpm20-attestation was sent.

~~~~
{::include ietf-tpm-remote-attestation-stream_tpm20-attestation.tree}
~~~~

All YANG objects above are defined within {{-rats-yang-tpm-charra}}.  The \<tpm20-attestation\> is not replayable.

## Filtering Evidence at the Attester

It can be useful *not* to receive all Evidence related to a PCR.  An example of this is would be a when a Verifier maintains known good values of a PCR.  In this case, it is not necessary to send each extend operation.

To accomplish this reduction, when an RFC8639 \<establish-subscription\> RPC is sent, a \<stream-filter\> as per RFC8639, Section 2.2 can be set to discard a \<pcr-extend\>  notification when the \<pcr-index-changed\> is uninteresting to the verifier.

## Replaying previous PCR Extend events

To verify the value of a PCR, a Verifier must either know that the value is a known good value {{KGV}} or be able to reconstruct the hash value by viewing all the PCR-Extends since the Attester rebooted. Wherever a hash reconstruction might be needed, the \<attestation\> Event Stream MUST support the RFC8639 \<replay\> feature. Through the \<replay\> feature, it is possible for a Verifier to retrieve and sequentially hash all of the PCR extending events since an Attester booted. And thus, the Verifier has access to all the evidence needed to verify a PCR’s current value.

{: #configuring "Configuring the Attestation Stream"}
## Configuring the \<attestation\> Event Stream

{{attestationconfig}} is tree diagram which exposes the operator configurable elements of the \<attestation\> Event Stream. This allows an Attester to select what information should be available on the stream. A fetch operation also allows an external device such as a Verifier to understand the current configuration of stream.

Almost all YANG objects below are defined via reference from {{-rats-yang-tpm-charra}}. There is one object which is new with this model however. \<tpm2-heartbeat\> defines the maximum amount of time which should pass before a subscriber to the Event Stream should get a \<tpm20-attestation\> notification from devices which contain a TPM2.

~~~~
{::include ietf-tpm-remote-attestation-stream_attestation-config.tree}
~~~~
{: #attestationconfig title="Configuring the \<attestation\> Event Stream"}

{: #YANG-Module}
# YANG Module

This YANG module imports modules from {{-rats-yang-tpm-charra}} and {{RFC8639}}.  It is also work-in-progress.

~~~~ YANG
<CODE BEGINS> ietf-rats-attestation-stream@2024-07-06.yang
{::include ietf-tpm-remote-attestation-stream@2020-12-15.yang}
<CODE ENDS>
~~~~

{: #otherstreams}
# Event Streams for Conceptual Messages

Analogous to the {{RFC8639}} compliant \<attestation\> Event Stream for the conveyance of remote attestation Evidence as defined in Section {{attestationstream}}, additional Event Streams can be defined for this YANG augment. Additional Event Streams require separate YANG augment specifications that provide the Event Stream definition and optionally a content format definition either via subscriptions to YANG datastores or dedicated YANG Notifications. It is possible to use either YANG subscription methods to other YANG modules for RATS Conceptual Messages or to define Event Streams for other none-YANG-modeled data. In the context of RATS Conceptual Messages, both options MUST be a specified via YANG augments to this specification.

# Security Considerations

To be written.

# IANA Considerations {#IANA}

To be written.

--- back

# Change Log

v00-v01

* minor updates, party based on the dependent Charra going through IESG.


# Acknowledgements
{: numbered="no"}

Thanks to ...
