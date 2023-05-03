## Design Considerations
In this section we discuss the options for incorporating attestations into C2PA as well as the considerations that led to proposed design.  The sections that follow 
contain the normative requirements for incorporating explicit and impicit attestations into a C2PA manifest.

### Explicit Attestation

The current C2PA trust model is built using two signatures:

1)	A signature over a CBOR-serialized Claim.  The signature is encoded in the manifest, and the certificate chain for the signing key is usually also included.
2)	Optionally, a countersignature from an RFC 3161-compliant time stamping service.

Attestations – in the context of this document – are also digital signatures.  This section describes options for how the additional signature can be added to the 
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
    id1(Claim Signature) --> id2(Attestation) --> id3(Claim);
    id2 --> id1;
    id1 --> id3;
```

