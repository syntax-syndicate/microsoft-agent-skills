---
name: azure-identity-rust
description: |
  Azure Identity library for Rust. Microsoft Entra ID authentication for all Azure SDK clients.
  Triggers: "azure identity rust", "DeveloperToolsCredential", "authentication rust", "managed identity rust", "credential rust", "Entra ID rust".
license: MIT
metadata:
  author: Microsoft
  version: "1.0.0"
  package: azure_identity
---

# Azure Identity library for Rust

Microsoft Entra ID authentication for Azure SDK clients.

Use this skill when:

- An app needs to authenticate to Azure services from Rust
- You need `DeveloperToolsCredential` for local development
- You need `ManagedIdentityCredential` for Azure-hosted workloads
- You need service principal auth with secret or certificate
- You need federated identity credential (FIC) auth

> **IMPORTANT:** Only use official `azure_*` crates published by the [azure-sdk](https://crates.io/users/azure-sdk) crates.io user. Do NOT use the deprecated `azure_sdk_*` crates (MindFlavor/AzureSDKForRust) or community crates. Official crates use underscores in names and none have version 0.21.0.

> **Note:** The Rust SDK does not have `DefaultAzureCredential`. Use `DeveloperToolsCredential` for local development and `ManagedIdentityCredential` for production.

## Installation

```sh
cargo add azure_identity tokio
```

> **Do not** add `azure_core` directly to `Cargo.toml`. It is re-exported by service crates.

## Environment Variables

```bash
AZURE_TENANT_ID=<your-tenant-id>         # Required for service principal auth
AZURE_CLIENT_ID=<your-client-id>         # Required for service principal or user-assigned managed identity
AZURE_CLIENT_SECRET=<your-client-secret> # Required for ClientSecretCredential
```

## Authentication

### DeveloperToolsCredential (Local Development)

Tries Azure CLI then Azure Developer CLI:

```rust
use azure_identity::DeveloperToolsCredential;
use azure_security_keyvault_secrets::SecretClient;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Local dev: DeveloperToolsCredential. Production: use ManagedIdentityCredential.
    let credential = DeveloperToolsCredential::new(None)?;
    let client = SecretClient::new(
        "https://<vault-name>.vault.azure.net/",
        credential.clone(),
        None,
    )?;

    let secret = client.get_secret("secret-name", None).await?.into_model()?;
    println!("Secret: {:?}", secret.value);
    Ok(())
}
```

Ensure you are logged in:

```sh
az login        # Azure CLI
azd auth login  # or Azure Developer CLI
```

| Order | Credential                  | Login Command    |
| ----- | --------------------------- | ---------------- |
| 1     | AzureCliCredential          | `az login`       |
| 2     | AzureDeveloperCliCredential | `azd auth login` |

### ManagedIdentityCredential (Production)

For Azure-hosted resources (VMs, App Service, Functions, AKS):

```rust
use azure_identity::ManagedIdentityCredential;

// System-assigned managed identity
let credential = ManagedIdentityCredential::new(None)?;

// User-assigned managed identity
let options = ManagedIdentityCredentialOptions {
    client_id: Some("<managed-identity-client-id>".into()),
    ..Default::default()
};
let credential = ManagedIdentityCredential::new(Some(options))?;
```

### ClientSecretCredential (Service Principal)

For CI/CD pipelines and service accounts:

```rust
use azure_identity::ClientSecretCredential;

let credential = ClientSecretCredential::new(
    "<tenant-id>",
    "<client-id>",
    "<client-secret>",
    None,
)?;
```

### Federated Identity Credential (FIC)

Authenticate an Entra application with an access token from a managed identity. See [Configure an application to trust a managed identity](https://learn.microsoft.com/entra/workload-id/workload-identity-federation-config-app-trust-managed-identity) for details.

```rust
use azure_core::credentials::TokenCredential;
use azure_core::http::ClientMethodOptions;
use azure_identity::{ClientAssertion, ClientAssertionCredential, ManagedIdentityCredential};
use azure_security_keyvault_secrets::SecretClient;
use std::sync::Arc;

#[derive(Debug)]
struct AccessTokenAssertion {
    credential: Arc<dyn TokenCredential>,
}

#[async_trait::async_trait]
impl ClientAssertion for AccessTokenAssertion {
    async fn secret(&self, _: Option<ClientMethodOptions<'_>>) -> azure_core::Result<String> {
        Ok(self
            .credential
            .get_token(&[&"api://AzureADTokenExchange/.default"], None)
            .await?
            .token
            .secret()
            .to_string())
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let assertion = AccessTokenAssertion {
        credential: ManagedIdentityCredential::new(None)?,
    };

    let credential = ClientAssertionCredential::new(
        String::from("tenant-id"),
        String::from("client-id"),
        assertion,
        None,
    )?;

    let client = SecretClient::new("https://<vault>.vault.azure.net/", credential.clone(), None)?;
    Ok(())
}
```

## Credential Types

| Credential                    | Use Case                               |
| ----------------------------- | -------------------------------------- |
| `DeveloperToolsCredential`    | Local development — tries CLI tools    |
| `ManagedIdentityCredential`   | Azure VMs, App Service, Functions, AKS |
| `WorkloadIdentityCredential`  | Kubernetes workload identity           |
| `ClientSecretCredential`      | Service principal with secret          |
| `ClientCertificateCredential` | Service principal with certificate     |
| `AzureCliCredential`          | Direct Azure CLI auth                  |
| `AzureDeveloperCliCredential` | Direct azd CLI auth                    |
| `AzurePipelinesCredential`    | Azure Pipelines service connection     |
| `ClientAssertionCredential`   | Custom assertions (federated identity) |

## Best Practices

1. **Use `DeveloperToolsCredential`** for local dev, **`ManagedIdentityCredential`** for production — the Rust SDK does not have `DefaultAzureCredential`
2. **Never hardcode credentials** — use environment variables for service principals
3. **Clone credentials** — pass `credential.clone()` when constructing multiple clients; credentials are `Arc`-wrapped
4. **Reuse clients** — clients are thread-safe (`Send + Sync`); create once, share across tasks
5. **Assign RBAC roles** — ensure the identity has appropriate roles for the target service (e.g., "Key Vault Secrets User" for secret reads)

## Reference Links

| Resource      | Link                                                                                                      |
| ------------- | --------------------------------------------------------------------------------------------------------- |
| API Reference | https://docs.rs/azure_identity                                                                            |
| crates.io     | https://crates.io/crates/azure_identity                                                                   |
| Source        | https://github.com/Azure/azure-sdk-for-rust/tree/main/sdk/identity/azure_identity                         |
| Credentials   | https://github.com/Azure/azure-sdk-for-rust/tree/main/sdk/identity/azure_identity#credential-structures   |
