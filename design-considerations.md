## Design Considerations
In this section we discuss the options for incorporating attestations into C2PA as well as the considerations that led to proposed design.  The sections that follow 
contain the normative requirements for incorporating explicit and impicit attestations into a C2PA manifest.

### Explicit Attestation

The current C2PA trust model is built using two signatures:

1)	A signature over a CBOR-serialized Claim.  The signature is encoded in the manifest, and the certificate chain for the signing key is usually also included.
2)	Optionally, a countersignature from an RFC 3161-compliant time stamping service.

Attestations, in the context of this document, are also digital signatures.  This section describes options for how the additional signature can be added to the 
two that are already defined.  Note that the discussion in this section is simplified: a more complete description follows. 

In the following, when we say *"attestation over X"* we mean that the attestation is a signature over the hash of the serialization of X.  

The simplest option for adding an attestation would be to include it as a second countersignature over the serialized claim.  However, with this design, the 
claim signature or the attestation can be independently removed and replaced.
Whether these are troublesome attacks will depend on scenario (and the sophistication of the relying party) which makes a thorough security analysis difficult.
However, the following proposal prevents the attestation and claim signatures being independently replaced with only modest complexity.  

To achieve the claim-signature/attestation binding, an attestation over the serialized claim is performed first and then the claim signature is performed over the attestation. 
This prevents the attestation being removed independently of the claim signature because any modification of the attestation will incalidate the claim signature  

To prevent the claim signature being replaced, the attestation contains the public key of the subsequent claim signer.  This means that 
validators can detect and reject a claim signature if the claim signature key specified in the attestation does not match the keys used for the actual claim signature.

The binding is illustrated in the following figure.  The attestation is bound to the claim so the claim cannot be edited without the attestation being invalidated.  
Similarly, the claim signature is bound to the attestation and the claim: again, neither the attestation or the claim signature can be edited without invalidating the claim signature.  
Finally, the attestation contains the claim creator public key, so the claim signature cannot be replaced by a different claim signer without the attestation being invalidated. 

``` mermaid
graph RL;
    id1(Claim Signature) --> id2(Attestation) --> id3(Claim) --> id4(Asset);
    id2 --> id1;
    id1 --> id3;
```
Figure X. Cryptographic dependencies for an asset, claim, attestation and claim, signature.  An arrow pointing from A to B means that B is cryptographically linked to A.

## What is Attested?
Claim Signatures are calculated over the serialization of a claim, and the claim itself is cryptographically linked to assertions.  Two types of assertion are important here: assertions that contain metadata, and assertions that bind a claim to an asset with a hash or hashes of an asset or asset fragment.  By signing the Claim, the claim creator vouches for both the asset and the metadata.

We desire similar bindings for the attestation signature: Most or all scenarios benefit from binding to the asset (e.g., for the Trusted Camera Application, “this image was captured by this program running on this device.”)  Some scenarios will benefit from binding to assertions (e.g., for a Trusted Camera Application “the GPS coordinates obtained from the radio when this image was captured” or for the ML scenario, “the following objects were recognized in the image.”)  However, note that attestation will *not* improve trust for all types of assertion data: for example, user-input is hearsay as far as the attesting application is concerned.

The approach we have taken is that the attestation is over the serialization of the entire claim – i.e., the same data structure that the claim generator signs (with one exception, which we describe below.)  As noted above, not all assertions/metadata gain security benefit from attestation, but signing all assertions provides the most flexibility for future scenarios.    For such advanced use cases, we define a field that an attestation generator can use to indicate which assertions are “attested.” However, the use and interpretation of this field is beyond the current scope of C2PA.

## Embedding an Explicit Attestation in a Manifest
Attestations are like claim signatures and could be encoded in a manifest similarly.  However, we also have the additional security requirement noted earlier: i.e., that the claim-signer should also sign the attestation result.

A natural way of encoding the attestation so that it is signed by the claim signer is to embed it as an assertion in the claim.  If we do this, then the attestation will naturally be signed in the same way as any other sort of assertion.  However, naïve attempts at implementing this will fail because we are introducing a circular dependency: the claim cannot be finalized before the attestation is created, but the attestation requires a finalized claim in order to calculate the claim hash.

We choose to break this dependency by specifying that the attestation is created over the hash of the serialized claim but omitting the attestation assertion (which has not been created yet.)  Once the attestation has been created, it is embedded in the manifest and claim identically to any other assertion and will subsequently be signed by the claim generator.  The claim with one or more attestation assertions elided is called a partial claim.

This architecture is attractive because normal claim creation and claim validation are unaffected.  It is also attractive because an attestation-enhanced claim can be processed by an attestation-unaware validator without changes (the attestation assertion can be treated like any other third-party or unrecognized assertion.)
The (simplified) logical flow for creating a manifest with an attestation is illustrated in Figure 1. 

>> Add Figure here

*Figure XX: : Simplified steps for creating a C2PA manifest and a manifest containing an attestation assertion.*
*In the no-attestation case, assertions are created and boxed in the assertion store.  A Claim is then prepared, with the assertions array set to the location and hash of the referenced assertions. The Claim is then serialized, hashed, and signed by the Claim Creator.  The Assertion Store, Claim, and Claim Signature are then packaged as a manifest (not shown.)*
*To support attestations, the additional steps in the shaded box are required. As before, an Assertion Store is populated with the desired assertions, but we call it a Partial Assertion Store because one additional assertion will be added before the Claim is finalized.  The Partial Claim is then serialized and claim hash is then attested using the appropriate platform attestation service. Next, the attestation is packaged as an assertion and added to the Partial Assertion Store to create the finalized Assertion Store.  Finally, the serialized-Claim (with the embedded attestation assertion) is signed by the Claim Creator.*

Note that this architecture does demand that attestation-aware validators perform additional steps: Specifically, validators must edit the claim to remove any attestation assertions, and then re-serialize the resulting partial claim to validate the binding in the attestation assertion.  We consider this tradeoff to be acceptable: the cost is borne by the creators and consumers of attestation, but the creation and validation workflow for non-attestation-aware entities is unchanged.

Additional details are included in the normative parts of the draft specification below.

## Multiple Explicit Attestations
This specification supports more than one attestation for a claim – for example, there may be one attestation for code running in an enclave, and a second for the overall platform (the “rich OS.”)

If more than one attestation is required, then they are ordered, and each subsequent attestation is over the partial claim containing the prior attestations.  For example, if two attestations are required, the first attestation to be added will be over the partial claim with no attestation assertions.  The first attestation assertion is then added to the claim, and the second attestation will be over the claim with just the first attestation assertion included.  Of course, the second attestation assertion is then added, and the resulting finalized claim is signed by the claim generator.

Attestors that wish to include multiple attestations are free to decide the order that they are created and embedded, but since later attestations include earlier attestations, the validation order must be the same as the creation order. To ensure that that this occurs, this specification requires that attestations are embedded in the assertions array in the order that they are created.

## Normative Requirements for Explicit Attestations

### Data Structures

#### Partial Claim Definition
A Partial Claim is a C2PA `claim-map` but omitting any Attestation Assertions. (Note: for simplicity we say that an assertion is “included” or “omitted”, but what is actually added or omitted from the Claim is actually a `hashed-uri-map` link in the `assertions` array.)  Partial Claims use the same `claim-map` data structure as a standard Claim: a Partial Claim only differs in that one or more Attestation Assertions are removed from the `assertions` array.

Attestation Assertions can be included at any location in the assertion array, although placing them last in the assertions array simplifies processing and is preferred.

During Claim creation, the attestations are calculated then embedded one at a time.  For example, the Partial Claim without attestation assertions is created, serialized, and then the first attestation is gathered.  The resulting attestation result is then encoded as an `attestation-info-map` assertion and then added to the end of the `assertions` array in the Claim.  If a second attestation is required, the Partial Claim with the first Attestation Assertion included is serialized and the second attestation is performed.  The second Attestation Assertion is encoded in a `attestation-info-map` then added to the end of the `assertions` array to form the final Claim, which is then signed by the Claim Creator.  This specification does not restrict the location in the `assertions` array where the Attestation Assertions are added, although, as previously noted, incorporating them last in the array is preferred). However, the order in the `assertions` array is important when more than one Attestation Assertion is incorporated.  In this case, higher-index Attestation Assertions are performed after lower-index entries.

#### Attestation "To-Be-Signed" Definition
The `attestation-tbs-map` (attestation to-be-signed map) is an envelope data structure for the information that is to be serialized, hashed, and signed by the attestation machinery. The most important field in the `attestation-tbs-map` is the `partial-claim-hash`: essentially the hash of the asset and its referenced assertions.

[source,cddl]
----
include::attestation-tbs-map.cddl
----


#### Attestation Assertion Definition
`attestation-info-map` is the envelope data structure that contains the attestation signature and related information.  It is used to embed an Attestation Assertion in the Assertion Store.

`attestation-info-maps` are boxed with label `c2pa.attestation` (or `c2pa.attestation`, `c2pa.attestation_001`, `c2pa.attestation_002`.)

Attestation Assertions can be placed at any index in the `assertions` array but adding them as the final entries in the order that they are created is preferred.

The `attestation-info-map` definition presented here can encode many types of attestation. See Appendix * for currently defined attestation encodings.

[source,cddl]
----
include::attestation-info-map.cddl
----

#### Validating a Claim Containing Explicit Attestations
To perform attestation validation, first all Attestation Assertions are removed from the `assertions` array in the Claim.  The first Attestation Assertion is then checked against the hash of the CBOR-serialized Partial Claim with all Attestation Assertions removed.  If this succeeds, then the first Attestation Assertion is added back into the `assertions` array, the resulting Partial Claim is re-serialized, and the second Attestation Assertion is validated, and so on.  Note that for validation to succeed, the Attestation Assertions must be re-added in the same location in the assertions array.  This is simpler if they are last in the `assertions` array, which is the reason for this recommendation.  However, attestation validators should be able to process Attestation Assertions in any location in the array.

Note that when an Attestation Assertion is removed, the Claim is serialized using normal CBOR rules.  For example, if a claim included two standard assertions and one Attestation Assertion, then the Partial Claim will contain an `assertions` array containing two elements. If a Partial Claim contains two standard assertions and two Attestation Assertions, then two Partial Claims are defined: one omitting just the last Attestation Assertion in the `attestations` array, and one omitting both Attestation Assertions.  Of course, the order of the Attestation Assertions must be preserved as the Partial Claim is created and validated.

## Normative Requirements for Implicit Attestation
