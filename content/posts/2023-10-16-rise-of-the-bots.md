---
title: Rise of the Bots
author: Matthew
date: 2023-10-16T10:06:23+01:00
---

As an engineer I attempt to automate as much as possible. This is normally done locally first, and then naturally migrates to a pipeline.

When a pipeline makes changes to the repo the challenge is giving it the required access.

Some examples of pipelines that require access to GitHub are:

- Commenting on PR based on the build output
- Labeling a PR based on the build output
- Keeping documentation up to date with [`terraform-docs`](https://terraform-docs.io/)
- Keeping versions up to date with [`updatecli`](https://www.updatecli.io/) (for when dependabot doesn't cut it)
- Automatically Backporting PRs based on labels using [`tibdex/backport`](https://github.com/tibdex/backport)

Pipeline access can be solved in three ways:

- Using GitHub Actions in built token generation
- Using GitHub App to create a token
- Using PAT

Using PATs was taken off the table after a discussion internally. This is for a couple of reasons:

If the PAT is created as a user all the operations will be done as the user which is bad practice in a pipeline.

If the PAT is created using a bot user this comes at the cost of a seat to the organization, and requires management of the user and token.

So I set about evaluating the other options.

## Using GitHub Actions in built token generation

This is a good option as it uses built in features of GitHub workflows but events triggered by these tokens will not create new workflow runs, this is a very deliberate limitation which prevents users from accidentally creating recursive workflow runs.

This makes it a good candidate for:

- Commenting on PR based on the build output
- Labeling on PR based on the build output

Neither of which trigger workflow runs. But this means it doesn't work for the others that create a commit or PR if there is a required status check (build, linting, etc).

Example below:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
        //...
      - name: Pull Request Comment
        uses: actions/github-script@v6
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            #### Terraform Plan \`${{ steps.plan.outcome }}\`

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })
```

## Using GitHub App to create a token

This is an excellent alternative to the built in method when the events triggered by these tokens should create a new workflow run.

It comes at the cost of some set-up that requires admin access at the organization level but once this has been done it's available to the repos.

The app was set-up with limited API access to certain repos and organization secrets were scoped to the repos that require them.

Generating a token in the GitHub pipeline is done using `tibdex/github-app-token` providing the App ID and Private Key.

The step provides an output that can be used by other actions.

Under the covers the action:

1. Creates and Signs a JWT
2. Makes the required API calls to generate the token

This means it can also be done [outside of GitHub](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token-for-a-github-app), which I've used in Azure DevOps to access GitHub API, and on self hosted runners to manage its own registration.

An example of using it in `actions/checkout`:

```yaml
jobs:
  terraform-docs:
    name: Update Terraform Docs
    runs-on: ubuntu-latest
    steps:
      - name: Generate GitHub app token
        id: github-app-token
        uses: tibdex/github-app-token@v2
        with:
          app_id: ${{ secrets.GH_REPO_CONTENT_APP_ID }}
          private_key: ${{ secrets.GH_REPO_CONTENT_APP_PRIVATE_KEY }}
          installation_retrieval_mode: organization
          installation_retrieval_payload: ${{ github.repository_owner }}
          repositories: >-
            ["${{ github.event.repository.name }}"]
          permissions: >-
            {"contents": "write"}

      - name: Checkout
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.ref }}
          token: ${{ steps.github-app-token.outputs.token }}

      // ...

      - name: Commit and Push
        env:
          GH_USER_NAME: mcwarman[bot]
          GH_USER_EMAIL: 137313888+mcwarman[bot]@users.noreply.github.com
          FILE_PATTERN: README.md
          COMMIT_MESSAGE: |
            docs: update terraform docs
        run: |
          shopt -s globstar
          for file in **/"${FILE_PATTERN}"; do
            git add -- "$file"
          done
          if [ -n "$(git diff --staged)" ]; then
            git config user.name  "${GH_USER_NAME}"
            git config user.email "${GH_USER_EMAIL}"
            git commit -m "${COMMIT_MESSAGE}"
            git push
          fi
```

## Conclusion

Using a combination of both of these methods covers all of my GitHub API access needs in a pipeline.

We're even using it to manage our GitHub repositories settings via terraform executed in GitHub pipeline. But thats a blog for another day.

## References

- [Automatic token authentication](https://docs.github.com/en/actions/security-guides/automatic-token-authentication)
- [Triggering a workflow from a workflow](https://docs.github.com/en/actions/using-workflows/triggering-a-workflow#triggering-a-workflow-from-a-workflow)
