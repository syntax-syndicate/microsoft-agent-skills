---
name: azure-keyvault-certificates-rust
description: |
  Azure Key Vault Certificates library for Rust. Create, manage, and use X.509 certificates including self-signed and CA-issued.
  Triggers: "keyvault certificates rust", "CertificateClient rust", "create certificate rust", "self-signed certificate rust", "X.509 rust".
license: MIT
metadata:
  author: Microsoft
  version: "1.0.0"
  package: azure_security_keyvault_certificates
---

# Azure Key Vault Certificates library for Rust

Manage X.509 certificates for TLS/SSL, code signing, and authentication.

Use this skill when:

- An app needs to create or manage X.509 certificates in Key Vault from Rust
- You need self-signed or CA-issued certificates
- You need long-running operations (LRO) for certificate issuance
- You need to sign data using a certificate's key

> **IMPORTANT:** Only use the official `azure_security_keyvault_certificates` crate published by the [azure-sdk](https://crates.io/users/azure-sdk) crates.io user. Do NOT use unofficial or community crates. Official crates use underscores in names and none have version 0.21.0.

## Installation

```sh
cargo add azure_security_keyvault_certificates azure_identity tokio futures
```

> **Do not** add `azure_core` directly to `Cargo.toml`. It is re-exported by `azure_security_keyvault_certificates`.

## Environment Variables

```bash
AZURE_KEYVAULT_URL=https://<vault-name>.vault.azure.net/ # Required for all operations
```

## Authentication

```rust
use azure_identity::DeveloperToolsCredential;
use azure_security_keyvault_certificates::CertificateClient;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Local dev: DeveloperToolsCredential. Production: use ManagedIdentityCredential.
    let credential = DeveloperToolsCredential::new(None)?;
    let client = CertificateClient::new(
        "https://<vault-name>.vault.azure.net/",
        credential.clone(),
        None,
    )?;

    let cert = client
        .get_certificate("cert-name", None)
        .await?
        .into_model()?;
    println!("Certificate: {:?}", cert.id);
    Ok(())
}
```

## Core Workflow

### Create Self-Signed Certificate (LRO)

Creating a certificate is a long-running operation. `begin_create_certificate` returns a `Poller<CertificateOperation>` that implements `IntoFuture` — just `.await`:

```rust
use azure_security_keyvault_certificates::{
    models::{
        CertificatePolicy, CreateCertificateParameters, IssuerParameters,
        X509CertificateProperties,
    },
    ResourceExt,
};

let policy = CertificatePolicy {
    x509_certificate_properties: Some(X509CertificateProperties {
        subject: Some("CN=DefaultPolicy".into()),
        ..Default::default()
    }),
    issuer_parameters: Some(IssuerParameters {
        name: Some("Self".into()),
        ..Default::default()
    }),
    ..Default::default()
};
let body = CreateCertificateParameters {
    certificate_policy: Some(policy),
    ..Default::default()
};

// Poller implements IntoFuture — await directly for completion
let cert = client
    .begin_create_certificate("cert-name", body.try_into()?, None)?
    .await?
    .into_model()?;
```

### Update Certificate Properties

```rust
use azure_security_keyvault_certificates::models::UpdateCertificatePropertiesParameters;
use std::collections::HashMap;

#[allow(clippy::needless_update)]
let params = UpdateCertificatePropertiesParameters {
    tags: Some(HashMap::from_iter(vec![("env".into(), "prod".into())])),
    ..Default::default()
};

client
    .update_certificate_properties("cert-name", params.try_into()?, None)
    .await?
    .into_model()?;
```

### Delete Certificate

```rust
client.delete_certificate("cert-name", None).await?;
```

### List Certificates (Pagination)

`list_certificate_properties` returns a `Pager<T>` — `Pager` implements `Stream`, so iterate directly:

```rust
use azure_security_keyvault_certificates::ResourceExt;
use futures::TryStreamExt;

let mut pager = client.list_certificate_properties(None)?;
while let Some(cert) = pager.try_next().await? {
    println!("Found: {}", cert.resource_id()?.name);
}
```

## Signing with a Certificate's Key

Certificates in Key Vault have an associated key. Use the Key Vault Keys SDK for crypto operations:

```rust
use azure_core::base64;
use azure_security_keyvault_certificates::{
    models::{
        CertificatePolicy, CreateCertificateParameters, CurveName, IssuerParameters,
        KeyProperties, KeyType, KeyUsageType, X509CertificateProperties,
    },
    ResourceExt,
};
use azure_security_keyvault_keys::{
    models::{KeyClientSignOptions, SignParameters, SignatureAlgorithm},
};
use openssl::sha::sha256;

let plaintext = "plaintext";

// Create an EC certificate policy for signing
let policy = CertificatePolicy {
    x509_certificate_properties: Some(X509CertificateProperties {
        subject: Some("CN=DefaultPolicy".into()),
        key_usage: Some(vec![KeyUsageType::DigitalSignature]),
        ..Default::default()
    }),
    issuer_parameters: Some(IssuerParameters {
        name: Some("Self".into()),
        ..Default::default()
    }),
    key_properties: Some(KeyProperties {
        key_type: Some(KeyType::Ec),
        curve: Some(CurveName::P256),
        ..Default::default()
    }),
    ..Default::default()
};

let body = CreateCertificateParameters {
    certificate_policy: Some(policy),
    ..Default::default()
};

// Wait for the certificate operation to complete
let certificate = client
    .begin_create_certificate("ec-signing-certificate", body.try_into()?, None)?
    .await?
    .into_model()?;
let certificate_version = certificate
    .resource_id()?
    .version
    .expect("certificate version required");

// Hash the plaintext to be signed
let digest = sha256(plaintext.as_bytes()).to_vec();

// Sign the digest using the certificate's key via a KeyClient
let body = SignParameters {
    algorithm: Some(SignatureAlgorithm::Es256),
    value: Some(digest),
};

let signature = key_client
    .sign(
        "ec-signing-certificate",
        body.try_into()?,
        Some(KeyClientSignOptions {
            key_version: Some(certificate_version.clone()),
            ..Default::default()
        }),
    )
    .await?
    .into_model()?;

if let Some(signature) = signature.result.map(base64::encode_url_safe) {
    println!("Signature: {}", signature);
}
```

## Certificate Formats

| Format  | Content Type             | Use Case                            |
| ------- | ------------------------ | ----------------------------------- |
| PKCS#12 | `application/x-pkcs12`   | Bundled cert + private key          |
| PEM     | `application/x-pem-file` | Base64-encoded, common in Linux/web |

## RBAC Roles

For Entra ID auth, assign one of these roles:

| Role                             | Access                      |
| -------------------------------- | --------------------------- |
| `Key Vault Certificate User`     | Use certificates            |
| `Key Vault Certificates Officer` | Full certificate management |

## Error Handling

```rust
match client.get_certificate("certificate-name", None).await {
    Ok(response) => println!("Certificate: {:#?}", response.into_model()?.x509_thumbprint),
    Err(err) => println!("Error: {:#?}", err.into_inner()?),
}
```

## Best Practices

1. **Use `DeveloperToolsCredential`** for local dev, **`ManagedIdentityCredential`** for production — the Rust SDK does not have `DefaultAzureCredential`
2. **Never hardcode credentials** — use environment variables or managed identity
3. **Use `..Default::default()`** with `#[allow(clippy::needless_update)]` for model struct updates
4. **Use `ResourceExt`** to extract certificate name/version from IDs
5. **LROs** — `begin_create_certificate` returns a `Poller`; just `.await` for completion
6. **Reuse clients** — `CertificateClient` is thread-safe; create once, share across tasks

## Reference Links

| Resource      | Link                                                                                                                         |
| ------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| API Reference | https://docs.rs/azure_security_keyvault_certificates                                                                         |
| crates.io     | https://crates.io/crates/azure_security_keyvault_certificates                                                                |
| Source        | https://github.com/Azure/azure-sdk-for-rust/tree/main/sdk/keyvault/azure_security_keyvault_certificates                      |
| Examples      | https://github.com/Azure/azure-sdk-for-rust/tree/main/sdk/keyvault/azure_security_keyvault_certificates/examples             |
