---
title: Fixing Automating Terraform with GitHub Actions
author: Matthew
date: 2023-03-03T14:47:19Z
---

I am involved in evaluating GitHub Actions as a part of migration activity, one of the technologies that is used in our CI/CD pipeline is Terraform.

Hashicorp provides a [tutorial](https://developer.hashicorp.com/terraform/tutorials/automation/github-actions) as a start for Automating Terraform with GitHub Actions.

Its a good start but in my opinion it is unusable for production environments as there is no [Interactive Approval of Plans](https://developer.hashicorp.com/terraform/tutorials/automation/automate-terraform#interactive-approval-of-plans) and instead it uses [Auto-Approval of Plans](https://developer.hashicorp.com/terraform/tutorials/automation/automate-terraform#auto-approval-of-plans) something thats discouraged in the Terraform documentation for production environments.

The way it uses PRs as planning mechanism gives the perception of the _Interactive Approval of Plans_, but once the changes are merged everything is recalculated and the PR plan is discarded. For most scenarios this will _probably_ be fine, but if there is an extended period of time between approval and merge the plan is redundant and _Auto-Approval of Plans_ could have unintended effects on the infrastructure. Reusing the plan will only apply the changes approved.

![review](/blog/images/posts/fixing-automating-terraform-with-github-actions-review.png "Deployment Review")

![plan summary](/blog/images/posts/fixing-automating-terraform-with-github-actions-plan-summary.png "Summary to Review")

The _Interactive Approval of Plans_ is possible using a combination of GitHub Actions features:

- Multiple Jobs with needs to add dependencies between them.
- Environments and [Protection Rules](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment#environment-protection-rules) for interactive approval of plans.
- [Step Summaries](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#adding-a-job-summary) to remove the need to dig into Job Logs to see the plan and apply outcome.
- [Artifacts](https://docs.github.com/en/actions/using-workflows/storing-workflow-data-as-artifacts) for sharing the plan between jobs.
- [Terraform Provider Plugin Cache](https://developer.hashicorp.com/terraform/cli/config/config-file#provider-plugin-cache) to speed up jobs storing and reusing downloaded plugins.

## Setting Up Repo

### Environment and Protection Rules

In the repository `Settings > Environment`, create a new environment and configure.

On the configure screen add the required reviewers. This will allow for the plan to be checked and approved before applying.

Optionally set the allowed deployment branch. and disable admins bypass protection rules.

![protection-rules](/blog/images/posts/fixing-automating-terraform-with-github-actions-protection-rules.png "Environment protection rules")

## Review Actions workflow

The workflow is built up of two jobs with dependency on each other:

```mermaid
graph TD;
    plan[Terraform Plan] --> apply[Terraform Apply]

    click plan "{{< relref "#terraform-plan" >}}"
    click apply "{{< relref "#terraform-apply" >}}"
```

The workflow itself can run in three scenarios on a `push` to `main`, `pull_request` on `main` and `workflow_dispatch`. The latter allows you to run the workflow on demand.

```yaml
name: "Terraform"

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
  workflow_dispatch:
```

### Common Setup

Both the [Terraform Plan]({{< relref "#terraform-plan" >}}) and [Terraform Apply]({{< relref "#terraform-apply" >}}) jobs run the same steps to begin with.

```mermaid
graph TD;
    checkout[Checkout] --> setup
    setup[Setup Terraform] --> config_cache
    config_cache[Configure Terraform Provider Plugin Cache] --> cache
    cache[Cache Terraform]

    click checkout "{{< relref "#checkout" >}}"
    click setup "{{< relref "#setup-terraform" >}}"
    click config_cache "{{< relref "#configure-terraform-provider-plugin-cache" >}}"
    click cache "{{< relref "#cache-terraform" >}}"
```

#### Checkout

Checks out code the code.

```yaml
- name: Checkout
  uses: actions/checkout@v3
```

#### Setup Terraform

Installs the terraform binary and the GitHub Action wrapper a script that exports the outputs.

```yaml
- name: Setup Terraform
  uses: hashicorp/setup-terraform@v2
```

#### Configure Terraform Provider Plugin Cache

Sets `plugin_cache_dir` setting in the `.terraformrc` and creates the directory to ensure its there. The setting of `.terraformrc` will mean during init plugins will be downloaded referenced from that directory.

```yaml
- name: Configure Terraform Provider Plugin Cache
  run: |
    plugin_cache_dir="$HOME/.terraform.d/plugin-cache"
    printf 'plugin_cache_dir="%s"' "${plugin_cache_dir}" > ~/.terraformrc
    mkdir --parents "${plugin_cache_dir}"
```

#### Cache Terraform

Restores the cache if available, the cache is keyed on the lock file hash. But the `restore-keys` will mean other caches will be used even if the lock file has changed. Having this step will mean a _Post_ step will be added to uploaded changes to the cache.

```yaml
  - name: Cache Terraform
    uses: actions/cache@v3
    with:
      path: |
        ~/.terraform.d/plugin-cache
      key: ${{ runner.os }}-terraform-${{ hashFiles('**/.terraform.lock.hcl') }}
      restore-keys: |
        ${{ runner.os }}-terraform-
```

### Terraform Plan

The Terraform Plan job is responsible for creating the plan, updating the summary, and updating the PR or uploading the plan.

```mermaid
graph TD;
    setup[Setup] --> format
    format[Terraform Format] --> init
    init[Terraform Init] --> plan
    plan[Terraform Plan] --> summary
    summary[Update Git Step Summary] --> ispr
    ispr{Is PR?} --> |Yes| comment
    comment[Update Pull Request]
    ispr{Is PR?} --> |No| upload
    upload[Encrypt & Upload Plan]

    click setup "{{< relref "#common-setup" >}}"
    click format "{{< relref "#terraform-format" >}}"
    click init "{{< relref "#terraform-init" >}}"
    click plan "{{< relref "#terraform-plan-1" >}}"
    click summary "{{< relref "#update-git-step-summary" >}}"
    click comment "{{< relref "#update-pull-request" >}}"
    click upload "{{< relref "#encrypt--upload-plan" >}}"
```

The job itself requires permissions to update the pull requests.

```yaml
jobs:
  terraform-plan:
    name: "Terraform Plan"
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
```

#### Terraform Format

Checks the formatting of the terraform code recursively.

```yaml
- name: Terraform Format
  id: fmt
  run: terraform fmt -no-color -check -diff -recursive
```

#### Terraform Init

Initialises the terraform workspace setting the lockfile to readonly so only expected versions of the plugins are allowed. Due to cache settings if this is not the first run of the repo plugins will be used from the cache.

```yaml
- name: Terraform Init
  id: init
  run: terraform init -no-color -lockfile=readonly
```

#### Terraform Validate

Validate the configuration is correct without making any connection to remote services or provider APIs.

```yaml
- name: Terraform Validate
  id: validate
  run: terraform validate -no-color
```

#### Terraform Plan

Plan the terraform changes against the current state, also output the plan to file `tfplan`

```yaml
- name: Terraform Plan
  id: plan
  run: terraform plan -no-color -input=false -out=tfplan
```

#### Update Git Step Summary

Update the Step Summary by sending markdown to `$GITHUB_STEP_SUMMARY` using the outputs of the terraform tasks and stdout from the plan.

Also reuses the Step Summary as a step output with a key `markdown` using `$GITHUB_OUTPUT`.

The `if success() || failure()` is important, because if any of the earlier steps fail the summary is updated with which ones failed, and which ones were skipped.

```yaml
- name: Update Git Step Summary
  id: report
  if: success() || failure()
  env:
    fmt_outcome: ${{ steps.fmt.outcome }}
    init_outcome: ${{ steps.init.outcome }}
    validate_outcome: ${{ steps.validate.outcome }}
    plan_outcome: ${{ steps.plan.outcome }}
    plan_stdout: ${{ steps.plan.outputs.stdout }}
  run: |
    {
      printf "#### Terraform Format and Style :pencil2: \`%s\`\n\n" "${fmt_outcome}";
      printf "#### Terraform Initialization :gear: \`%s\`\n\n" "${init_outcome}";
      printf "#### Terraform Validation :clipboard: \`%s\`\n\n" "${validate_outcome}";
      printf "#### Terraform Plan :book: \`%s\`\n\n" "${plan_outcome}";
      printf "<details><summary>Show Plan</summary>\n\n";
      printf "\`\`\`\terraformn%s\n\`\`\`\n\n" "${plan_stdout}";
      printf "</details>\n\n";
    } > "$GITHUB_STEP_SUMMARY"
    eof=$(head -c15 /dev/urandom | base64)
    {
      printf "markdown<<%s\n" "${eof}";
      cat "$GITHUB_STEP_SUMMARY";
      printf "%s\n" "${eof}";
    } >> "$GITHUB_OUTPUT"
```

![plan summary](/blog/images/posts/fixing-automating-terraform-with-github-actions-plan-summary.png "Example Plan Summary")

#### Update Pull Request

Code adapted from the [`README.md`](https://github.com/hashicorp/setup-terraform#usage) of `hashicorp/setup-terraform` which will create or update existing comment using the output from the previous step.

The `if` again uses `success() || failure()` which is important, because if any of the earlier steps fail the summary is updated with which ones failed, and which ones were skipped. It also has the added condition to only run on a `pull_request` event.

```yaml
- name: Update Pull Request
  uses: actions/github-script@v6
  if: (success() || failure()) && github.event_name == 'pull_request'
  env:
    markdown: ${{ steps.report.outputs.markdown }}
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    script: |
      // 1. Retrieve existing bot comments for the PR
      const { data: comments } = await github.rest.issues.listComments({
        owner: context.repo.owner,
        repo: context.repo.repo,
        issue_number: context.issue.number,
      })
      const botComment = comments.find(comment => {
        return comment.user.type === 'Bot' && comment.body.includes('Terraform Format and Style')
      })
      // 2. Prepare format of the comment
      const output = `${process.env.markdown}`;
      // 3. If we have a comment, update it, otherwise create a new one
      if (botComment) {
        github.rest.issues.updateComment({
          owner: context.repo.owner,
          repo: context.repo.repo,
          comment_id: botComment.id,
          body: output
        })
      } else {
        github.rest.issues.createComment({
          issue_number: context.issue.number,
          owner: context.repo.owner,
          repo: context.repo.repo,
          body: output
        })
      }
```

#### Encrypt & Upload Plan

The plan is the encrypted and uploaded as artefact, the plan contains sensitive information which is why encryption is important.

The encryption uses a passphrase that is secret set as an action repository secret.

The `if` ensures that the job is only executed on the default branch on either `push` or `workflow_dispatch` event.

```yaml
- name: Encrypt plan
  if:  github.ref == format('refs/heads/{0}', github.event.repository.default_branch) && contains(fromJSON('["push", "workflow_dispatch"]'), github.event_name)
  run: gpg --quiet --symmetric --cipher-algo AES256 --batch --yes --passphrase '${{ secrets.TF_PLAN_PASSPHRASE }}' --output tfplan.gpg tfplan

- name: Upload plan
  if:  github.ref == format('refs/heads/{0}', github.event.repository.default_branch) && contains(fromJSON('["push", "workflow_dispatch"]'), github.event_name)
  uses: actions/upload-artifact@v3
  with:
    name: tfplan
    path: tfplan.gpg
    retention-days: 1
```

### Terraform Apply

The Terraform Apply job is responsible for downloading the plan, applying it, and updating the summary.

```mermaid
graph TD;
    setup[Setup] --> download
    download[Download & Decrypt Plan] --> init
    init[Terraform Init] --> apply
    apply[Terraform Apply] --> summary
    summary[Update Git Step Summary]

    click setup "{{< relref "#common-setup" >}}"
    click download "{{< relref "#download--decrypt-plan" >}}"
    click init "{{< relref "#terraform-init-1" >}}"
    click apply "{{< relref "#terraform-apply-1" >}}"
    click summary "{{< relref "#update-git-step-summary-1" >}}"
```

The job has a few important settings:

- An `environment`, the environment is setup in Settings with protection rules and _Required reviewers_ to enable for interactive approval.
- Has a dependency on the `terraform-plan` using `needs`
- The `if` ensures that the job is only executed on the default branch on either `push` or `workflow_dispatch` event
- The `concurrency: cancel-in-progress: true` ensures that only the latest version of the action can run.

```yaml
jobs:
  //...
  terraform-apply:
    name: "Terraform Apply"
    runs-on: ubuntu-latest
    environment: default
    needs: terraform-plan
    if:  github.ref == format('refs/heads/{0}', github.event.repository.default_branch) && contains(fromJSON('["push", "workflow_dispatch"]'), github.event_name)
    concurrency:
      group: ${{ github.ref }}
      cancel-in-progress: true
```

#### Download & Decrypt Plan

The plan is downloaded and decrypted using the same passphrase from the action repository secret.

```yaml
- name: Download Plan
  uses: actions/download-artifact@v3
  with:
    name: tfplan

- name: Decrypt plan
  run: gpg --quiet --batch --yes --decrypt --passphrase '${{ secrets.TF_PLAN_PASSPHRASE }}' --output tfplan tfplan.gpg
```

#### Terraform Init

Initialises the terraform workspace setting the lockfile to readonly so only expected versions of the plugins are allowed. Due to cache and this always running after the plan job, it is expected that plugins will all be used from the cache.

```yaml
- name: Terraform Init
  id: init
  run: terraform init -no-color -lockfile=readonly
```

#### Terraform Apply

Apply the changes using the approved plan.

```yaml
- name: Terraform Apply
  id: apply
  run: terraform apply -no-color -input=false tfplan
```

#### Update Git Step Summary

Update the Step Summary by sending markdown to `$GITHUB_STEP_SUMMARY` using the outputs of the terraform tasks and stdout from the apply.

Again the `if success() || failure()` is important, because if any of the earlier steps fail the summary is updated with which ones failed, and which ones were skipped.

```yaml
- name: Update Git Step Summary
  id: report
  if: success() || failure()
  env:
    init_outcome: ${{ steps.init.outcome }}
    apply_outcome: ${{ steps.apply.outcome }}
    apply_stdout: ${{ steps.apply.outputs.stdout }}
  run: |
    {
      printf "#### Terraform Initialization :gear: \`%s\`\n\n" "${init_outcome}";
      printf "#### Terraform Apply :rocket: \`%s\`\n\n" "${apply_outcome}";
      printf "<details><summary>Show Outcome</summary>\n\n";
      printf "\`\`\`terraform\n%s\n\`\`\`\n\n" "${apply_stdout}";
      printf "</details>\n\n";
    } > "$GITHUB_STEP_SUMMARY"
```

![apply summary](/blog/images/posts/fixing-automating-terraform-with-github-actions-apply-summary.png "Example Apply Summary")

## Conclusion

For me this solves a lot of issues I had with action tutorial that hashicorp provided.

A working example can be found at [`mcwarman/terraform-workflow-test`](https://github.com/mcwarman/terraform-workflow-test).
