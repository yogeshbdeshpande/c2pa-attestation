# Introduction:

Attestation is a key security feature that enables relying parties, to deduce whether a given entity
is trustworthy (example in a correct state and is operating as intended). The given entity can be an application, a platform or a composite device that is been attested.

 
This specification is a proposal to introduce attestation to C2PA Eco System, i.e. C2PA trust model and its data structures.

C2PA seeks feedback on this proposal, especially from implementers. The underlying draft will be
refined based on inputs. The goal is to either refine a non-draft specification or a revision to the main specification.


This draft assumes a working understanding of the C2PA Architecture and Data Structures: in particular,
Claims, Assertions, Claim Signatures and how these data objects are packaged into C2PA manifests.

 
## Relevance to C2PA

There are scenarios where introducing attestation within C2PA Eco-System further enhance
the overall security posture of the system. The below examples are few use cases.
 

### Trusted Camera Application on an Android Phone

Sophisticated and easy to use media editing tools as well as synthetic image generation technologies
are widely available. The existence of these tools degrades the probative value of all digital imagery.

 
A Trusted Camera application can use attestation to

    (a) indicate the application that captured the image,
    (b) implicitly prove that an image was obtained from the sensor,
    (c) indicate that the phone operating system is up-to-date and configured securely.

While no system is foolproof and immune from security threats, for some scenarios an attestation
can markedly increase the likelihood that an image is of a real scene.


Android Play Integrity is a Google service for obtaining such attestations.
We expect (but do not require) that the key used to create the Claim Signature
is associated with the owner/user of the camera phone.

 
### AI/ML Models running in a container or Isolated Virtual Machine

Machine learning models are increasingly responsible for security-critical tasks, and training sets,
models, and inferences will all benefit from C2PA-style provenance signals and protection.
In some cases, ML models execute in environments with poor or unknown security characteristics.
In these cases, attestations can be added to demonstrate that a model or inference was trained/run
in a secure environment.

For concreteness, we assume an ML image recognition model with a valid C2PA manifest running in an SGX enclave, which then outputs metadata - information about any recognized objects - with an asset manifest that references the input image and the model that produced the information about the objects. The resulting manifest is signed by the claim creator (as normal), but also includes an attestation that indicates the code running in the enclave that performed the image recognition steps. Relying parties can use the manifest and attestation assertion to ensure that the image was processed by an authorized tool or model.


## Specifics of Attestation

Attestation is (typically) a specialized type of signing operation where the set of claims about the platform and application (that requested attestation), is signed by the trusted authority to generate attestation token, also known as Evidence in attestation eco-system.

For example: 

* TPM quote-operations are a signature over some application-provided data together with an encoding of current Platform Configuration Registers (PCR) values.  Compliant platforms record the booting software into the PCRs, so the TPM quote reports reflects the software running on the platform.

* Android Play Integrity is a system/cloud-service that allows app-developers to generate signed statements indicating the package that requested the integrity verdict, as well information about the security posture of the requesting device.

* SGX provides a reporting/quoting feature that is a signature over the measurement of the code that was loaded into the enclave, as well as other critical security settings.

* PSA provides the calling application to collect an attestation token ( which is a signature over a set of claims about underlying platform with a supplied freshness parameter from application).

Attestations can be directly consumed by relying parties or can be forwarded to a Trust Broker (independent Verification Service).

Trust Brokers are described below. (Android Play Integrity verdicts use a Google-provided Trust Broker service.)


## Implicit Attestations

TPM quotes and Play Integrity verdicts are explicit attestations:

However a simpler approach for attestation is to use an "Implicit Attestation".

In this model, during the "C2PA claim signing key creation/provisioning phase",
the claim signing key would undergo an explicit attestation with a trust broker (verification service)
to establish the following facts:

* The key always resides in a secure location within the platform

* Only certain authorized applications have access to the signing key, using a "sealing primitive"

Here the signing key is only available to the authorized C2PA Claim Creator application, signing the C2PA Claim and generating the C2PA Claim signature.

A C2PA Validator Verifying the C2PA Claim Signature using Public Key pertaining to Claim Signer, has 
"implicit" indication that the Claim Creating/Signing application is a trustworthy application.

The only requirement for the Implicit Attestation to work within C2PA Eco-system is:
 
    1) Key resides in a secure storage within the C2PA device.
    2) C2PA device supports "Explicit Key Attestation" during initial key creation and provisioning phase.
    3) When the Signing Key is provisioned in the C2PA device, it undergoes explicit key attestation with the secure key provider service.
    4) If the signing key is revoked for any reason (e.g. due to expiry), then a new key to be provisioned, has to undergo the same "Explicit Attestation Protocol".

Specific Explicit "Key Attestation" Protocol is out of scope of C2PA Specification.

## Explicit Attestations

Explicit Attestation allow a C2PA Claim Signer to undergo attestation with a local or remote
Trust broker and introduce the attestation as part of C2PA Manifest.
Compared to Implicit Attestation an Explicit Attestation offers additional guarantees, such as:

1. An attestation aware C2PA Validator can get a more fine and granular information conveyed via attestation, example a specific version of firmware running on the device.
2. A more rich set of information can be conveyed via explicit attestation. For example, a Device generating a C2PA Manifest has undergone a certain level of certification to comply with security standards.
3. Explicit Attestation is extensible in a way that Attestation in future could be from a composite device, consisting of individual components (like camera sensor and Camera CPU hardware-firmware) combined together. This further augments the trust metrics.

### Design Considerations

The current C2PA trust model is built using two signatures:
1. A signature over a CBOR-serailized Claim. The signature is encoded in the manifest, and the 
certificate chain for the signing key is usually also included.
2. Optionally, a countersignature from an RFC 3161-compliant time stamping service.

Attestations, in the context of this document are also digital signatures. This section describes options for how the additional signatures can be added to the two that are already defined.
This section provides a high level view of the design approach used, while a more detailed design for the chosen approach is given in 
[detail design](#detail-design)

Throughout this section, when it is referred as "Attestation over - X", it implies, attestation is a signature over the hash of the serialization of X.

The simplest option for adding attestation specific data would be to include it as a second counter signature over a serialized claim. However with this design, the claim signature or attestation can be independently removed and replaced. Whether this type of attack can take place depends on the scenario and sophistication of the relying party, however it cannot be simply ignored.

Subsequent section aims to provide simple steps which can mitigate the above mentioned threat.

To achieve claim signature and attestation binding, an attestation over the serialized claim 
is performed first and then the claim signature is perfomed over the attestation. This prevents
an attestation being removed indepedently of the claim signature. Please note that the details of this design is described in 
[detail design](#detail-design)

To prevent the claim signature being replaced, the attestation also binds the public key of the claim signer. This means that the validators can reject claim signature if the claim signature key specified in the attestation does not match the actual claim signature.

### Detail Design

This section describes in more detail, how an attestation is created and encoded in a manifest.

#### Linking Attestation Data to C2PA

Current C2PA design establishes cryptographic linkage between asset, assertions, claim and its claim signature. Claim Signatures are calculated over the serialization of a claim, and the claim itself is cryptographically linked to assertions. Two types of assertion are important, assertions that contain the metadata and assertions that bind a claim to an asset with a hash or hashes of an asset or asset fragment. By signing the Claim, the claim creator vouches for both asset and the metadata.

The design proposed in this section introduces similar bindings/cryptographic linkages for the attestation signatures [Glossary](#glossary). Majority of scenarios benefit binding to the asset (e.g
for a Trusted Camera Application *"this image was captured by this program running on this device*"
Some scenarios benefit from binding to an Assertion, e.g. *"the GPS coordinates obtained from the radio when this image was captured"* or for the ML scenario, *"the following objects were recognized in the image"*)

The approach taken in current design is to bind the attestation to the serailization of the entire claim, i.e. the same data structure that the claim generator signs (with one exception which is described below). Not all assertions/metadata gain security benefit from attestation (as opposed to simple claim signature), but signing all assertions provide the most flexibility for future scenarios. TODO: Check this sentence for relevance: For such advanced use cases, we introduce a field that an attestation generator can use to indicate, which assertions are "attested".

#### Embedding Attestation in a Manifest

Attestations are like claim signatures (signed by Attestation signers) and could be encoded in a manifest similarly. However, there is also an additional security requirement that claim signers should also sign them to avoid the "strip and replace" problem as mentioned above.

A simple approach could have been to encode Attestation Data ( either Attestation Result or Attestation Token), as a C2PA assertion and then generate the claim and sign the claim (as done in
normal C2PA flow). However, this approach will not work, as there is a circular dependency:
one cannot generate attestation without complete claim refer [linking c2pa](#linking-attestation-data-to-c2pa) and claim cannot
be finalized before the attestation assertion is created.

The design breaks the circular dependency by specifying that:
1.  Attestation Data is created using the hash of serialized claim *but ommiting the attestation assertion (which has not been created yet)*. This is known as *partial claim*
2. Once the Attestation has been created, it is embedded in the manifest as "attestation assertion"
and its associated claim, identical to any other assertion and subsequently be signed by "claim signature" as per the normal C2PA sequence. 
3. This approach is fully backwards compatible, as explained in section [compatibility](#compatibility-with-current-specification)

A simplified logical flow for creating "attestation assertion" is given below

@Akshay TODO Insert .png for Manifest Creation 

#### Attestation Data Structures

### Validating flow for a Attestation Aware C2PA Validator



### Security Considerations

The security properties of Attestation within C2PA is similar to normal C2PA trust model.
Just as non attestation claim signature can be stripped and replaced, so can the combined attestation/claim signature pair. However, neither can be replaced independently.


## Compatibility with current specification
The proposed design is fully backwards compatible, as explained below:

* Normal claim creation and claim validation is unaffected.
* An Attestation enhanced claim can be processed by an *attestation-unaware* C2PA Validator,
without changes. The attestation assertion can be treated as any other third party assertion
or unrecognized assertion.


# GLOSSARY

Following terms are extensively used in this document.

* Claim Signature: The definition is the same as documented in C2PA core specification.

* Attestation Signature: A signature generated over an Attestation Statement, produced either by:
(a) Remote Attestation Verifier when Attestation Statement is an Attestation result generated from a remote verifier (b) An application or underlying platform of a C2PA capturing device, when the Attestation data is generated locally containing attestation information (known as Attestation token in RATS world)

* Attestation over - X.

* 