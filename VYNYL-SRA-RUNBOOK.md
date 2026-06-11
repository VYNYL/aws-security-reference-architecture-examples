# VYNYL SRA Deployment Runbook (`vynyl-sra-base`)

This branch is VYNYL's maintained base for deploying the AWS Security Reference Architecture
(SRA) to **non–Control Tower** AWS Organizations. It is the upstream `aws-samples`
SRA at commit **`f3c57b7`** plus a small set of generic bug fixes (below) that are required
for the SRA to deploy successfully against the **current AWS APIs**.

> **Deploy any new org's `easy_setup` CodeBuild project from this branch.** All
> org-specific values (account IDs, regions, emails, which solutions to enable) go in the
> **CloudFormation stack parameters**, never in this branch.

First proven on the OCR org (June 2026). OCR runs the equivalent `ocr206-cloudtrail-fix` branch.

---

## What's patched vs. upstream `f3c57b7`

| # | File | Change | Why |
|---|---|---|---|
| 1 | `solutions/cloudtrail/cloudtrail_org/lambda/src/app.py` | data-event ARNs `arn:aws:s3:::*`→`arn:aws:s3:::`, `:lambda:*`→`:lambda` | Current CloudTrail API **rejects** the upstream wildcard forms (`InvalidEventSelectorsException`). Without this, enabling management events fails and can leave the stack `UPDATE_ROLLBACK_FAILED`. |
| 2 | `solutions/guardduty/guardduty_org/lambda/src/guardduty.py` | `check_members` made **non-fatal** + **per-region client**; `CHECK_ACCT_MEMBER_RETRIES` 10→30 | Upstream waits only ~100s for GuardDuty member association. First-time org enablement of never-before-enabled accounts exceeds that → "Check members failure" → rollback. Org auto-enable enrolls lagging members in the background, so proceeding (with a warning) is safe. |
| 3 | `easy_setup/templates/sra-easy-setup.yaml` | CodeBuild image `aws/codebuild/standard:5.0`→`7.0` | `standard:5.0`'s Python 3.9 is **EOL** and can't `pip install` recent boto3. `7.0` (Python 3.11/3.12) de-EOLs the builder and enables boto3 1.43.x. |
| 4 | `solutions/cloudtrail/cloudtrail_org/layer/boto3/package.txt` | pin `boto3;1.43.24` (+ layer `Description`) | Refreshes the vendored boto3 layer for Amazon Inspector readiness (no stale-dependency findings). |

All four are **org-agnostic** (no account IDs / region pins). Confirm with:
`git diff f3c57b7 vynyl-sra-base -- aws_sra_examples/`

---

## Deploying a new (non-CT) org

1. **Prereqs:** AWS Organizations with all-features; a delegated **Security/Audit** account and a
   **Log Archive** account; admin access to the **management** account; this fork is **public**
   (CodeBuild clones it with no credentials) and keeps the repo name
   `aws-security-reference-architecture-examples`.
2. **Deploy `easy_setup`** (`aws_sra_examples/easy_setup/templates/sra-easy-setup.yaml`) into the
   management account's home region, with parameters set for the org, and critically:
   - `pRepoURL = https://github.com/VYNYL/aws-security-reference-architecture-examples.git`
   - `pRepoBranch = vynyl-sra-base`
   - `pControlTower = false`
   - Per-solution flags (`pDeploy<Service>Solution`) for what you want.
3. **Security Hub gotcha:** Security Hub's nested stack only deploys when
   `(pDeployConfigManagementSolution ∈ {Yes, "Already Deployed"}) AND pDeploySecurityHubSolution = Yes`.
   If AWS Config already records in the management account, set
   `pDeployConfigManagementSolution = "Already Deployed"`. Setting only `pDeploySecurityHubSolution=Yes`
   is a **silent no-op**.
4. **Region scope** is the main Security Hub cost lever. Security Hub defaults to **governed regions only**
   (its `pControlTowerRegionsOnly` default is `true`, unset by easy_setup). GuardDuty defaults to **all**
   enabled regions (`pGuardDutyCustomerGovernedRegionsOnly=false`); set it `true` to scope GuardDuty to
   governed regions too (do this if you also plan region-blocking SCPs).
5. **Cost:** GuardDuty + Security Hub have recurring cost; GuardDuty includes a **30-day free trial** per
   account/region — measure actuals before committing.

---

## Operational playbook (this is NOT a normal CloudFormation stack)

- **Use `aws cloudformation update-stack`, NOT change sets.** Nested CloudTrail/Config templates use
  cross-account `{{resolve:secretsmanager}}` references that **fail at change-set creation** but resolve
  fine on a direct `update-stack`. (You lose pre-execute diff review; rely on known param deltas + rollback.)
- **CodeBuild does NOT auto-run on stack updates.** Its custom resource has a static `ServiceToken`, so it
  fires only at stack **creation**. To (re)stage solution artifacts you must run it explicitly:
  ```bash
  aws codebuild start-build --project-name sra-codebuild-project --region <home-region>
  # stage a specific branch/commit without changing the stack:
  #   --environment-variables-override name=SRA_REPO_URL,value=<git-url>,type=PLAINTEXT \
  #                                    name=SRA_REPO_BRANCH_NAME,value=<branch-or-sha>,type=PLAINTEXT
  ```
- **Lambda layer/code S3 keys are static** → re-staging overwrites the zip but CloudFormation publishes
  **no new LayerVersion** for already-deployed solutions. To force a vendored-dependency refresh, bump the
  `AWS::Lambda::LayerVersion` `Description` (immutable resource) so a new version is published.
- **"Only X changes" is not achievable.** Any parent `update-stack` re-evaluates nested stacks and
  re-publishes Lambdas; foundational stacks (Config, etc.) show as `Modify` even with identical templates.
  Benign, but expected.
- **Security Hub = "Security Hub CSPM"** (classic standards/posture). The newer **AWS Security Hub
  (HubV2 / exposure findings)** is a separate product, **not** deployed by SRA (manual `enable-security-hub-v2`).

---

## Recovery runbook

**Stuck `UPDATE_ROLLBACK_FAILED` from a deeply-nested custom resource** (e.g. CloudTrail Lambda):
`--resources-to-skip` accepts only one nesting level and `continue-update-rollback` can't run on child
stacks. Instead **make the rollback succeed**: patch the failing Lambda's code out-of-band
(`aws lambda update-function-code` with a fixed zip), then `aws cloudformation continue-update-rollback`
on the root (no skip). The next proper deploy from the branch reconciles the code.

**GuardDuty enablement fails ("Check members failure", or delivery StackSet errors):** failed attempts
leave **orphans** that block retries. Before retrying, clean up **all** of them, org-wide, in the enabled regions:
- Standalone member-account **detectors** (no admin) — `aws guardduty list-detectors`/`delete-detector` in
  each member account. Verify they're today's artifacts via `get-detector` `CreatedAt` before deleting.
- The **delivery S3 bucket** (`sra-guardduty-org-delivery-<logarchive-acct>-<region>`) in Log Archive — empty + delete.
- Orphaned **delivery KMS keys** ("SRA GuardDuty Delivery Key", no alias/grants) — these go to
  `PendingDeletion` automatically; they hold no retained data (failed attempts never export findings).
- Confirm no GuardDuty org admin remains (`list-organization-admin-accounts`).

---

## Maintenance

- **Base is `f3c57b7`** (pre–Control Tower 4.0), chosen because CT 4.0 (#344) restructures Config/CloudTrail/
  S3 log buckets and is untested in non-CT mode. For non-CT orgs that's the right base.
- **Adopting upstream updates (CT 4.0 / latest `main`) is a deliberate, tested upgrade** — not a merge
  side-effect: rebase these fixes onto the new base, resolve the CloudTrail-template conflict, validate
  Config/CloudTrail delivery, then repoint deployments. Don't `git merge` this branch into `main`.
- The patched bugs are genuine upstream defects; `aws-samples` doesn't accept PRs, but filing GitHub
  **issues** could get them fixed upstream and shrink this fork's maintenance burden.
