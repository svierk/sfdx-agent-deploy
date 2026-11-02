# 🤖 SFDX Agent Deploy

This repository implements a GitHub composite action that brings the **Agentforce agents** of a Salesforce project to any target org. It publishes and activates agents, validates their Agent Script on a dry-run, and deploys their tests once the agents exist.

A project without agents or agent tests skips everything, so the action is safe to add to any pipeline.

## Why a plain deployment is not enough

Getting an agent into an org takes more than `sf project deploy start`:

- An authoring bundle has to be **published** to become an agent, and the new version has to be **activated**.
- Publishing does not deploy the Apex classes and flows behind the agent's actions, so they have to arrive first.
- A test definition fails the whole deployment (`BotVersion not found`) while its agent has no version in the org, so tests have to arrive last.
- Every publish creates a new version, even without a change.
- Deploying an active version fails (`Can't edit an active bot version`), even if it is unchanged.

So the action runs twice, around [SFDX Deploy](https://github.com/svierk/sfdx-deploy).

## Usage

### Deploy agents to an org

```yaml
# Least-privilege token: this pipeline only reads the repository.
permissions:
  contents: read

jobs:
  deployment:
    name: Deployment
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v7.0.1
        with:
          fetch-depth: 0
          persist-credentials: false

      - name: Install SF CLI
        uses: svierk/sfdx-cli-setup@v1.2.0

      - name: Salesforce Org Login
        uses: svierk/sfdx-login@v1.4.3
        with:
          client-id: ${{ secrets.SFDX_CONSUMER_KEY }}
          jwt-secret-key: ${{ secrets.SFDX_JWT_SECRET_KEY }}
          username: ${{ secrets.SFDX_USERNAME }}
          alias: awesome-org

      - name: Prepare Agents
        id: agents
        uses: svierk/sfdx-agent-deploy@v1.0.0
        with:
          phase: prepare
          source-dir: force-app
          target-org: awesome-org

      - name: Deploy Metadata
        uses: svierk/sfdx-deploy@v1.4.0
        with:
          source-dir: force-app
          exclude: ${{ steps.agents.outputs.exclude }}
          target-org: awesome-org

      - name: Deploy Agents
        uses: svierk/sfdx-agent-deploy@v1.0.0
        with:
          source-dir: force-app
          target-org: awesome-org
```

The following actions were also used in the example workflow to create the prerequisites:

- [SFDX CLI Setup](https://github.com/svierk/sfdx-cli-setup) | Installs the Salesforce CLI and related plugins
- [SFDX Login](https://github.com/svierk/sfdx-login) | Handles Salesforce login via SFDX Auth URL or the JWT flow
- [SFDX Deploy](https://github.com/svierk/sfdx-deploy) | Deploys Salesforce metadata to a target org

### Validate agents on a pull request

```yaml
- name: Prepare Agents
  id: agents
  uses: svierk/sfdx-agent-deploy@v1.0.0
  with:
    phase: prepare
    target-org: ci

- name: Validate Deployment
  uses: svierk/sfdx-deploy@v1.4.0
  with:
    source-dir: force-app
    exclude: ${{ steps.agents.outputs.exclude }}
    target-org: ci
    dry-run: true

- name: Validate Agents
  uses: svierk/sfdx-agent-deploy@v1.0.0
  with:
    target-org: ci
    dry-run: true
```

On a dry-run, the action only compiles the Agent Script of the draft bundles. Nothing is saved or activated.

### Run the agent tests

Agent tests are a test type of [SFDX Run Tests](https://github.com/svierk/sfdx-run-tests), which judges the result of each test case:

```yaml
- name: Run Agent Tests
  uses: svierk/sfdx-run-tests@v1.4.0
  with:
    apex: false
    agent: true
    target-org: awesome-org
```

## What the action does

| In your source                                                            | `prepare` leaves out of the deployment  | `publish` then                                                                                  |
| ------------------------------------------------------------------------- | --------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **Draft bundle** - an `AiAuthoringBundle` without `<target>`              | Compiled files of the same agent        | Publishes it if its script differs from the latest version in the org, and activates that version |
| **Compiled agent** - `AiAgentDefinition` or `Bot` files, no draft bundle  | The version that is active in the org   | Activates the highest version in your source                                                    |
| **Agent tests** - `AiEvaluationDefinition`, `AiTestingDefinition`         | All of them                             | Deploys them when all agents succeeded                                                          |

An agent built with Agent Script can only be deployed as compiled metadata together with its bundle, including a `<target>`. A publish expects the bundle in the default package directory of `sfdx-project.json`.

The action uses `--skip-retrieve` when publishing, as CI would only download the compiled metadata into a workspace that is thrown away.

## Inputs

| Name               | Required | Default     | Description                                                                                                                                                                                                                              |
| ------------------ | -------- | ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `phase`            | no       | `publish`   | `prepare` runs **before** SFDX Deploy and outputs what the deployment has to leave out. `publish` runs **after** it.                                                                                                                       |
| `source-dir`       | no       | `force-app` | Source directories to look for agents and agent tests in, comma-separated or multi-line.                                                                                                                                                   |
| `target-org`       | no       |             | Username or alias of the target org. Not required if the default org is set.                                                                                                                                                               |
| `dry-run`          | no       | `false`     | Only compile the Agent Script of the draft bundles. Nothing is saved, activated or deployed.                                                                                                                                               |
| `activate`         | no       | `true`      | Activate the agent versions. Set to `false` to leave the currently active versions in place.                                                                                                                                               |
| `deploy-tests`     | no       | `true`      | Deploy the agent test definitions once the agents exist.                                                                                                                                                                                   |
| `post-publish-dir` | no       |             | Further source directories to deploy once the agents exist, comma-separated or multi-line. For metadata referencing an agent, like a permission set granting access to it. Keep them outside of the source deployed before the publish.    |
| `wait`             | no       | `33`        | Number of minutes to wait for the deployment of the agent tests and post-publish directories.                                                                                                                                              |
| `api-version`      | no       |             | Override the api version used for api requests, e.g. `68.0`.                                                                                                                                                                               |
| `step-summary`     | no       | `true`      | Write a result section to the GitHub Actions [job summary](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#adding-a-job-summary). Set to `false` to avoid collisions with a custom workflow summary. |

## Outputs

| Name        | Description                                                                                                  |
| ----------- | ------------------------------------------------------------------------------------------------------------ |
| `exclude`   | Comma-separated patterns for the `exclude` input of SFDX Deploy (phase `prepare`), empty if nothing to leave out. |
| `status`    | `Skipped` (no agents or agent tests), `Prepared`, `Validated`, `Deployed` or `Failed`.                         |
| `published` | Comma-separated API names of the agents a new version was published for.                                     |
| `unchanged` | Comma-separated API names of the agents whose script matches the latest version in the org.                  |
| `activated` | Comma-separated `name vN` entries of the versions that are active after this run.                            |

## Prerequisites

- **Connected app:** with the JWT flow, the connected app needs the OAuth scopes `chatbot_api`, `sfap_api` and `web`, next to `api` and `refresh_token`. The CLI uses them to reach the agent APIs.
- **Target org:** Einstein and Agentforce enabled, and an integration user with the permissions to publish agents.
- **API version:** `"sourceApiVersion": "68.0"` in `sfdx-project.json`. The `AiAgentDefinition` types need it.
- **Values per org:** a service agent names its agent user in the script. Use a `replaceWithEnv` [string replacement](https://developer.salesforce.com/docs/ai/agentforce/guide/agent-dx-deploy-metadata.html) in `sfdx-project.json` and set the variable on the steps of both this action and SFDX Deploy. The comparison with the org applies these replacements too; a `replaceWithFile` replacement can't be compared, so that script is published on every run.

## References

- [Retrieve and Deploy Agent Metadata](https://developer.salesforce.com/docs/ai/agentforce/guide/agent-dx-deploy-metadata.html)
- [The New Agentforce Metadata and Development Lifecycle](https://developer.salesforce.com/blogs/2026/05/new-agentforce-metadata-and-development-lifecycle)
- [Salesforce DevOps Starter Kit](https://github.com/svierk/salesforce-devops-starter-kit) | The [Agentforce guide](https://svierk.github.io/salesforce-devops-starter-kit/agentforce) covers the whole setup

## License

This project is licensed under the [MIT License](LICENSE).
