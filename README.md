# AWS CDK Diff PR Commenter

A GitHub Action that posts the output of `cdk diff` as a comment on Pull Requests. This action helps teams review infrastructure changes directly within their PR workflow, making it easier to catch potential issues before deploying CDK changes.

## Features

- Automatically posts formatted CDK diff output to PR comments
- Updates existing comments instead of creating duplicates
- Skips posting when there are no changes
- Supports custom headers for better organization in multi-stack setups
- Parses and highlights IAM statement changes, Security Group changes, Parameters, and Resources
- This GitHub Action is developed using native JavaScript, so it executes way faster compared to an action build using Docker.


## Inputs

| Input       | Description                                                             | Required | Default               |
| ----------- | ----------------------------------------------------------------------- | -------- | --------------------- |
| `diff-file` | Path to the CDK diff output file to post as comment in the Pull Request | Yes      | -                     |
| `token`     | The GitHub or PAT token to use for posting comments to Pull Requests    | Yes      | `${{ github.token }}` |
| `header`    | Header to use for the Pull Request comment                              | Yes      | `📝 CDK Diff`          |

## Outputs

| Output     | Description                                                  |
| ---------- | ------------------------------------------------------------ |
| `markdown` | The raw markdown output of the `cdk diff` command            |
| `empty`    | Whether the `cdk diff` contains any changes (`true`/`false`) |

## Usage

### Example 1: Complete Workflow

```yaml
name: CDK Diff PR

on:
  pull_request:
    branches:
      - main

jobs:
  cdk-diff:
    name: CDK Diff PR branch with environment
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
      pull-requests: write
    env:
      AWS_REGION: us-east-1

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v5
        with:
          ref: ${{ github.event.pull_request.head.sha }}

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: npm

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: ${{ env.AWS_REGION }}

      - name: Install Dependencies
        run: npm ci

      - name: CDK Diff
        run: |
          npx cdk diff --all --no-color > cdk-diff.txt 2>&1 || true

      - name: Post CDK Diff Comment in PR
        uses: towardsthecloud/aws-cdk-diff-pr-commenter@v1
        with:
          diff-file: cdk-diff.txt
          header: |
            ## CDK Diff for production in ${{ env.AWS_REGION }}
```

### Example 2: Reusable Workflow

Create a reusable workflow in `.github/workflows/cdk-diff-reusable.yml`:

```yaml
name: CDK Diff Reusable

on:
  workflow_call:
    inputs:
      aws-region:
        description: 'AWS Region where resources will be deployed'
        type: string
        required: true
      environment:
        description: 'Environment name (e.g., production, staging)'
        type: string
        required: true
      role-to-assume:
        description: 'AWS IAM role ARN to assume'
        type: string
        required: true
      node-version:
        description: 'Node.js version to use'
        type: string
        default: '20'

jobs:
  cdk-diff:
    name: CDK Diff for ${{ inputs.environment }}
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
      pull-requests: write

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: npm

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ inputs.role-to-assume }}
          aws-region: ${{ inputs.aws-region }}

      - name: Install Dependencies
        run: npm ci

      - name: CDK Diff
        run: |
          npx cdk diff --all --no-color > cdk-diff.txt 2>&1 || true

      - name: Post CDK Diff Comment in PR
        uses: towardsthecloud/aws-cdk-diff-pr-commenter@v1
        with:
          diff-file: cdk-diff.txt
          header: |
            ## CDK Diff for ${{ inputs.environment }} in ${{ inputs.aws-region }}
```

Then call this workflow from your main workflow:

```yaml
name: CDK Diff PR

on:
  pull_request:
    branches:
      - main

jobs:
  diff-production:
    uses: ./.github/workflows/cdk-diff-reusable.yml
    with:
      aws-region: us-east-1
      environment: production
      role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole

  diff-staging:
    uses: ./.github/workflows/cdk-diff-reusable.yml
    with:
      aws-region: us-west-2
      environment: staging
      role-to-assume: arn:aws:iam::987654321098:role/GitHubActionsRole
```

## Permissions

This action requires the following permissions:

```yaml
permissions:
  pull-requests: write  # Required to post comments on PRs
  contents: read        # Required to read repository contents
```

## Notes

- The action will update the same comment on subsequent pushes to the PR, avoiding comment spam
- No comment will be posted if the diff shows no changes
- The `diff-file` should contain the output of `cdk diff` (typically redirected to a file)
- The action automatically parses and highlights:
  - IAM Statement Changes
  - Security Group Changes
  - Parameter Changes
  - Resource Changes (additions, updates, removals)
- Use `--no-color` flag with `cdk diff` to ensure clean output parsing
- The action gracefully handles multiple stacks in a single diff output

## Author

Maintained by [Towards the Cloud](https://github.com/towardsthecloud)
