# Agent handoff: EAP-22958 template Preview validation and pipeline cleanup

Last updated: 2026-11-10. Written outside the repo so it is not committed. Read this first if you are resuming the work with a new agent.

## 1. Goal

The backlog task is to fix or decommission Azure DevOps consumer pipelines that are not working.

Steps as requested by the user:

1. Create a feature branch in the templates repo and enable the template Preview validation steps (uncomment the build-validation part that compiles consumer pipelines).
2. Raise a test PR and read the Allure report to see which consumer pipelines fail.
3. Analyse each project's pipelines and decide which to keep and which to remove.
4. Raise a separate PR to remove the unneeded pipelines.

Current stage: step 2 is done. Step 3 (classification) is complete for corp-ado-023, 006, 009, and 011. corp-ado-013 has 20 of 21 classified, with 1 failure still missing. The branch is pushed per local tracking (`origin/EAP-22958-enable-template-preview-validation` = `063e8e3`). Confirm on the remote in Azure DevOps.

## 2. Repositories and paths

- Templates repo (the "ES" repo under analysis): `es-api-shared-ado-templates`
  - Windows (UNC): `\\wsl.localhost\Ubuntu\home\rishithasj\Rishitha\corp-ado-023\es-api-shared-ado-templates`
  - WSL: `/home/rishithasj/Rishitha/corp-ado-023/es-api-shared-ado-templates`
- Workspace opened in the editor: `corp-ado-013` (`enterprise-pricing-services`). Treat as read-only for this task.
- Azure DevOps org: `cantire`. Projects in scope: `corp-ado-006`, `corp-ado-009`, `corp-ado-011`, `corp-ado-013`, `corp-ado-023`.
- Feature branch: `EAP-22958-enable-template-preview-validation`
- Tag on main: `1.4.0`

## 3. Git state

- `main` = `origin/main` = `cc614cb` (tag `1.4.0`).
- Feature branch: commit `063e8e3` ("Enable template Preview validation and Allure report"), one commit ahead of `origin/main`. Pushed per local tracking ref. The remote was not re-checked because it requires authentication.
- Commit contents (3 files, 20 insertions, 17 deletions):
  - `infra/ado/steps/validate-release.yml`: Preview compile step enabled and `publish-allure-report.yml` template enabled. The yaml-lint block stays commented out.
  - `VERSION`: `1.4.0` changed to `1.4.1`.
  - `CHANGELOG.md`: new `## [1.4.1]` entry above `## [1.4.0]`.
- Line endings: repo uses LF. A CRLF working copy caused full-file diffs earlier. The conversion to LF is committed.
- Stash `stash@{0}` ("preview-validation") still exists. Review, then drop.
- Unrelated pre-existing change: `infra/mongoDbAtlas/scripts/mongodb_restore.sh` (line endings). Not committed. Leave it alone.

### Git commands that work here

Run git inside WSL from PowerShell. Use `--no-pager` to avoid hangs. Do not chain with `&&` or `;` in PowerShell.

```
C:\Windows\System32\wsl.exe -d Ubuntu -- git --no-pager -C /home/rishithasj/Rishitha/corp-ado-023/es-api-shared-ado-templates status --short --branch
```

If git reports dubious ownership on the UNC path, pass `-c safe.directory=/home/rishithasj/Rishitha/corp-ado-023/es-api-shared-ado-templates` for each command.

## 4. What the validation step does

Files:

- `infra/ado/stages/validate-release.yml`: PR-triggered Validate job. Imports variable group `validation-pat`.
- `infra/ado/steps/validate-release.yml`: steps for the Validate stage. Runs `./infra/ado/script/Test-TemplateCompile.ps1` with `ADO_PREVIEW_PAT: $(validation-pat)`.
- `infra/ado/script/Test-TemplateCompile.ps1` (about 470 lines, PowerShell 7):
  - Requires `ADO_PREVIEW_PAT`. Rejects unresolved `$(...)` values.
  - For each project, lists pipelines with `GET {org}/{project}/_apis/pipelines?api-version=7.1&$top=100`, following continuation tokens. Discovery errors are recorded as the Allure item "Pipeline discovery - <project>".
  - Fetches each definition, selects YAML pipelines, then calls `POST .../_apis/pipelines/{id}/preview?api-version=7.1` with `previewRun=true` and the PR branch and SHA as the repo resource.
  - Writes Allure results. Per-pipeline preview failures are all recorded with the same generic message: "Preview validation failed; see the Azure task log for diagnostic details."
  - Throws at the end if any failures were recorded.
- `infra/ado/steps/publish-allure-report.yml`: publishes the Allure report (`PublishAllureReport@2`, version 2.36.0, `condition: always()`, `continueOnError: true`).

Known issues in the current YAML:

- `README.md` says `validation-pat` is not imported into the Validate job. That is out of date. Update the README.
- The README also warns: "Before mapping a PAT into PR-controlled YAML, move Preview validation behind a protected pipeline or required template." Still unresolved. A PR author can change the YAML to print the secret.

## 5. Test build and results

- Test build: `2235432` ("Enable template Preview validation and Allure report", branch `EAP-22958-enable-template-preview-validation`, pipeline `templates-validation`).
- History:
  1. First run: every project failed at discovery with the generic message "Pipeline discovery failed; see the Azure task log for details."
  2. Root cause: the PAT in the `validation-pat` variable group had expired. The task log returned an HTML "Access Denied: The Personal Access Token used has expired" page.
  3. The user created a new PAT and updated the variable group. The guidance given was scope "Build: Read & execute", and "Code: Read" only if preview fails on repository access (not verified). The user has confirmed the update.
  4. Rerun: discovery passes for all five projects. Preview runs then fail per pipeline.
- Allure totals for build 2235432 (per project, failed/passed):
  - corp-ado-006: 45 / 146
  - corp-ado-009: 27 / 130
  - corp-ado-013: 21 / 245
  - corp-ado-011: 11 / 82
  - corp-ado-023: 7 / 48
  - Total: 111 failed, 651 passed
- All 111 failures share one message in Allure. The real error for each pipeline is in the task log of step "Compile consumer pipelines against this commit", on lines `🔎 Previewing <project>/<id> (<name>)` followed by `❌ Preview failed for ...`.

## 6. Classification: corp-ado-023 (7 failures, all analysed)

| Pipeline (ID) | Error from task log | Suggested action |
|---|---|---|
| asb-kafka-connect (retired) (4652) | Build pipeline is disabled | Remove |
| Policy change detection (test) (14335) | Build pipeline is disabled | Remove |
| es-api-azdo-policy (15453) | `/pipeline/azdo-policy.yml` not found in `es-api-azdo-policy` on `main` | Remove (path not found) |
| Import ingress-nginx resources helm chart to ACR (confluent) (4274) | `/app/.pipelines/acr/confluent/pipeline.yml` not found in `es-api-shared-infra` on `main` | Remove (path not found) |
| Import ingress-nginx controller helm chart to ACR (625) | Template `/helm-chart/publish-to-acr.yml` not found in templates repo | Remove (template path not found) |
| Import ingress-nginx resources helm chart to ACR (627) | Same template path as 625 | Remove (template path not found) |
| manage-flink-batch-jobs (10778) | "The pipeline must contain at least one job with no dependencies" | Needs clarification (check whether templates from this repo are involved). |

Verified: `infra/acr/helm-chart/jobs/publish-to-acr.yml` exists in this repo. The consumer references `/helm-chart/publish-to-acr.yml`, which does not exist at that path.

Not yet verified: whether pipelines 4274, 15453, and 10778 are still in use. Use last-run date and target environment existence in Azure DevOps. Under the reporting rule in section 10, 625 and 627 are marked Remove. Before the removal PR, optionally confirm with `git log --follow` on `infra/acr/helm-chart/jobs/publish-to-acr.yml` that the template was not moved from `helm-chart/`.

## 7. Project status

- corp-ado-023: 7 failed, classified (6 Remove, 1 Needs clarification).
- corp-ado-006: 45 failed, classified (43 Remove, 2 Needs clarification).
- corp-ado-009: 27 failed, classified (24 Remove, 3 Needs clarification).
- corp-ado-011: 11 failed, classified (10 Remove, 1 Needs clarification).
- corp-ado-013: 21 failed, 20 classified (14 + 3 Remove, 3 Needs clarification). 1 failure still missing. Ask the user for it.

Section 10 holds the per-pipeline rows and is the source for the Confluent report.

## 8. Decisions and open items

1. Test PR open in Azure DevOps (cantire org): !247267 "Enable template Preview validation and Allure report", not merged. Its `templates-validation` build fails with 111 issues, as expected. Do not merge.
2. PAT security: resolve before merging. Options: move Preview validation behind a protected pipeline or required template, or get sign-off to accept the risk. The PAT is referenced in PR-controlled YAML.
3. PAT lifecycle: the PAT expired once already. Consider a service account owner and a calendar reminder for the expiry date.
4. Update the stale README note about `validation-pat` in the Validate job.
5. Drop `stash@{0}` after review.
6. Removal PR (separate): contains only pipelines confirmed unused. Pipelines to remove are listed in section 6 and section 7 once analysed.
7. Re-run the PR build after fixes to confirm the per-pipeline failures drop.

## 9. How to resume

1. Read this file and `git status` in the templates repo.
2. For the open corp-ado-013 failure, ask the user to paste the task-log error line. Do not ask for the PAT.
3. Classify using the table format in section 6.
4. Do not change pipeline definitions in other repos without asking. Edits in this repo are on the feature branch only.
5. Git account: the Git identity may change during the task. The branch was pushed under the previous account. The stash is local only. Confirm the active account before any further push. Do not record credentials here.

Do not record tokens or PAT values in this file or in chat.

## 10. Reporting format (Confluent page, pending)

The final report goes to a Confluent page. The page setup is not yet known, so this chart is the working source. Add rows for each project as it is analysed.

Rules:
- Remove: pipeline is disabled, or a YAML or template path is not found.
- Needs clarification: cause is unclear or depends on usage. Do not guess.

Columns: Project, Pipeline, Link, Cause, Action.

Link format (assumed, verify in Azure DevOps): `https://dev.azure.com/cantire/<project>/_build?definitionId=<id>`

### corp-ado-023 (7 failures)

| Project | Pipeline | Link | Cause | Action |
|---|---|---|---|---|
| corp-ado-023 | asb-kafka-connect (retired) | [4652](https://dev.azure.com/cantire/corp-ado-023/_build?definitionId=4652) | Pipeline disabled | Remove |
| corp-ado-023 | Policy change detection (test) | [14335](https://dev.azure.com/cantire/corp-ado-023/_build?definitionId=14335) | Pipeline disabled | Remove |
| corp-ado-023 | es-api-azdo-policy | [15453](https://dev.azure.com/cantire/corp-ado-023/_build?definitionId=15453) | YAML not found: `/pipeline/azdo-policy.yml` on `main` | Remove |
| corp-ado-023 | Import ingress-nginx resources helm chart to ACR (confluent) | [4274](https://dev.azure.com/cantire/corp-ado-023/_build?definitionId=4274) | YAML not found: `/app/.pipelines/acr/confluent/pipeline.yml` on `main` | Remove |
| corp-ado-023 | Import ingress-nginx controller helm chart to ACR | [625](https://dev.azure.com/cantire/corp-ado-023/_build?definitionId=625) | Template not found: `/helm-chart/publish-to-acr.yml` | Remove |
| corp-ado-023 | Import ingress-nginx resources helm chart to ACR | [627](https://dev.azure.com/cantire/corp-ado-023/_build?definitionId=627) | Template not found: `/helm-chart/publish-to-acr.yml` | Remove |
| corp-ado-023 | manage-flink-batch-jobs | [10778](https://dev.azure.com/cantire/corp-ado-023/_build?definitionId=10778) | No job without dependencies | Needs clarification |

### corp-ado-006 (45 failures)

Classified so far: 45 of 45 (batches 1 to 4). Complete. Grouped by cause.

| Project | Pipeline | Link | Cause | Action |
|---|---|---|---|---|
| corp-ado-006 | (Deprecated) Deploy ingress-nginx rules with tls cert and custom header validation for mongo | [4543](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=4543) | YAML not found: `/app/.pipelines/aks/ingress/controller-resources-mongo/pipeline.yml` on `main` | Remove |
| corp-ado-006 | (Deprecated) Deploy ingress-nginx rules with tls cert for transform-* services | [6019](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=6019) | YAML not found: `/app/.pipelines/aks/ingress/controller-resources-transform/pipeline.yml` on `main` | Remove |
| corp-ado-006 | azcopy-pipeline | [4611](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=4611) | YAML not found: `/app/.pipelines/storage-accounts/pipeline.yml` on `main` | Remove |
| corp-ado-006 | Copy Prod CosmosDB data to PreProd CosmosDB | [4645](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=4645) | YAML not found: `/infra/.pipelines/copy-prod-data-to-preprod.yaml` on `main` | Remove |
| corp-ado-006 | enterprise-product-services | [11182](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=11182) | YAML not found: `/app/.pipelines/aks/ingress/controller-resources/rotate_certs_pipeline.yml` on `main` | Remove |
| corp-ado-006 | inbound-adapters-ims-function | [1509](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=1509) | YAML not found: `/app/.pipelines/functions/inbound-adapters-ims/pipeline.yml` on `main` | Remove |
| corp-ado-006 | inbound-adapters-pcm-function | [1588](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=1588) | YAML not found: `/app/.pipelines/functions/inbound-adapters-pcm/pipeline.yml` on `main` | Remove |
| corp-ado-006 | inbound-adapters-pim-function | [1589](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=1589) | YAML not found: `/app/.pipelines/functions/inbound-adapters-pim/pipeline.yml` on `main` | Remove |
| corp-ado-006 | inbound-adapters-pmm-function | [1590](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=1590) | YAML not found: `/app/.pipelines/functions/inbound-adapters-pmm/pipeline.yml` on `main` | Remove |
| corp-ado-006 | Daily reconciliation tests | [5488](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=5488) | Pipeline disabled | Remove |
| corp-ado-006 | data-ingester-pcm-pulsar | [7971](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=7971) | Pipeline disabled | Remove |
| corp-ado-006 | data-ingester-pulsar | [8061](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=8061) | Pipeline disabled | Remove |
| corp-ado-006 | Cosmos DB SQL restore | [814](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=814) | Required parameters not supplied: `environment`, `file_name` | Needs clarification |
| corp-ado-006 | infra-dev-test-cc | [11809](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=11809) | YAML not found: `/infra/src/dev/test-cc/azure-pipelines.yml` on `main` | Remove |
| corp-ado-006 | infra-pprd-cc | [2917](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=2917) | YAML not found: `/infra/src/pprd/cc/azure-pipelines.yml` on `main` | Remove |
| corp-ado-006 | liquibase-mongodb-dev | [5123](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=5123) | YAML not found: `/app/.pipelines/liquibase/mongodb/pipeline-dev.yml` on `main` | Remove |
| corp-ado-006 | liquibase-mongodb-pprd | [5126](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=5126) | YAML not found: `/app/.pipelines/liquibase/mongodb/pipeline-pprd.yml` on `main` | Remove |
| corp-ado-006 | liquibase-mongodb-prod | [5127](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=5127) | YAML not found: `/app/.pipelines/liquibase/mongodb/pipeline-prod.yml` on `main` | Remove |
| corp-ado-006 | liquibase-mongodb-stage | [5125](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=5125) | YAML not found: `/app/.pipelines/liquibase/mongodb/pipeline-stage.yml` on `main` | Remove |
| corp-ado-006 | liquibase-mongodb | [4314](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=4314) | YAML not found: `/app/.pipelines/liquibase/mongodb/pipeline.yml` on `main` | Remove |
| corp-ado-006 | microservice-stage-secondary-app-only | [13483](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=13483) | YAML not found: `/app/.pipelines/bulk-deployment/pipeline-secondary.yml` on `main` | Remove |
| corp-ado-006 | microservice-stage-secondary-full-deploy | [13520](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=13520) | YAML not found: `/app/.pipelines/bulk-deployment/pipeline-secondary.yml` on `main` | Remove |
| corp-ado-006 | mongo-schema productdb-raw PPRD | [6063](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=6063) | YAML not found: `/app/.pipelines/liquibase/mongodb/productdb-raw/pipeline-pprd.yml` on `main` | Remove |
| corp-ado-006 | mongo-schema productdb-sql PPRD | [6066](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=6066) | YAML not found: `/app/.pipelines/liquibase/mongodb/productdb-sql/pipeline-pprd.yml` on `main` | Remove |
| corp-ado-006 | mongo-schema productdb-transformed-digital PPRD | [6069](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=6069) | YAML not found: `/app/.pipelines/liquibase/mongodb/productdb-transformed-digital/pipeline-pprd.yml` on `main` | Remove |
| corp-ado-006 | mongodb-pcmt-pprd | [6147](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=6147) | YAML not found: `/app/.pipelines/liquibase/mongodb/pcmt/pipeline-pprd.yml` on `main` | Remove |
| corp-ado-006 | Nightly reconciliation tests | [5489](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=5489) | Pipeline disabled | Remove |
| corp-ado-006 | notification-event-distributor-function | [1592](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=1592) | YAML not found: `/app/.pipelines/functions/notification-event-distributor/pipeline.yml` on `main` (enterprise-product-services) | Remove |
| corp-ado-006 | notification-function | [1591](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=1591) | YAML not found: `/app/.pipelines/functions/notification/pipeline.yml` on `main` (enterprise-product-services) | Remove |
| corp-ado-006 | Perf-PMM-upload-delta-file | [6052](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=6052) | YAML not found: `/v2/pipeline/PMM_upload_delta_file.yml` on `main` (enterprise-product-performance) | Remove |
| corp-ado-006 | Perf-Product-IMS-count-messages | [5768](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=5768) | YAML not found: `/v2/pipeline/Product_ims_count_messages.yml` on `main` (enterprise-product-performance) | Remove |
| corp-ado-006 | Perf-Send-messages-to-raw-updates-sbs-productFamily | [6858](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=6858) | YAML not found: `/v2/pipeline/Volume_testing_messages/Send_messages_to_raw_db_updates_sbs_pf.yml` on `main` (enterprise-product-performance) | Remove |
| corp-ado-006 | Perf-Send-messages-to-rawDB-updates-sbs | [6113](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=6113) | YAML not found: `/v2/pipeline/Send_messages_to_raw_db_updates_sbs.yml` on `main` (enterprise-product-performance) | Remove |
| corp-ado-006 | Perf-transform-digital-services-QA | [9201](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=9201) | YAML not found: `/v2/pipeline/Volume_testing_messages/Transform_digital_services_on_qa` on `main` (enterprise-product-performance; no `.yml` extension) | Remove |
| corp-ado-006 | POC-Integration-with-APIM-onboarding | [3772](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=3772) | YAML not found: `/app/.pipelines/integration-with-apim/pipeline.yml` on `main` (enterprise-product-services) | Remove |
| corp-ado-006 | process-triggers | [6822](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=6822) | YAML not found: `/infra/.pipelines/mongodb/apps/triggerApp/triggers/process-triggers.yml` on `main` (enterprise-product-services) | Remove |
| corp-ado-006 | Prod Cosmos DB SQL backup | [1177](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=1177) | Pipeline disabled | Remove |
| corp-ado-006 | productdb-raw-pulsar-pprd | [7797](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=7797) | YAML not found: `/app/.pipelines/liquibase/mongodb/productdb-raw-pulsar/pipeline-pprd.yml` on `main` (enterprise-product-services) | Remove |
| corp-ado-006 | productdb-v1-pulsar-pprd | [8058](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=8058) | YAML not found: `/app/.pipelines/liquibase/mongodb/productdb-v1-pulsar/pipeline-pprd.yml` on `main` (enterprise-product-services) | Remove |
| corp-ado-006 | SPN secret rotation | [12795](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=12795) | Template not found: `/infra/spn/steps/spn-secret-rotation.yml`, referenced by `/infra/.pipelines/spn/pipeline-secret-rotation.yml` in `es-api-shared-ado-templates` at the feature branch (`063e8e3`). Secret rotation is security-sensitive | Needs clarification |
| corp-ado-006 | Stage Cosmos DB SQL backup | [1037](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=1037) | Pipeline disabled | Remove |
| corp-ado-006 | trigger-app-functions | [6821](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=6821) | YAML not found: `/infra/.pipelines/mongodb/apps/triggerApp/functions/process-functions.yml` on `main` (enterprise-product-services) | Remove |
| corp-ado-006 | troubleshoot-acr-pipeline | [4531](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=4531) | YAML not found: `/app/.pipelines/test/pipeline.yml` on `main` (enterprise-product-services) | Remove |
| corp-ado-006 | troubleshoot-pipeline | [4472](https://dev.azure.com/cantire/corp-ado-006/_build?definitionId=4472) | YAML not found: `/app/.pipelines/troubleshoot/pipeline.yml` on `main` (enterprise-product-services) | Remove |

Note: 31 pipelines fail with YAML not found on `main` in `enterprise-product-services` (commit `170977a`): nine in batch 1, thirteen in batch 2, six in batch 3, three in batch 4. Five more fail with YAML not found on `main` in `enterprise-product-performance` (commit `06fb8c0`), all in batch 3. Verify these paths in both repos before the removal PR. The `liquibase-mongodb-prod` pipeline (5127) is a production database migration, and the `Prod Cosmos DB SQL backup` pipeline (1177) is a production backup. Confirm both separately before removal.

Remaining: 0 failures. Classification for corp-ado-006 is complete: 43 Remove, 2 Needs clarification (Cosmos DB SQL restore 814, SPN secret rotation 12795). Confirm the production items (liquibase-mongodb-prod 5127, Prod Cosmos DB SQL backup 1177) before removal.

### corp-ado-009 (27 failures)

Classified so far: 27 of 27 (batches 1 and 2). Complete. Grouped by cause.

| Project | Pipeline | Link | Cause | Action |
|---|---|---|---|---|
| corp-ado-009 | [Deprecated]Deploy controller-nginx-mongoservices to AKS | [3896](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=3896) | YAML not found: `/app/.pipelines/aks/ingress/controller-nginx-mongoservices/pipeline.yml` on `main` (enterprise-inventory-services) | Remove |
| corp-ado-009 | [Deprecated]Deploy ingress-nginx rules with tls cert (confluent) | [4127](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=4127) | YAML not found: `/app/.pipelines/aks/ingress/controller-resources-confluent/pipeline.yml` on `main` (enterprise-inventory-services) | Remove |
| corp-ado-009 | [Deprecated]Deploy ingress-nginx rules with tls cert and custom header validation for mongo | [4542](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=4542) | YAML not found: `/app/.pipelines/aks/ingress/controller-resources-mongo/pipeline.yml` on `main` (enterprise-inventory-services) | Remove |
| corp-ado-009 | add trigger app test | [6726](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=6726) | YAML not found: `/infra/.pipelines/mongodb/apps/triggerApp/add-app.yml` on `main` (enterprise-inventory-services) | Remove |
| corp-ado-009 | Build ctc-cp-kafka-connect docker image | [4184](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=4184) | YAML not found: `/app/.pipelines/confluent/ctc-cp-kafka-connect-image/pipeline.yml` on `main` (enterprise-inventory-services) | Remove |
| corp-ado-009 | confluence-hub-plugins-and-image-to-Artifactory | [4155](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=4155) | YAML not found: `/app/.pipelines/confluent/upload-plugins-and-image-to-artifactory/pipeline.yml` on `main` (enterprise-inventory-services) | Remove |
| corp-ado-009 | confluence_openAi-pipeline | [14187](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=14187) | YAML not found: `/app/.pipelines/confluence_doc_poc/openai/pipeline.yml` on `main` (enterprise-inventory-services) | Remove |
| corp-ado-009 | conflunce_poc | [14177](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=14177) | YAML not found: `/app/.pipelines/confluence_doc_poc/pipeline.yml` on `main` (enterprise-inventory-services) | Remove |
| corp-ado-009 | Deploy Confluent components | [4104](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=4104) | YAML not found: `/app/.pipelines/aks/confluent/pipeline.yml` on `main` (enterprise-inventory-services) | Remove |
| corp-ado-009 | enterprise-inventory-services (6806) | [6806](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=6806) | YAML not found: `/infra/.pipelines/mongodb/apps/triggerApp/functions/process-functions.yml` on `main` (enterprise-inventory-services) | Remove |
| corp-ado-009 | enterprise-inventory-services (6808) | [6808](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=6808) | YAML not found: `/infra/.pipelines/mongodb/apps/triggerApp/triggers/process-triggers.yml` on `main` (enterprise-inventory-services) | Remove |
| corp-ado-009 | enterprise-inventory-services | [4019](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=4019) | YAML not found: `/app/.pipelines/mongo-poc/pipeline.yaml` on `main` (enterprise-inventory-services) | Remove |
| corp-ado-009 | infra-dev-scheduled | [15458](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=15458) | Pipeline disabled | Remove |
| corp-ado-009 | inventory-batch-processor | [15175](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=15175) | YAML not found: `/app/.pipelines/microservices/inventory/batch-processor/pipeline.yml` on `main` (enterprise-inventory-services) | Remove |
| corp-ado-009 | inventory-common | [1162](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=1162) | YAML not found: `/app/.pipelines/libraries/inventory-common/pipeline.yml` on `main` (enterprise-inventory-services) | Remove |
| corp-ado-009 | inventory-enterprise-api | [15176](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=15176) | YAML not found: `/app/.pipelines/microservices/inventory/enterprise-api/pipeline.yml` on `main` (enterprise-inventory-services) | Remove |
| corp-ado-009 | microservice-stage-secondary-app-only | [13664](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=13664) | YAML not found: `/app/.pipelines/microservices/bulk-deployment/pipeline-secondary.yml` on `main` (enterprise-inventory-services) | Remove |
| corp-ado-009 | MongoDB Restore Secondary | [13799](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=13799) | Pipeline disabled | Remove |
| corp-ado-009 | Perf-batch-processors | [14577](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=14577) | YAML not found: `/v2/pipeline/volume-testing/Inventory_batch_processors.yml` on `main` (enterprise-inventory-performance, commit `0fea8e9`) | Remove |
| corp-ado-009 | Perf_Volume_AFE_message_sender | [6562](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=6562) | YAML not found: `/azure-pipelines.yml` on `main` (enterprise-inventory-performance, commit `0fea8e9`) | Remove |
| corp-ado-009 | Renovate test | [11045](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=11045) | YAML not found: `/app/.pipelines/renovate/azure-kafka/pipeline.yaml` on `main` (enterprise-inventory-services) | Remove |
| corp-ado-009 | reservation-secrets | [357](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=357) | YAML not found: `/app/.pipelines/test.yml` on `main` (enterprise-inventory-services) | Needs clarification |
| corp-ado-009 | reservations-api | [15177](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=15177) | YAML not found: `/app/.pipelines/microservices/reservation/api-service/pipeline.yml` on `main` (enterprise-inventory-services) | Remove |
| corp-ado-009 | Restore Cosmos db Mongo API | [763](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=763) | YAML not found: `/infra/.pipelines/restore-cosmos-mongo-pipeline.yaml` on `main` (enterprise-inventory-services) | Remove |
| corp-ado-009 | Restore Cosmos db SQL API | [764](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=764) | Required parameters not supplied: `environment`, `file_name` | Needs clarification |
| corp-ado-009 | rfid-quantities-export-service | [15178](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=15178) | YAML not found: `/app/.pipelines/microservices/rfid/quantities-export-service/pipeline.yml` on `main` (enterprise-inventory-services) | Remove |
| corp-ado-009 | status-api | [15179](https://dev.azure.com/cantire/corp-ado-009/_build?definitionId=15179) | YAML not found: `/app/.pipelines/microservices/rfid/status-api/pipelline.yml` on `main` (enterprise-inventory-services). Filename looks misspelled (`pipelline.yml`) | Needs clarification |

Note: 12 of the 13 fail with YAML not found on `main` at commit `83798e7` in `enterprise-inventory-services`. Pipelines 6806 and 6808 reference the same `/infra/.pipelines/mongodb/apps/triggerApp/` paths as corp-ado-006 (6821, 6822). Verify the paths before the removal PR. The pipeline named `enterprise-inventory-services` (4019) has a generic name, so confirm its purpose before removal.

Remaining: 0 failures. Classification for corp-ado-009 is complete: 24 Remove, 3 Needs clarification (reservation-secrets 357, Restore Cosmos db SQL API 764, status-api 15179).

### corp-ado-011 (11 failures)

Classified so far: 11 of 11 (batch 1). Complete. Grouped by cause.

| Project | Pipeline | Link | Cause | Action |
|---|---|---|---|---|
| corp-ado-011 | liquibase-service (retired) | [714](https://dev.azure.com/cantire/corp-ado-011/_build?definitionId=714) | Pipeline disabled | Remove |
| corp-ado-011 | Perf-test-location-calendar-service | [3885](https://dev.azure.com/cantire/corp-ado-011/_build?definitionId=3885) | Pipeline disabled | Remove |
| corp-ado-011 | testing-import-export | [12002](https://dev.azure.com/cantire/corp-ado-011/_build?definitionId=12002) | Pipeline disabled | Remove |
| corp-ado-011 | exp pipeline exec | [14549](https://dev.azure.com/cantire/corp-ado-011/_build?definitionId=14549) | YAML not found: `/infra/.pipelines/experiments/keep-alive.yml` on `main` (enterprise-location-services, commit `b0ddafa`) | Remove |
| corp-ado-011 | mysql-backupdb | [856](https://dev.azure.com/cantire/corp-ado-011/_build?definitionId=856) | YAML not found: `/infra/.pipelines/MySQL-backup/mysqlbackupdb-pipeline.yml` on `main` (enterprise-location-services) | Remove |
| corp-ado-011 | mysqldbrestore-pipeline | [1033](https://dev.azure.com/cantire/corp-ado-011/_build?definitionId=1033) | YAML not found: `/infra/.pipelines/MySQL-backup/mysqlrestoredb-pipeline.yml` on `main` (enterprise-location-services) | Remove |
| corp-ado-011 | odpal-cfdcent-processor | [15476](https://dev.azure.com/cantire/corp-ado-011/_build?definitionId=15476) | YAML not found: `/app/.pipelines/microservices/odpal-cfdcent-processor/pipeline.yml` on `main` (enterprise-location-services) | Remove |
| corp-ado-011 | service-api | [15072](https://dev.azure.com/cantire/corp-ado-011/_build?definitionId=15072) | YAML not found: `/app/.pipelines/microservices/service-api/pipeline.yml` on `main` (enterprise-location-services) | Remove |
| corp-ado-011 | SQL maintanence DEV,QA | [10745](https://dev.azure.com/cantire/corp-ado-011/_build?definitionId=10745) | Template variables file not found: `/app/.pipelines/sqldb/variables/dev.yml`, referenced by `/app/.pipelines/sqldb/maintenance.yml` on `main` | Remove |
| corp-ado-011 | sql-create-user-script | [13480](https://dev.azure.com/cantire/corp-ado-011/_build?definitionId=13480) | YAML not found: `/app/.pipelines/sqldb/create-login-and-user.yml` on `main`. Creates SQL logins and users, so access-sensitive | Needs clarification |
| corp-ado-011 | test-pe | [11781](https://dev.azure.com/cantire/corp-ado-011/_build?definitionId=11781) | YAML not found: `/app/.pipelines/sqldb/test-pe.yml` on `main` (enterprise-location-services) | Remove |

Note: All YAML-not-found entries resolve against `enterprise-location-services` at commit `b0ddafa`. Verify the paths on `main` before the removal PR. The two MySQL backup and restore pipelines (856, 1033) share one folder, so confirm them together. Pipeline 10745 fails on a missing variables file, so the rest of the pipeline may still exist. Confirm before removal.

Remaining: 0 failures. Classification for corp-ado-011 is complete: 10 Remove, 1 Needs clarification (sql-create-user-script 13480).

### corp-ado-013 (21 failures)

Classified so far: 20 of 21 (batches 1 and 2). Grouped by cause.

| Project | Pipeline | Link | Cause | Action |
|---|---|---|---|---|
| corp-ado-013 | automation-tests-utaf2.0_pricing_API_vNext | [15068](https://dev.azure.com/cantire/corp-ado-013/_build?definitionId=15068) | Pipeline disabled | Remove |
| corp-ado-013 | automation-tests-utaf2.0_pricing_vNext | [10508](https://dev.azure.com/cantire/corp-ado-013/_build?definitionId=10508) | Pipeline disabled | Remove |
| corp-ado-013 | Deploy Ingress-nginx resources testmongo | [6587](https://dev.azure.com/cantire/corp-ado-013/_build?definitionId=6587) | Invalid YAML: `/app/.pipelines/aks/ingress/controller-resources-testmongo/pipeline.yml` (Line 32): resource `bulk-trigger-pipeline` input is invalid | Needs clarification |
| corp-ado-013 | Deploy Ingress-nginx resources vnext | [9435](https://dev.azure.com/cantire/corp-ado-013/_build?definitionId=9435) | YAML not found: `/app/.pipelines/aks/ingress/controller-resources-vnext/pipeline.yml` on `main` (enterprise-pricing-services) | Remove |
| corp-ado-013 | enterprise-pricing-services-gitops | [9952](https://dev.azure.com/cantire/corp-ado-013/_build?definitionId=9952) | YAML not found: `/helmcharts/dev/hello-world/azure-pipelines.yaml` on `main` (enterprise-pricing-services-gitops) | Remove |
| corp-ado-013 | export-pricining-db-dev | [11926](https://dev.azure.com/cantire/corp-ado-013/_build?definitionId=11926) | Template not found: `/infra/sqlServer/steps/powershell.yml`, referenced by `/app/.pipelines/sqldb/export-db.yml` in `es-api-shared-ado-templates` at the feature branch (`063e8e3`). Check `main` before removal | Remove |
| corp-ado-013 | mongo-schema | [4088](https://dev.azure.com/cantire/corp-ado-013/_build?definitionId=4088) | YAML not found: `/app/.pipelines/liquibase/mongodb/pipeline.yml` on `main` (enterprise-pricing-services) | Remove |
| corp-ado-013 | MongoDB Restore Secondary | [14135](https://dev.azure.com/cantire/corp-ado-013/_build?definitionId=14135) | Pipeline disabled | Remove |
| corp-ado-013 | New Relic infrastructure agent (POC) | [5924](https://dev.azure.com/cantire/corp-ado-013/_build?definitionId=5924) | Invalid YAML: `/app/.pipelines/aks/newrelic/infrastructure-agent-bundle/pipeline.yaml` (Line 34): resource `bulk-trigger-pipeline` input is invalid | Needs clarification |
| corp-ado-013 | New Relic mongodb-exporter (POC) | [5951](https://dev.azure.com/cantire/corp-ado-013/_build?definitionId=5951) | Invalid YAML: `/app/.pipelines/aks/newrelic/mongodb-exporter/pipeline.yaml` (Line 32): resource `bulk-trigger-pipeline` input is invalid | Needs clarification |
| corp-ado-013 | Perf-Pricing-APIs-suite | [9760](https://dev.azure.com/cantire/corp-ado-013/_build?definitionId=9760) | Pipeline disabled | Remove |
| corp-ado-013 | Perf-Pricing-composite-api | [3881](https://dev.azure.com/cantire/corp-ado-013/_build?definitionId=3881) | Pipeline disabled | Remove |
| corp-ado-013 | SQL Import Export Secondary | [14136](https://dev.azure.com/cantire/corp-ado-013/_build?definitionId=14136) | Pipeline disabled | Remove |
| corp-ado-013 | vNext Assign testProducts to zones | [12266](https://dev.azure.com/cantire/corp-ado-013/_build?definitionId=12266) | YAML not found: `/v2/pipeline/vNext_assign_products_to_zones.yml` on `main` (enterprise-pricing-performance) | Remove |
| corp-ado-013 | vNext Export API | [12264](https://dev.azure.com/cantire/corp-ado-013/_build?definitionId=12264) | YAML not found: `/v2/pipeline/vNext_export_api.yml` on `main` (enterprise-pricing-performance) | Remove |
| corp-ado-013 | vNext script | [12265](https://dev.azure.com/cantire/corp-ado-013/_build?definitionId=12265) | YAML not found: `/v2/pipeline/vNext_script.yml` on `main` (enterprise-pricing-performance) | Remove |
| corp-ado-013 | vnext-deploy-pricing-service-accounts | [10819](https://dev.azure.com/cantire/corp-ado-013/_build?definitionId=10819) | Pipeline disabled | Remove |
| corp-ado-013 | vnext-infrastructure-agent-bundle | [12290](https://dev.azure.com/cantire/corp-ado-013/_build?definitionId=12290) | Pipeline disabled | Remove |
| corp-ado-013 | vnext-mongo-exporter | [12289](https://dev.azure.com/cantire/corp-ado-013/_build?definitionId=12289) | Pipeline disabled | Remove |
| corp-ado-013 | vnext-opentelemetry-collector | [11318](https://dev.azure.com/cantire/corp-ado-013/_build?definitionId=11318) | Pipeline disabled | Remove |

Note: The three `bulk-trigger-pipeline` input errors (6587, 5924, 5951) share one cause. They may come from a shared resource definition, so fix them together. The `enterprise-pricing-services` YAML-not-found entries (9435, 4088) match corp-ado-006 and corp-ado-009 paths (`/app/.pipelines/liquibase/mongodb/pipeline.yml`, `/app/.pipelines/aks/ingress/...`).

Remaining: 1 failure (21 total, 20 received), to be added in the next batch.

