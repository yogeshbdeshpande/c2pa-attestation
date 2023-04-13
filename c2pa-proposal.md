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
