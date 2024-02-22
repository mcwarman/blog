---
title: Dynamic GitHub Templates
author: Matthew
date: 2024-02-22T11:01:37Z
---

Templates are useful way to bootstrap a repository, once I see a pattern in repositories, I will create a template version of the repository locally and use `rysnc` to copy files across to a newly created repository. This works when you're the only one producing the repositories, but that's almost never the case. I wanted to share my template with others.

I started by creating a [template repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-template-repository) in GitHub, this works great for a simple template that contains dotfiles and GitHub workflows. It doesn't retain history from template repository which is perfect (something GitLab used to do from memory, this might have changed though).

But I started to question whether we can make these dynamic. In order to try prove this out I started research the simplest of tasks "Could we update the title in the `README.md` of the new repository?"

You can loose yourself in conversations about platforms as a service (PaaS) and tooling that will support this dream. But I stumbled across a GitHub discussion with a helpful [comment](https://github.com/orgs/community/discussions/22183#discussioncomment-4585507), that suggested using a GitHub workflow which runs on the `create` event. This got the cogs turning. And now I'm doing exactly what a consultant for a PaaS provider once said, "You guys will end up building your own first, everyone always does".

The approach I decided on was a workflow that made the updates and creates a pull request with those updates, allowing the creator of that repo to review them. I could have made the commit straight from workflow to the default branch but I had some other plans which mean the PR made sense.

The `on.create` runs on the creation of new branches, in order to limit it to only running once for repository, we check the `github.run_number` is `1`, this could be extended further to only run on default branches. The `Update files` step replaces the a placeholder in the `README.md` and removes the bootstrap workflow.

I also added `on.workflow_dispatch`, this allowed me to test the workflow in the template repository itself deleting the branch and pull request after the testing was complete.

The simplest version of the workflow looked like this:

```yaml
name: Template Bootstrap

on:
  create:
  workflow_dispatch:

jobs:
  template:
    name: Template Bootstrap
    if: |
      (github.event_name == 'create' && github.run_number == 1) ||
      github.event_name == 'workflow_dispatch'
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull_request: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.repository.default_branch }}

      - name: Update files
        env:
          GITHUB_REPOSITORY_NAME: ${{ github.event.repository.name }}
        run: |
          sed -i "1s/repo_name/${GITHUB_REPOSITORY_NAME}/g" README.md
          rm .github/workflows/template-bootstrap.yml

      - name: Create Pull Request
        uses: peter-evans/create-pull-request@v6
        with:
          commit-message: "initial: customise template"
          branch: initial/template
          title: "initial: customise template"
          body: |
            Automated changes by [Workflow Run](https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}).
```

This worked perfectly and had the desired effect but the cogs continued to turn.

The template was for a terraform projects and I wanted to update the state backend. The problem is the state backend could point to different environments. So there was decision for the consumer of the template.

What I decided to do was use a review comments to make a suggestion allowing the consumer to choose which environment.

A review comment can only be made on a change so I needed to create a change to be able to make the suggestion. The starting point is a `main.tf`, with a commented out `backend`.

```hcl
terraform {
  # backend "azurerm" {
  # }
}
```

This had multiple benefits we had a [Terraform Workflow]({{< relref "2023-03-03-fixing-automating-terraform-with-github-actions" >}}) setup that would run on creation of the repository a backend pointing nowhere would mean the `terraform.tfstate` would get created but then forgotten about.

The `Update files` step was updated to uncomment the `backend` in `main.tf`, this meant that the _Terraform Workflow_ now fails because it doesn't know where to send the state. Which again is perfect because it means the reviewer needs to do something before accepting the changes.

```yaml
- name: Update files
  env:
    GITHUB_REPOSITORY_NAME: ${{ github.event.repository.name }}
  run: |
    sed -i "1s/repo_name/${GITHUB_REPOSITORY_NAME}/g" README.md
    sed -i "2,3s/^  # /  /g" main.tf
    rm .github/workflows/template-bootstrap.yml
```

In order to help the reviewer the workflow would suggest the available options for the environment. This was done using [`actions/github-script`](<https://github.com/actions/github-script>) and [`github.rest.pulls.createReviewComment`](<https://octokit.github.io/rest.js/v20#pulls-create-review-comment>). The `pull_request` and `pull_request_head_sha` both provided as outputs to from [`peter-evans/create-pull-request`](https://github.com/peter-evans/create-pull-request).

```yaml
- name: Comment terraform backend update
  uses: actions/github-script@v7
  env:
    pull_request: ${{ steps.pr.outputs.pull-request-number }}
    pull_request_head_sha: ${{ steps.pr.outputs.pull-request-head-sha }}
  with:
    github-token: ${{ steps.github-app-token.outputs.token }}
    script: |
      const { pull_request, pull_request_head_sha } = process.env;
      const output = `
      Review the suggestions below

      If this project is targeting \`Development\`, apply this:

      \`\`\`suggestion
        backend "azurerm" {
          tenant_id            = "00000000-0000-0000-0000-000000000000"
          subscription_id      = "00000000-0000-0000-0000-000000000000"
          resource_group_name  = "StorageAccount-ResourceGroup-Dev"
          storage_account_name = "terraformdev"
          container_name       = "tfstate"
          key                  = "${context.repo.repo}.tfstate"
        }
      \`\`\`

      If this project is targeting \`Production\`, apply this:

      \`\`\`suggestion
        backend "azurerm" {
          tenant_id            = "00000000-0000-0000-0000-000000000000"
          subscription_id      = "00000000-0000-0000-0000-000000000000"
          resource_group_name  = "StorageAccount-ResourceGroup-Prod"
          storage_account_name = "terraformprod"
          container_name       = "tfstate"
          key                  = "${context.repo.repo}.tfstate"
        }
      \`\`\`
      `
      github.rest.pulls.createReviewComment({
        owner: context.repo.owner,
        repo: context.repo.repo,
        pull_number: pull_request,
        body: output,
        commit_id: pull_request_head_sha,
        path: "main.tf",
        start_line: 2,
        line: 3
      })

```

The reviewer can then accept a suggestion which updates the backend, approve and merge. Then start writing the infrastructure code they intended.

The last part is very terraform specific but the concept applies to any changes that require a decision.
