
# TypeScript Lambda Workflows

Shared GitHub workflows for TypeScript Lambda function applications. There are two workflows in this repository:

* [Build Release Candidate](#Build-Release-Candidate-Workflow)
* [Deploy Application](#Deploy-Applicaiton-Workflow)

## Build Release Candidate Workflow

This workflow builds a release candidate for an application using the following steps:

1. **Generate tag** - Generate a unique tag for the release candidate in the form `{semver}-{build_number}-{short_sha}`, such as `1.2.0-123-abcd123`.
    * `semver` is the application's sematic version that is extracted from the `version` property in the application's `package.json` file.
    * `build_number` is the build number for the GitHub workflow.
    * `short_sha` is the first seven characters of the commit SHA.
1. **Draft a release** - Create a [GitHub release](https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases) in draft status using the generated tag.
1. **Optimize function code** - For each Lambda function in the application, bundle the function code and its dependencies into a single file using [esbuild](https://esbuild.github.io/). This process only includes external dependencies that are statically referenced in the Lambda function's code. This optimizes the size of the function's code and reduces cold start times.
1. **Attach Lambda function archives to release** - Take the bundled code, archive this into a zip file, and attach that file to the draft GitHub release.
1. **Publish release** - Update the status of the release from draft to published. This event can be used to trigger downstream workflows, such as automatically deploying this new release candidate to the Dev environment.

> **Note**
> This workflow assumes that the `package.json` file is at the root of the repository and that every Lambda function is in a directory under `./src/functions`.

To use this workflow, add the following to your workflow file:

```yaml
jobs:
  build-release-candidate:
    name: Build release candidate
    uses: invitation-homes/typescript-lambda-workflows/.github/workflows/build-release-candidate.yml@v1
    secrets: inherit
```

This workflow accepts two optional input variables. The first is `esbuild-external`, which is passed to the esbuild process as the value for the `external` CLI option. See the [esbuild documentation](https://esbuild.github.io/api/#external) for more details.

```yaml
jobs:
  build-release-candidate:
    name: Build release candidate
    uses: invitation-homes/typescript-lambda-workflows/.github/workflows/build-release-candidate.yml@v1
    with:
      esbuild-external: pg-native
```

The second input variable is `use-source-map`. The default value is `false`. When this is set to `true`, the Lambda function's code will be bundled with source maps. To take advantage of this, Lambda functions must also have the environment variable `NODE_OPTIONS=--enable-source-maps`. See the [esbuild documentation](https://esbuild.github.io/api/#sourcemap) and [this blog post](https://serverless.pub/aws-lambda-node-sourcemaps/) for more details.

```yaml
jobs:
  build-release-candidate:
    name: Build release candidate
    uses: invitation-homes/typescript-lambda-workflows/.github/workflows/build-release-candidate.yml@v1
    with:
      use-source-map: true
```

## Deploy Application Workflow

This workflow deploys a GitHub release built by the previous workflow to AWS using the following steps:

1. **Download release artifacts** - Download the release and its assets for the given tag.
1. **Upload artifacts to S3** - Upload each zip file asset to the corresponding Lambda function S3 bucket.
1. **Apply Terraform changes** - Apply the Terraform changes for the application.
1. **Post deployment details** - When the deployment successfully completes, post the details of the deployment to our CI/CD API so that the deployment will be reflected in our [CI/CD Dashboard])(https://ci-cd-dashboard-prod.herokuapp.com/).

> **Note**
> This workflow assumes that every Lambda function is in a directory under `./src/functions` and the Terraform configuration for the application is under `./terraform/application`.

To use this workflow, add the following to your workflow file:

```yaml
jobs:
  deploy-application:
    name: Deploy applications
    uses: invitation-homes/typescript-lambda-workflows/.github/workflows/deploy-application.yml@v1
    secrets: inherit
    with:
      environment: ${{ THE DEPLOYMENT ENVIRONMENT GOES HERE }}
      tag: ${{ THE TAG TO BE DEPLOYED GOES HERE }}
```

This workflow requires two input variables:
* `environment` - The environment the application should be deployed to. This must be `dev`, `qa`, `uat`, or `prod`.
* `tag` - The tag corresponds to the GitHub release that is to be deployed.