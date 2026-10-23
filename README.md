# cloud-pravaha-modules

The approved module catalog for Cloud Pravaha. This repository holds **no HCL and no custom modules**. It lists public OpenTofu registry modules, pins them to exact versions, and records the security baseline each one must run with.

The catalog is the only thing that decides what Cloud Pravaha can run:
- The control plane rejects any stack that uses something outside it.
- Runners that pin this directory (`PRAVAHA_CATALOG_DIR`) also re-check every downloaded module and provider against it.

## Layout: segregated by provider

```
catalog/
  providers/
    aws.json      hashicorp/aws
    gcp.json      hashicorp/google, hashicorp/google-beta
    azure.json    Azure/azapi, Azure/modtm
    common.json   hashicorp/random (no cloud credentials)
  modules/
    aws/          vpc, s3-bucket, security-group
    gcp/          network, cloud-storage
    azure/        storage-account
    common/       azure-naming, cloudposse-label (usable in any environment)
```

The loader enforces these rules, and `pravaha catalog verify` and CI fail on any violation:

- **The folder decides the cloud.** A module in `modules/aws/` is an AWS module and can only be used in `aws` environments. Files must not set `cloud` themselves.
- **No cross-cloud providers.** A module may use only providers from its own cloud's file plus `common.json`. For example, an AWS module can never pull in `hashicorp/google`, and a `common` module can use no cloud provider at all.
- **Known clouds only.** Folders and provider files must be `aws`, `gcp`, `azure` or `common`. Supporting a new cloud is a backend change (`catalog.Clouds` and provider authentication in `spec/generate.go`), not just a new folder.
- **Each provider is listed once**, in exactly one file.
- **Module IDs are global.** Stacks reference them, so they never change when files move.

Each provider folder has its own reviewers in `.github/CODEOWNERS`, so, for example, the AWS platform team approves AWS catalog changes.

## Module entry

```jsonc
// catalog/modules/aws/s3-bucket.json
{
  "id": "aws-s3-bucket",                       // what stack specs reference
  "title": "AWS S3 bucket",
  "description": "…",
  "source": "terraform-aws-modules/s3-bucket/aws",   // public registry address
  "versions": ["5.17.0"],                      // exact pins, oldest first; the last is the default
  "providers": ["hashicorp/aws"],              // from providers/aws.json or providers/common.json
  "dependencies": [ … ],                       // nested registry modules the module pulls in, pinned
  "inputs": [                                  // the only inputs users may set
    { "name": "bucket", "type": "string", "required": true, "pattern": "^[a-z0-9.-]{3,63}$", "description": "…" }
  ],
  "enforced": { "block_public_acls": true, … }, // always set by the platform; users cannot override
  "outputs": ["s3_bucket_arn"]                 // the only outputs stacks may expose or reference
}
```

Input types: `string`, `number`, `bool`, `list(string)`, `list(number)`, `map(string)`, `map(bool)`, `json` (structured values checked by the module's own type constraints). Constraints: `pattern`, `item_pattern`, `allowed`, `min`, `max`, `max_items`.

## Current catalog

| Folder | Module | Source | Version | Enforced baseline |
| --- | --- | --- | --- | --- |
| aws | aws-vpc | terraform-aws-modules/vpc/aws | 6.7.3 | flow logs on, no public IPs on launch, default SG locked down |
| aws | aws-s3-bucket | terraform-aws-modules/s3-bucket/aws | 5.17.0 | public access blocked, TLS-only, KMS encryption, versioning, owner-enforced |
| aws | aws-security-group | terraform-aws-modules/security-group/aws | 6.0.0 | 0.0.0.0/0 ingress blocked by policy |
| gcp | gcp-network | terraform-google-modules/network/google | 18.3.1 | custom subnets only, default internet routes removed |
| gcp | gcp-cloud-storage | terraform-google-modules/cloud-storage/google | 12.4.1 | public access prevention enforced |
| azure | azure-storage-account | Azure/avm-res-storage-storageaccount/azurerm | 0.10.0 | no public network access or shared keys, TLS 1.2, infrastructure encryption, telemetry off |
| common | azure-naming | Azure/naming/azurerm | 0.4.4 | none: only creates random strings, needs no cloud (used by e2e) |
| common | cloudposse-label | cloudposse/label/null | 0.25.0 | none: creates no resources |

## Changing the catalog

1. Add or update the entry in its cloud's folder (`common/` if it needs no cloud credentials). Pin exact versions only. Add any new provider to that cloud's `providers/<cloud>.json`.
2. Check that every `inputs`, `enforced` and `outputs` name exists in that module version.
3. Run the verifier from the backend repository:
   ```bash
   cd ../cloud-pravaha-backend && go run ./cmd/pravaha catalog verify -dir ../cloud-pravaha-modules/catalog -online
   ```
   It validates the structure and segregation rules, and confirms each pinned module, dependency and provider version exists on registry.opentofu.org.
4. Open a pull request. Catalog changes widen what can run in production, so the folder's code owners must approve.

When upgrading a module, add the new version next to the old one first, so existing stacks keep validating. Remove the old version once stacks have moved.
