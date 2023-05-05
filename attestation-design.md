# Incorporating Explicit and Implicit Attestations into C2PA Manifests
In this section we discuss the options for incorporating attestations into C2PA as well as the considerations that led to proposed design.  Following this discussion, the specific normative requirements for incorporating explicit and impicit attestations into a C2PA manifest are defined.

## Explicit Attestation

The current C2PA trust model is built using two signatures:

1.	A signature over a CBOR-serialized Claim.  The signature is encoded in the manifest, and the certificate chain for the signing key is usually also included.
1.	Optionally, a countersignature from an RFC 3161-compliant time stamping service.

Attestations, in the context of this document, are also digital signatures.  This section describes options for how the additional signature can be added to the two that are already defined.  Note that the discussion in this section is simplified: later sections present the complete design and requirements. 

In the following, when we say *"attestation over X"* we mean that the attestation is a signature over the hash of the serialization of X.  

The simplest option for adding an attestation would be to include it as a second countersignature over the serialized claim.  However, with this design, the claim signature or the attestation can be independently removed and replaced. Whether these are troublesome attacks will depend on scenario (and the sophistication of the relying party) which makes a thorough security analysis difficult. However, the following proposal prevents the attestation and claim signatures being independently replaced with only modest complexity.  

To achieve the claim-signature/attestation binding, an attestation over the serialized claim is performed first and then the claim signature is performed over the attestation.  This prevents the attestation being removed independently of the claim signature because any modification of the attestation will invalidate the claim signature  

To prevent the claim signature being replaced, the attestation contains the public key of the subsequent claim signer.  This means that validators can detect and reject a claim signature if the claim signature key specified in the attestation does not match the keys used for the actual claim signature.

The binding is illustrated in the following figure.  The attestation is bound to the claim so the claim cannot be edited without the attestation being invalidated.  Similarly, the claim signature is bound to the attestation and the claim: again, neither the attestation or the claim signature can be edited without invalidating the claim signature.  

Finally, the attestation contains the claim creator public key, so the claim signature cannot be replaced by a different claim signer without the attestation being invalidated. 

``` mermaid
graph RL;
    id1(Claim Signature) --> id2(Attestation) --> id3(Claim) --> id4(Asset);
    id2 --> id1;
    id1 --> id3;
```
*Figure X. Cryptographic dependencies for an Asset, Claim, Attestation and Claim, Signature.  An arrow pointing from A to B means that B cryptographically depends on A, so A is integrity protected.*

### What is Attested?
Claim Signatures are calculated over the serialization of a claim, and the claim itself is cryptographically linked to its embedded assertions.  Two types of assertion are important here: assertions that contain metadata, and assertions that bind a Claim to an Asset with a hash or hashes of an asset or asset fragment.  By signing the Claim, the Claim Creator vouches for both the asset (via the binding assertion) and the metadata.

We desire similar bindings for the explicit attestation signature. Most or all scenarios benefit from binding to the asset (e.g., for the Trusted Camera Application, *“this image was captured by this program running on this device.”*)  Some scenarios will benefit from binding to assertions (e.g., for a Trusted Camera Application *“the GPS coordinates obtained from the radio when this image was captured”* or for the ML scenario, *“the following objects were recognized in the image.”*)  However, note that attestation will *not* improve trust for all types of assertion data: for example, user-input is hearsay as far as the attesting application is concerned.   

The approach we have taken is that the attestation is over the serialization of the entire Claim – i.e., the same data structure that the Claim Generator signs (with one exception, which we describe below.)  As noted above, not all assertions/metadata gain security benefit from the attestation, but signing all assertions provides the most flexibility for future scenarios. For future advanced use cases, we define a field that an attestation generator can use to indicate which assertions are meaninfully “attested.” However, the use and interpretation of this field is beyond the current scope of C2PA.

### Embedding an Explicit Attestation in a Manifest
Explicit attestations are similar to Claim Signatures and could be encoded in a manifest similarly.  However, we also have the additional security requirement noted earlier: i.e., that the Claim Signer should also sign the attestation result.

A natural way of encoding the attestation so that it is signed by the Claim Signer is to embed it as an assertion in the Claim.  If we do this, then the attestation will naturally be signed in the same way as any other sort of assertion.  However, naïve attempts at implementing this will fail because we are introducing a circular dependency: the Claim cannot be finalized before the attestation is created, but the attestation requires a finalized Claim in order to calculate the Claim hash.

We choose to break this dependency by specifying that the attestation is created over the hash of the serialized claim but **omitting the attestation assertion** (which has not been created yet.)  Once the attestation has been created, it is embedded in the Manifest and Claim identically to any other assertion and will subsequently be signed by the Claim Creator.  The Claim with one or more attestation assertions elided is called a Partial Claim.

This architecture is attractive because normal claim creation and claim validation are unaffected.  It is also attractive because an attestation-enhanced claim can be processed by an attestation-unaware validator without changes (the attestation assertion can be treated like any other third-party or unrecognized assertion.)

The (simplified) logical flow for creating a manifest with an attestation is illustrated in Figure 1. 

>> Add Figure here

*Figure XX: : Simplified steps for creating a C2PA manifest and a manifest containing an attestation assertion.*
*In the no-attestation case, assertions are created and boxed in the assertion store.  A Claim is then prepared, with the assertions array set to the location and hash of the referenced assertions. The Claim is then serialized, hashed, and signed by the Claim Creator.  The Assertion Store, Claim, and Claim Signature are then packaged as a manifest (not shown.)*
*To support attestations, the additional steps in the shaded box are required. As before, an Assertion Store is populated with the desired assertions, but we call it a Partial Assertion Store because one additional assertion will be added before the Claim is finalized.  The Partial Claim is then serialized and claim hash is then attested using the appropriate platform attestation service. Next, the attestation is packaged as an assertion and added to the Partial Assertion Store to create the finalized Assertion Store.  Finally, the serialized-Claim (with the embedded attestation assertion) is signed by the Claim Creator.*

Note that this architecture *does* demand that attestation-aware Validators perform additional steps: Specifically, Validators must edit the Claim to remove any attestation assertions, and then re-serialize the resulting Partial Claim to validate the binding in the attestation assertion.  We consider this tradeoff to be acceptable: the rewriting cost is borne by the creators and consumers of attestation, but the creation and validation workflow for non-attestation-aware entities is unchanged.

Additional details are included in the normative parts of the specification below.

### Multiple Explicit Attestations
This specification supports more than one attestation for a claim – for example, there may be one attestation for code running in an enclave, and a second for the overall platform (the “rich OS.”)

If more than one attestation is required, then they are ordered, and each subsequent attestation is over the Partial Claim containing the prior attestations.  For example, if two attestations are required, the first attestation to be added will be over the Partial Claim with no attestation assertions.  The first attestation assertion is then added to the Claim, and the second attestation will be over the claim with just the first attestation assertion included.  Of course, the second attestation assertion is then added, and the resulting finalized Claim is signed by the Claim Creator.

Attestors that wish to include multiple attestations are free to decide the order that they are created and embedded, but since later attestations include earlier attestations, the validation order must be the same as the creation order. To ensure that that this occurs, this specification requires that attestations are embedded in the assertions array in the order that they are created.

## Normative Requirements for Explicit Attestations

### Data Structures

#### Partial Claim Definition
A Partial Claim is a C2PA `claim-map` but omitting any Attestation Assertions. (Note: for simplicity we say that an assertion is “included” or “omitted”, but what is actually added or omitted from the Claim is actually a `hashed-uri-map` link in the `assertions` array.)  Partial Claims use the same `claim-map` data structure as a standard Claim: a Partial Claim only differs in that one or more Attestation Assertions are removed from the `assertions` array.

Attestation Assertions can be included at any location in the assertion array, although placing them last in the assertions array simplifies processing and is therefore preferred.

During Claim creation, the attestations are calculated then embedded one at a time.  For example, the Partial Claim without attestation assertions is created, serialized, and then the first attestation is gathered.  The resulting attestation result is then encoded as an `attestation-info-map` assertion and then added to the end of the `assertions` array in the Claim.  

If a second attestation is required, the Partial Claim with the first Attestation Assertion included is serialized and the second attestation is performed.  The second Attestation Assertion is encoded in a `attestation-info-map` then added to the end of the `assertions` array to form the final Claim, which is then signed by the Claim Creator.  

This specification does not restrict the location in the `assertions` array where the Attestation Assertions are added,[^1] although, as previously noted, incorporating them last in the array is preferred). However, the order in the `assertions` array is important when more than one Attestation Assertion is incorporated.  In this case, higher-index Attestation Assertions are performed after lower-index entries.

[^1]: We do not demand that attestation assertions appear last because doing so would limit the future evolution of the C2PA standard.

#### Attestation "To-Be-Signed" Definition
The `attestation-tbs-map` (attestation to-be-signed map) is an envelope data structure for the information that is to be serialized, hashed, and signed by the attestation machinery. The most important field in `attestation-tbs-map` is the `partial-claim-hash`: essentially the hash of the asset and its referenced assertions.

[source,cddl]
----
include::attestation-tbs-map.cddl
----

| Field | Description |
|-------|---------|
| `partial-claim-hash` | The hash of a partial claim – i.e., a Claim that omits all or some of the Attestation Assertions. If the Claim contains a single Attestation Assertion, then this will be the hash of the CBOR-serialized claim omitting the Attestation Assertion – i.e., usually the final entry in the `assertions` array. If the claim contains *n* Attestation Assertions, then *n* Partial Claim hashes are defined.| 
| `alg` | The hash algorithm  used to compute `partial-claim-hash`. |
| `pub-key` | (Optional) The DER-encoded public key and algorithm of the claim signer (the Subject Public Key Info) as specified in RFC***. |
| `created` | (Optional) The UTC time that the attestation was created. Can be used to reduce the risk that the same claim signer can later add a different claim signature. |
| `other-tbs-info` | (Optional) Additional information that the Claim Creator wishes to associate with the attestation. |
| `other-tbs-info-2` | (Optional) Additional information that the Claim Creator wishes to associate with the attestation. |


#### Attestation Assertion Definition
`attestation-info-map` is the envelope data structure that contains the attestation signature and related information.  It is used to embed an Attestation Assertion in the Assertion Store.

`attestation-info-maps` are boxed with label `c2pa.attestation` (or `c2pa.attestation`, `c2pa.attestation_001`, `c2pa.attestation_002`.)

Attestation Assertions can be placed at any index in the `assertions` array but adding them as the final entries in the order that they are created is preferred.

The `attestation-info-map` definition presented here can encode many types of attestation. See Appendix * for currently defined attestation encodings.

[source,cddl]
----
include::attestation-info-map.cddl
----

| Field | Description |
|-------|---------|
| `att-type` | The attesation type. E.g. `c2pa.SGX`, See Appendix A for currently defined types. If non-standard attestations are encoded, the same convention for non-standard Assertion labels should be used.  E.g., `com.litware.custom-assertion` |
| `attestation-tbs` | The `attestation-tbs-map` used when the assertion was obtained. | 
| `attestation-results` | Binary encoding of the attestation measurement. |
| `creation-time` | (Optional) UTC time when the attestation was created. |
| `other-info` | (Optional) Any additional information that the Claim Creator wishes to associate wsith the attestation. |
| `other-info-2` | (Optional) Any additional information that the Claim Creator wishes to associate wsith the attestation. |

## Creating a Claim Containing One or More  Explicit Attestations 
These are the detailed steps for creating an embedding an Attestation in a Claim. In the 1.3 version of the TCPA specification, these steps should be performed immediately prior to *“11.3.2.4. Signing a Claim”*.  Following these steps, the Claim should be signed as described int the main specification.

1. Create a copy of the `claim-map` containing all non-attestation assertions.  
1. Calculate the hash of the serialized (Partial) `claim-map` to form *hash(claim-map)*.
1. Create an `attestation-tbs-map` data structure, including *hash(claim-map)*, the time, and (optionally) the public key of the Claim Signer.  Add any desired optional data to `attestation-tbs-map`.  CBOR-serialize and hash to form *hash(attestation-tbs-map)*.
1. Use the appropriate platform service to obtain an attestation over *hash(attestation-tbs-map)* to form attestation *attest(hash(`attestation-tbs-map`))*
1. Create a new `attestation-info-map`, then:
   1. Set `att-type` to the type-selector for the attestation performed (Appendix A)
   1. Set `attestation-tbs` to `attestation-tbs-map`
   1. Set `attestation-results` to *attest(hash(attestation-tbs-map))*.  Defined types and encodings are specified in Appendix A.
1. Create a JUMBF attestation container with label `c2pa.attestation` (or `c2pa.attestation_00n`, if this is not the first attestation added) containing the  `attestation-info-map` being prepared.
1. Add the `attestation-info-map` to the Assertion Store.
1. Add a new `hashed-uri-map` to the `assertions` array in `claim-map`, referencing the newly added Attestation Assertion. 
1. If more than one attestation is required, go back to (2), but start with the current Partial `claim-map` containing the attestations calcuated thus far.

At this point, the final claim has been created, and processing can continue according to *“11.3.2.4. Signing a Claim”* using the `claim-map` with all embedded Attestation Assertions.

Note that this generic flow does not specify how an attestation is encoded into `attestation-results`.  This is scheme-specific and is described in Appendix A.

## Validating a Claim Containing Explicit Attestations
This section describes how Claims containing expolicit attestations should be validated. This section also describes how non-attestation aware validators should behave: this text will be replicated in the main C2PA technical specification.

### Attestation Aware Validator
PRELIMINARY: MORE DETAIL NEEDED.

To perform attestation validation, first all Attestation Assertions are removed from the `assertions` array in the Claim.  The first Attestation Assertion is then checked against the hash of the CBOR-serialized Partial Claim with all Attestation Assertions removed.  If this succeeds, then the first Attestation Assertion is added back into the `assertions` array, the resulting Partial Claim is re-serialized, and the second Attestation Assertion is validated, and so on.  Note that for validation to succeed, the Attestation Assertions must be re-added in the same location in the assertions array.  This is simpler if they are last in the `assertions` array, which is the reason for this recommendation.  However, attestation validators should be able to process Attestation Assertions in any location in the array.

Note that when an Attestation Assertion is removed, the Claim is serialized using normal CBOR rules.  For example, if a claim included two standard assertions and one Attestation Assertion, then the Partial Claim will contain an `assertions` array containing two elements. If a Partial Claim contains two standard assertions and two Attestation Assertions, then two Partial Claims are defined: one omitting just the last Attestation Assertion in the `attestations` array, and one omitting both Attestation Assertions.  Of course, the order of the Attestation Assertions must be preserved as the Partial Claim is created and validated.

### Non-Attestation Aware Validator
(This text belongs in the main specification.)

This text will be added to the current main-specification validation section. It ensures that assets including attestation assertions compliant with this specification are not rejected. 

*“For validators that do not consume attestations, any assertion with label starting with `c2pa.attestation` should be ignored. Third party unrecognized attestations, including third-party attestation assertions, are ignored as specified elsewhere.”*

# Implicit Attestation
This section describes the design considerations and normative requirements for incoporating implicit attestations into the C2PA.

Implicit Attestation, sometimes called Key Attestation, in the context of this specification, is the use of a standard C2PA Claim Signature using a key that can only be used by an authorized application or applications.  If a key can only be used by the authorized application, then the presence of a well-formed Claim Signature implies that the authorized application signed the claim: i.e. the claim has been *implicitly attested*.  Such keys are called Implicit Attestation Claim Signing keys, or IACS-keys.

The current (v1.3) version of the C2PA supports Implicit Attestation without changes:

*Identity of Signers 
The identity of a signatory is not necessarily a human actor, and the identity  presented may be a pseudonym, completely anonymous, or pertain to a service or trusted hardware device with its own identity, including an application running inside such a service or trusted hardware.*

This specification provides guidance on the issuance and lifetime management of claim-signing keys for implicit attestation, with appropriate consideration of privacy issues.*

## Issuing / Certifying Keys for Implicit Attestation 
Devices that support attestation provide cryptographic building blocks that can be used in protocols to prove the identity of the device and running software to relying parties. One common cryptographic primitive is called quote. Quotes are typically digital signatures over user-supplied data and platform/quote-engine-supplied data that describes the running program and the environment/TCB in which it is executing.

Quotes (and related primitives) can be used to create Explicit Attestations (section *) but can also be used in a protocol to provision or certify a Claim Signing key.  A common building-block is that an application creates a keypair and “Quotes” the resulting public key to a trusted service.  The service checks that the device is known or in good-standing, and that the program measurements conveyed in the quote are in policy.  If the checks succeed, the service creates a certificate for the newly created implicit attestation public key.  

Once provisioned, the Claim Signer uses this key and certificate in exactly the same way that other (non-implicit-attestation) Claim Signing keys are used.

Note that this description is simplfied: implementers should consult platform documentation for recommended provisioning protocols for implicit attestation keys.

## IACS-Key Certificates

The certificate for the implicit attestation claim signing key should comply with the requirements in the main C2PA specification.  Optionally, the certificate may contain additional fields that convey additional information about the application or security posture of the device to which it was provisionied.

IACS-keys and cerfificates are associated with devices, as opposed to people or organizations.  Claim Creators may use the `CreativeWork` assertion to encode a person or organization. 

## Protecting Implicit Attestation Signing Keys 
Most platforms that provide attestation capabilities also provide security primitives to allow a system to protect stored keys and other data so that the data is only accessible to the attesting application, or other applications explicitly authorized by the attesting application.  This operation is commonly called sealing.

Implementers should consult platform documentation for usage.

## Timeline for Provisioning and Use of Implicit Attestation Signing Keys

Summarizing the previous sections in the form of a timeline, the following steps are required when provisioning and using a key for implicit attestation.

1. It is assumed that devices are provisioned with a platform key and certificate (or a key and a pre-established trust relationship with a server.) This is usually performed during manufacture.
1. At some point in the life of the device, the C2PA-compliant Claim Creation application is installed.
1. During installation (or later), the Claim Creator will interact with a trusted service to create a certified key for signing claims.  The Claim Creator uses Quotes (or similar) to prove that it is an authorized application running on a trusted device. The Quote directly or indirectly uses the platform key in (1).
1. The certified IACS-key and certificate can be used immediately to sign Claims.
1. If the platform provides suitable *sealing* facilities, the IACS-private key can be persisted in such a way that only the authorized application can retrieve it.
   1. If this is the case, then the IACS-key can be stored and protected so that the application be closed and restarted and use the same key
   1. If this is not the case, then the IACS-key should not be persisted: instead, the application should follow stel (3) to obtain a new IACS-key on each startup.

## Claim Creation with Implicit Attestation

Claim Creators use IACS-keys to sign Claims in exactly the same way as documented in the main C2PA specification for keys associated with organizations or people.

## Claim Validation of Manifests Using Implicit Attestation

Claims signed with IACS-keys are validated identically to validation of manifests signed with keys associated with organizations or people.

We expect that IACS-keys will employ specialized CAs and Trust Roots, although this is not required.
