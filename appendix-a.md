# Appendix A - Attestation Encoding Rules for Explicit Attestation

This appendix describes the currently defined attestation schemes, and how the attestation information is incorporated into a C2PA manifest.

This section is preliminary and just contains the gist.  We need to refine and validate with implementations.

## Android Play Integrity
Android Play Integrity is a platform service for determining the identity of an Android application, and whether that application is running on an in-policy Android device. 

#### Obtaining an Attestation 

Attestations are typically obtained using: 

integrityManager.requestIntegrityToken(IntegrityTokenRequest.builder().setNonce(nonce).build());
IntegrityTokenRequest.Builder.setNonce(NonceString).

Where the `NonceString` is the base64-encoded hash of the `attestation-tbs-map` for the Claim.

#### Definition of Fields in `attestation-info-map`

| Field | Value |
|-------|---------|
| `att-type` | `c2pa.AndroidPlayIntegrity` |
| `att-result` | UTF-8-encoding of the null-terminated attestation result |
| `other-info` | (Optional) Certificate chain for IACS key (encoding?) | 

## Intel SGX

## TPM

## IETF RATS


