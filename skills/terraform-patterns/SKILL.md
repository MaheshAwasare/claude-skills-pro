---
name: terraform-patterns
description: Terraform module layout, remote state, environments, drift detection, and the small-team patterns that don't appear in the official docs. Use when starting a new Terraform repo, adding a second environment, or untangling a Terraform repo that has grown into a hairball.
---

# Terraform Patterns

Terraform's docs cover *syntax*; they don't cover *layout*. This skill is the layout. Optimized for small-to-medium teams (1–20 engineers) running on AWS, GCP, or Azure.

## When to use

- New Terraform repo from scratch.
- Adding a second/third environment (staging → prod, or multi-region).
- Refactoring a single-`main.tf`-with-1000-lines repo.
- Setting up CI/CD for Terraform.

## When NOT to use

- Massive enterprise (500+ engineers) — go read Gruntwork's IAC patterns.
- You're using Pulumi or CDKTF — different idioms.
- One-off scripts — Terraform's overhead isn't worth it for "spin up an EC2 once."

## Decisions made for you

| Decision | Choice | Why |
|---|---|---|
| State backend | S3 + DynamoDB lock (AWS) / GCS (GCP) / azurerm (Azure) | Free, audited, well-supported |
| State per env | Separate state file per env | Blast-radius isolation |
| Module sources | `./modules/...` local refs, NOT a separate repo | Single PR, single review |
| Workspaces | Avoid for envs | Use directories instead — workspaces hide the env in CLI state |
| Versions | Pin Terraform + provider in `versions.tf` | Reproducible across machines |
| Secrets | NEVER in `.tfvars`, NEVER in state if avoidable | Use AWS SM / GCP SM / 1Password |
| `tfsec` / `tflint` / `checkov` | All three in CI | They catch different things |

## Repo layout

```
infra/
  envs/
    dev/
      main.tf                 # composes modules, dev-specific values
      backend.tf              # S3 backend config (dev bucket/key)
      providers.tf
      variables.tf
      terraform.tfvars        # dev values (NO secrets)
    staging/
      ...
    prod/
      ...
  modules/
    network/                  # one module per logical resource group
      main.tf
      variables.tf
      outputs.tf
      versions.tf
      README.md
    database/
    compute/
    monitoring/
  global/                     # things shared across envs (IAM, ECR repos)
    main.tf
    backend.tf
  .github/workflows/
    plan.yml                  # on PR
    apply.yml                 # on merge to main
  .tflint.hcl
  Makefile
```

**One state file per env.** A single state file across envs means a `terraform apply` typo can wipe production. Don't.

**Modules are local refs**, not separately versioned at first. You'll be tempted to publish modules to a registry. Don't until you have a second consumer team — until then it's overhead.

## Module structure (per module)

```hcl
# modules/database/variables.tf
variable "name"        { type = string }
variable "environment" { type = string }
variable "vpc_id"      { type = string }
variable "subnet_ids"  { type = list(string) }
variable "instance_class" {
  type    = string
  default = "db.t4g.medium"
}

# modules/database/main.tf
resource "aws_db_instance" "this" {
  identifier             = "${var.name}-${var.environment}"
  engine                 = "postgres"
  engine_version         = "16.4"
  instance_class         = var.instance_class
  allocated_storage      = 20
  storage_encrypted      = true
  vpc_security_group_ids = [aws_security_group.db.id]
  db_subnet_group_name   = aws_db_subnet_group.this.name
  skip_final_snapshot    = var.environment != "prod"
  deletion_protection    = var.environment == "prod"
  backup_retention_period = var.environment == "prod" ? 30 : 7
  tags = { Environment = var.environment }
}

# modules/database/outputs.tf
output "endpoint" { value = aws_db_instance.this.endpoint }
output "id"       { value = aws_db_instance.this.id }
```

The `var.environment` switches behavior for prod (deletion protection, longer backups, no skip_final_snapshot). One module, three envs, no copy-paste.

## State backend (S3 + DynamoDB)

```hcl
# envs/prod/backend.tf
terraform {
  backend "s3" {
    bucket         = "acme-tfstate-prod"
    key            = "infra/prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "acme-tfstate-locks"
    encrypt        = true
  }
}
```

The bucket itself bootstraps in `global/` with versioning + encryption + public-access-block + lifecycle for old versions. Lock table is one PAY_PER_REQUEST DynamoDB table, $0/month.

## CI: plan on PR, apply on merge

```yaml
# .github/workflows/plan.yml
name: terraform plan
on:
  pull_request:
    paths: ['infra/**']

jobs:
  plan:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        env: [dev, staging, prod]
    permissions:
      id-token: write             # OIDC to AWS
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123:role/tf-plan
          aws-region: us-east-1
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.9.0
      - run: terraform init
        working-directory: infra/envs/${{ matrix.env }}
      - run: terraform plan -no-color -out=plan.bin
        working-directory: infra/envs/${{ matrix.env }}
      - run: terraform show -no-color plan.bin > plan.txt
        working-directory: infra/envs/${{ matrix.env }}
      - uses: actions/upload-artifact@v4
        with:
          name: plan-${{ matrix.env }}
          path: infra/envs/${{ matrix.env }}/plan.bin
      - name: Comment plan on PR
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('infra/envs/${{ matrix.env }}/plan.txt', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `### Plan: ${{ matrix.env }}\n\`\`\`\n${plan.slice(0, 60000)}\n\`\`\``
            });
```

Use **OIDC** (federated identity) to assume IAM roles, never long-lived AWS keys in GH secrets. Long-lived keys leaking is the #1 Terraform incident.

## Drift detection (the part nobody runs)

Schedule a daily `terraform plan -detailed-exitcode` job; alert on exit 2 (drift detected). Drift = someone clicked in the console. Catch it within 24h or lose the will to track it.

## Anti-patterns

- **Single state file across envs** — one bad apply = prod wipe.
- **Workspaces for environments** — they hide which env you're in. Directories are explicit.
- **`-target` to apply parts** — manageable in dev, lethal in prod (skips dependencies). Fix the config instead.
- **Secrets in `.tfvars` or git** — even with `.gitignore`, they end up in shell history and CI logs. Pull from secret manager via data source.
- **`terraform apply` from a laptop to prod** — apply runs in CI only, with audit trail. Local `apply` on prod is how teams overwrite production from a stale checkout.
- **Modules in a separate repo before you have a second consumer** — version drift, PR-across-repos, slower iteration. Promote later.
- **Provider versions unpinned** — every `apply` becomes a roulette. `~> 5.40` at minimum, exact version for prod.
- **Terraform + manual console clicks on the same resources** — drift forever. One control plane.
- **`count = var.enabled ? 1 : 0`** for optional resources without `for_each` — refactor pain when count flips.
- **No `tfsec`/`checkov`** — public S3 buckets, unencrypted RDS, IMDSv1 — these stay as default unless a scanner yells.

## Verify it worked

- [ ] `terraform plan` in dev/staging/prod produces three different plans (no env mixing).
- [ ] State files are in three different S3 keys, encrypted, versioned.
- [ ] DynamoDB lock prevents concurrent applies (`apply` from two terminals — second fails fast).
- [ ] OIDC role works from CI; no AWS access keys exist as GitHub secrets.
- [ ] Plan is posted as a PR comment on every Terraform PR.
- [ ] `tfsec` and `checkov` run in CI; both green.
- [ ] Daily drift job runs; intentionally clicking in console triggers an alert within 24h.
- [ ] Module README exists for every module (inputs, outputs, example).
- [ ] Production destroy is blocked: `deletion_protection = true` for stateful resources.
- [ ] `terraform fmt -check` and `terraform validate` are green in pre-commit.
