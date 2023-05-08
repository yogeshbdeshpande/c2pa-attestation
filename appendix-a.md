# Appendix A - Attestation Encoding Rules for Explicit Attestation

This appendix describes the currently defined attestation schemes, and how the attestation information is incorporated into a C2PA manifest.

In all cases, attestations should be calculated 'over' the hash of the serialization of the Partial Claim. 

This section is preliminary and just contains the gist.  We need to refine and validate with implementations.

## Android Play Integrity
Android Play Integrity is a platform service for determining the identity of an Android application, and whether that application is running on an in-policy Android device. 

#### Obtaining an Attestation 

Attestations are typically obtained using: 

integrityManager.requestIntegrityToken(IntegrityTokenRequest.builder().setNonce(nonce).build());
IntegrityTokenRequest.Builder.setNonce(NonceString).

Where the `NonceString` is the base64-encoded hash of the `attestation-tbs-map` for the Claim.

#### Definition of Fields in `attestation-info-map` for Android Play Integrity

| Field | Value |
|-------|---------|
| `att-type` | `c2pa.AndroidPlayIntegrity` |
| `att-result` | UTF-8-encoding of the null-terminated attestation result |
| `other-info` | (Optional) | 

## Intel SGX

### Obtaining an Attestation 
TBD.   

#### Definition of Fields in `attestation-info-map` for Intel SGX.

| Field | Value |
|-------|---------|
| `att-type` | `c2pa.SGX` |
| `att-result` |  |
| `certificate` |  |
| `other-info` |  | 


## TPM 2.0
### Obtaining an Attestation 
Applications should create C2PA attestations using the `TPM2_Quote` operation with the `qualifyingData` set to the digest of the Partial Claim using the same hash algorithm as Platform Configuration Registers (PCRs) that are used. 

The key used for signing should be a *restricted signing key*, and applications should specify a set PCRs that adequately represent the security configuration of the host platform. 

Depending on scenario, it may be necessary or desireable to also include/embed a certificate chain for the quoting key.    

#### Definition of Fields in `attestation-info-map` for TPM 2.0

| Field | Value |
|-------|---------|
| `att-type` | `c2pa.TPM2.0` |
| `att-result` | Raw binary `TPMT_SIGNATURE` obtained from the TPM |
| `certificate` | base-64 encoded PEM-encoded certificate chain for the quoting key. |
| `other-info` | Raw binary `TPM2B_ATTEST` data obtained from the TPM. | 

## IETF RATS

## Obtaining an Attestation 

TBD.

#### Definition of Fields in `attestation-info-map` for IETF RATS

| Field | Value |
|-------|---------|
| `att-type` | `c2pa.RATS` |
| `att-result` | TBD |
| `certificate` | TBD |
| `other-info` | TBD | 
