# GCP Security Workflows

Reusable GitHub Actions workflows for a minimal GCP-only security scanning pipeline:

1. DEV: Trivy filesystem, secret, and config scan on push and pull request.
2. BUILD: Trivy container image scan after image build and before push.
3. DEPLOY: Prowler scan against live GCP configuration after `terraform apply`.
4. NETWORK: Nmap plus an OpenVAS/GVM integration placeholder on a nightly schedule.

The real workflow logic lives in this central repository. Individual infrastructure
repositories should call these files with GitHub Actions reusable workflows, as shown in
`gcp-infra-repo-example/.github/workflows/security.yml`.

## Required Configuration

| Name | Type | Used by | Purpose |
| --- | --- | --- | --- |
| `WIF_PROVIDER` | Secret | all reusable workflows | Full GCP Workload Identity Provider resource name used by `google-github-actions/auth`. |
| `RESULTS_UPLOADER_SA` | Secret | all reusable workflows | Narrowly scoped service account used to upload scan output to GCS. Grant it `roles/storage.objectCreator` on the results bucket. |
| `PROWLER_SA` | Secret | `deploy-scan.yml` | GCP service account email that Prowler impersonates to scan live project configuration. |
| `GCP_PROJECT_ID` | Repository variable | caller deploy job | Project ID passed to the reusable deploy scan after Terraform succeeds. |
| `RESULTS_BUCKET` | Repository variable | all reusable workflows | GCS bucket name passed as the required `results_bucket` input for durable scan result storage. |
| `SCAN_TARGET_CIDR` | Repository variable | caller network job | CIDR range or target list used by the nightly network scan. |

Use Workload Identity Federation only. Do not create or store static GCP JSON service
account keys for these workflows.

Each reusable workflow requires a `results_bucket` input. Results are uploaded with
`gcloud storage cp -r` into stage-specific prefixes:

| Workflow | GCS prefix |
| --- | --- |
| `dev-scan.yml` | `gs://<results_bucket>/dev/<run_id>/` |
| `build-scan.yml` | `gs://<results_bucket>/build/<sha>/` |
| `deploy-scan.yml` | `gs://<results_bucket>/deploy/<sha>/` |
| `network-scan.yml` | `gs://<results_bucket>/network/<sha>/` |

Create the results bucket before using these workflows. Recommended bucket settings:
uniform bucket-level access, public access prevention, object versioning, and a lifecycle
rule such as 365-day expiration. Scan output can reveal vulnerabilities, secrets,
reachable services, and cloud misconfigurations, so treat the bucket as sensitive.

## Workflow Notes

`dev-scan.yml` blocks on Trivy findings because it runs before code is merged. This is the
lowest-cost place to fix dependency, secret, and IaC configuration issues.

`build-scan.yml` scans the built container image and blocks only on `CRITICAL` and `HIGH`
findings. It uploads SARIF as an artifact and attempts to publish the same SARIF to GitHub
code scanning. It also copies the SARIF into GCS for longer-term retention.

`deploy-scan.yml` authenticates to GCP through GitHub OIDC and Workload Identity Federation,
then runs `prowler gcp --project-ids <project_id>`. Prowler findings do not fail the job
because the infrastructure is already live after `terraform apply`; the workflow uploads
the raw report artifacts for alerting and follow-up, then copies the same output directory
to GCS.

`network-scan.yml` is designed for scheduled execution only from caller repositories. It
runs Nmap with service detection and leaves a clear OpenVAS/GVM placeholder because real
GVM integrations depend on your organization's manager endpoint, authentication, scanner
IDs, and report export format. It uploads the network result directory both as a GitHub
artifact and to GCS.
