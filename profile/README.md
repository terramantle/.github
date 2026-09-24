<p align="center">
  <a href="https://terramantle.dev">
    <img src="assets/mark.svg" alt="Terramantle" width="96" height="96">
  </a>
</p>

<h1 align="center">Terramantle</h1>

<p align="center">
  Private module, provider and state registry for Terraform and OpenTofu.<br>
  Pin modules to versions, not Git refs, and see every workspace that consumes one before you change it.
</p>

<p align="center">
  <a href="https://terramantle.dev">Website</a> ·
  <a href="https://terramantle.dev/docs">Docs</a> ·
  <a href="https://portal.terramantle.dev">Portal</a> ·
  <a href="https://terramantle.dev/blog">Blog</a> ·
  <a href="https://terramantle.dev/faq">FAQ</a> ·
  <a href="https://terramantle.dev/pricing">Pricing</a>
</p>

## Git is not a registry

Most Terraform stacks pull modules straight from Git. A `~> 3.2` constraint means nothing against a Git source. A force-push or a moved tag silently changes what every consumer receives on its next `init`. Nobody can tell you which workspaces depend on the module you are about to break, so breaking changes land as surprises and deprecations travel by chat message. Meanwhile the long-lived token that lets CI publish sits on a runner, one leak away from your whole module distribution path.

Terramantle is the registry and state layer that fixes this, and nothing more. It does not run your pipelines and it never holds your cloud credentials. Plan and apply keep running in GitHub Actions or GitLab CI exactly where they run today. Your pipelines pull modules, providers and state from Terramantle over OIDC.

## What it does

Terramantle implements the Terraform Registry Protocol v1 and the standard HTTP state backend. Terraform and OpenTofu work unchanged, with no new CLI or plugin to install.

- **Base artefact registry.** Immutable, semantically versioned modules, states, plans and providers. What you published is what gets consumed. Version overwrites are refused.
- **Dependency graph.** Every module, version and workspace that consumes a module, visible before you change it. 
- **Deprecation notices** reach consumers through the toolchain
- **Keyless CI publishing.** Pipelines prove their identity at publish time with GitHub or GitLab OIDC. Nothing to rotate, nothing to leak.
- **Patched Provider registry.** Think Chainguard but for Providers. Serve built and upstream providers you choose, with per-platform checksums and GPG-signed SHA256SUMS, behind your own access control.
- **Supply chain scanning.** Secrets, misconfiguration, malware and CVE checks run on every published version. Provider binaries get an SPDX SBOM. Findings feed OPA policies, and a version that fails an enforcing policy returns 403 on download instead of being served quietly.
- **Signing bound to gates.** A Terramantle signature says more than who signed. Before the HSM signs a provider, four gates must pass, independently of the build: Semgrep SAST on the source tree, AI intent review for backdoors and exfiltration, ClamAV on the built binary, and a dependency audit with govulncheck.
- **State insight.** Versioned, locked state with audit logs, secret detection and public endpoint discovery on every push. Delegated force unlock, promotion and rollback from the portal, without handing out state credentials.
- **Access control.** Scoped tokens, RBAC and audit logs. Every publish and every consume is recorded: who published which version, and which workspace pulled it.

## Read more

- [Problems it solves](https://terramantle.dev/problems)
- [Why there are no runners](https://terramantle.dev/architecture/no-runners)
- [Supply chain scanning](https://terramantle.dev/supply-chain)
- [Provider registry](https://terramantle.dev/provider-registry)
- [Air-gapped deployment](https://terramantle.dev/air-gapped)
- [OpenTofu](https://terramantle.dev/opentofu)
- [Blog](https://terramantle.dev/blog)

## Cloud or self-hosted

The cloud tier is free for individuals and small teams. Module archives, provider binaries and state live in S3-compatible storage in the EU (Ireland).

Self-hosting is on the roadmap. Run it on Kubernetes (Rancher, OpenShift, Tanzu), Docker Compose or any Linux host, backed by your own S3-compatible object storage (MinIO, SeaweedFS, Ceph, AWS S3) and database. Authenticate standalone or against Keycloak, Microsoft Entra, Okta or another OIDC provider. There is no outbound internet dependency, so it runs inside air-gapped and regulated networks, and no data leaves your infrastructure.

## Quick start

1. Sign up at [portal.terramantle.dev](https://portal.terramantle.dev).
2. Enable Pipeline Trust for your GitHub or GitLab repository under Settings, so CI can publish over OIDC instead of a stored token.
3. Point your configuration at the registry.

Publish a module from GitHub Actions with no stored secret:

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - name: Publish via OIDC
    run: |
      TOKEN=$(curl -sH "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
        "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=terramantle" | jq -r '.value')
      tar -czf module.tar.gz -C . .
      curl -X PUT \
        "https://registry.terramantle.dev/v1/modules/<YOUR_ORG>/vpc/aws/1.2.3" \
        -u "terramantle-bot:$TOKEN" \
        --data-binary @module.tar.gz
```

Consume it, and pin a provider through your registry:

```hcl
module "vpc" {
  source  = "registry.terramantle.dev/<YOUR_ORG>/vpc/aws"
  version = "~> 1.2"
}

terraform {
  required_providers {
    aws = {
      source  = "<YOUR_ORG>.registry.terramantle.dev/hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

Store state with the standard HTTP backend:

```hcl
terraform {
  backend "http" {
    address        = "https://registry.terramantle.dev/state/<YOUR_ORG>/<YOUR_WORKSPACE>"
    username       = "terramantle-bot"
    lock_address   = "https://registry.terramantle.dev/state/<YOUR_ORG>/<YOUR_WORKSPACE>.lock"
    unlock_address = "https://registry.terramantle.dev/state/<YOUR_ORG>/<YOUR_WORKSPACE>.lock"
  }
}
```

The full integration guide, covering Pipeline Trust, state locking, provider signing and troubleshooting, is at [terramantle.dev/docs](https://terramantle.dev/docs).

## In this organisation

| Repository | What it is |
| --- | --- |
| [terramantle-cli](https://github.com/terramantle/terramantle-cli) | Experimental Rust CLI for registry discovery, lock-file upload from CI and state operations. Install with `brew install terramantle/tap/terramantle`. MIT. |
| [homebrew-tap](https://github.com/terramantle/homebrew-tap) | Homebrew tap for the CLI. |
| [modules-sample-mono-repo](https://github.com/terramantle/modules-sample-mono-repo) | Sample modules published into the Terramantle demo organisation. Shows the OIDC publishing flow end to end. |
| [awesome-tf](https://github.com/terramantle/awesome-tf), [awesome-opentofu](https://github.com/terramantle/awesome-opentofu) | Curated Terraform and OpenTofu resource lists. |
| `forks-*` | Auto-patch mirrors. CVE-patched provider source and builds, published so the patches can be audited. Do not push to them manually. |


## Contact

[info@terramantle.dev](mailto:info@terramantle.dev). Terramantle is built in the United Kingdom.
