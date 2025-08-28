# ironbank_release_action
GitHub Action for automating releases on to [Iron Bank](https://p1.dso.mil/ironbank)

The purpose of this action is to automate the routine changes required in an Iron Bank repository when you upgrade your application, which primarily involves going in and changing the version numbers of your application within the Dockerfile and hardening_manifest.yaml files.  Using this action lets you easily scale up a previously manual process to one that can handle all of your Iron Bank repositories for your applications and their various release types (multiple supported versions, different architectures, release vs latest, etc).

The manner in which this action works is to take your specific sequence of update steps which you supply in the `update_commands` parameter, and wrap them with all of the git and Iron Bank specific actions to get those changes pushed over to Iron Bank.  Conceptually, this action can be thought of as a "decorator" for the "function" consisting of your update commands where all you need to supply are the file manipulations needed to update the version strings in your Iron Bank repo and the action handles the rest.

## Pre- and postrequisites

In the general case, you will likely be building and pushing up a Docker container to a public registry which contains the latest version of your application.  You will then be referencing that build artifact in your hardening_manifest so that the Iron Bank pipelines know what to pull down on your behalf before you can reference that artifact over there.  Consequently, we recommend following a similar approach to the example below where you build and push the container and then grab the SHA that is generated, which you can then feed into your update commands.

Another consideration will be ensuring that your runner has all of the certificates and software it needs for the action to work properly.  Please ensure that your runner has the CA certificates required to communicate with https://repo1.dso.mil - usually the base set of certificates on the GitHub provided runners are fine, but occasionally Iron Bank upgrades their certificate which requires you to manually add it to your runner until such time that that certificate is propagated.  Please ensure that your runner has the following software installed on top of whatever you will need to do your update commands (ex. our example below uses `yq`):
  * `bash`
  * `curl`
  * `echo`
  * `eval`
  * `git`
  * `jq`
  * `mkdir`
  * `sed`

Once this action has run and attempted to create an issue and merge request on your behalf, it is your responsibility to confirm that it succeeded, to ensure that the resultant Iron Bank pipeline run went through properly, to address any comments or feedback provided by the Iron Bank engineers and reviewers, and to provide any justifications on the generated VAT as necessary.

## Security Concerns

Since this action allows you to supply arbitrary commands and also requires exposing a Personal Access Token (PAT) for a repository, you need to take appropriate action to reduce risk.  At minimum, we recommend 1) ensuring that only thoroughly vetted and tested code written by trusted authors is allowed to be placed in the `update_commands` input and 2) both the Iron Bank PAT and the GITHUB_TOKEN are scoped to the least required set of privileges. 

## Input and Output Arguments
### Input
#### `name` (Required)

The name of the application to be updated.  This string will be used in git branch names so ensure that the provided string is compatible with git's reference naming requirements such as by making sure there are no spaces or periods: https://git-scm.com/docs/git-check-ref-format.

No default value provided.

Example: `MyApplication`

#### `version` (Required)

The semver version number of the application to be released.  This string will be normalized by converting periods to hyphens for use in git branch names but otherwise the user must ensure that the provided string is compatible with git's reference naming requirements.

No default value provided.

Example: `1.2.3`

#### `working_directory_path` (Required)

The path to where the Iron Bank repo should be cloned.

Default value: `../tmp_ironbank_workspace`

Example: `../my_application_ironbank`

#### `ironbank_pat` (Required)

A GitLab personal access token (PAT) for the specified user who has access to the Iron Bank project - ensure that this is a GitHub repo or org secret so that it is not leaked in the logs: https://docs.gitlab.com/user/profile/personal_access_tokens/#create-a-personal-access-token https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions.

No default value provided.

Example: `preface-random1234characters`

#### `ironbank_username` (Required)

The GitLab username for the user who has access to the Iron Bank project and whose PAT is being supplied.

No default value provided.

Example: `alastname`

#### `ironbank_project_id` (Required)

The GitLab project id for your Iron Bank project: https://docs.gitlab.com/user/project/working_with_projects/#access-a-project-by-using-the-project-id.

No default value provided.

Example: `12345`

#### `ironbank_project_clone_url` (Required)

The clone url for your Iron Bank project without the 'https://' in front.

No default value provided.

Example: `repo1.dso.mil/dsop/organization/suborganization/project.git`

#### `git_commit_author_name` (Required)

The commit author's name for the update being pushed to Iron Bank.

Default value: `Automated Release`

Example: `Automated Application Release`

#### `git_commit_author_email` (Required)

The commit author's email for the update being pushed to Iron Bank.

No default value provided.

Example: `foobar@example.com`

#### `issue_labels` (Required)

A comma separated list of labels that will be applied to the issue filed on Iron Bank.

Default value: `Status::Review`

Example: `Status::Review`

#### `target_branch` (Not required)

The branch that the merge request will be targetting; if no or an empty input is provided, then the target branch will be the repository's default branch.  `development` is usually the default branch on Iron Bank.

No default value provided.

Example: `development`

#### `draft` (Not required)

Whether or not to mark the merge request as a draft; if "true", then the merge request will be marked as a draft.

No default value provided.

Example: `false`

#### `update_commands` (Required)

The bash script that will be evaluated in order to update all appropriate values within the Iron Bank repo; WARNING: only allow trusted content to be placed here, see [Security Concerns](#security-concerns).

No default value provided.

Example: `date >> CHANGELOG`

### Output

#### `issue_id`

Id for the generated GitLab issue on Iron Bank.

#### `mr_id`

Id for the generated GitLab merge request on Iron Bank.

## Secrets

At minimum, the following variable should be treated as a secret and not used in plain text within your workflow(s):

  * `ironbank_pat`

## Example

Below is an example action.

```
name: Push Heimdall Server to Docker Hub on every release, tag as release-latest and version, and then upgrade the tag on Iron Bank

on:
  release:
    types: [published]
  workflow_dispatch:
    inputs:
      version:
        description: 'Version'     
        required: true

permissions:
  contents: read

jobs:
  docker:
    runs-on: ubuntu-22.04
    steps:
      - name: Run string replace # remove the v from the version number before using it in the docker tag
        uses: frabert/replace-string-action@v2
        id: format-tag
        with:
          pattern: 'v'
          string: '${{ github.event.release.tag_name || github.event.inputs.version}}'
          replace-with: ''
          flags: 'g'
      - name: Checkout the Heimdall Repository
        uses: actions/checkout@v4
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      - name: Login to DockerHub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          platforms: linux/amd64
          tags: mitre/heimdall2:release-latest,mitre/heimdall2:${{ steps.format-tag.outputs.replaced }}
      - name: Get Docker SHA
        shell: bash
        run: echo "DOCKER_SHA=$(docker pull mitre/heimdall2:${{ steps.format-tag.outputs.replaced }} > /dev/null 2>&1 && docker inspect --format='{{index .RepoDigests 0}}' mitre/heimdall2:${{ steps.format-tag.outputs.replaced }} | cut -d '@' -f 2)" >> $GITHUB_ENV
      - name: Upgrade tag on Iron Bank
        uses: mitre/ironbank_release_action@v1
        with:
          name: "Heimdall"
          version: ${{ steps.format-tag.outputs.replaced }}
          ironbank_pat: ${{ secrets.IRONBANK_PAT }}
          ironbank_username: ${{ secrets.IRONBANK_USERNAME }}
          ironbank_project_id: 5450
          ironbank_project_clone_url: repo1.dso.mil/dsop/mitre/security-automation-framework/heimdall2.git
          git_commit_author_name: "Automated Heimdall Release"
          git_commit_author_email: "saf@mitre.org"
          update_commands: |
            yq e -i '.args.HEIMDALL_VERSION=${{ steps.format-tag.outputs.replaced }} | .tags[0]=${{ steps.format-tag.outputs.replaced }} | .labels."org.opencontainers.image.version"=${{ steps.format-tag.outputs.replaced }} | .resources[1].tag=mitre/heimdall2:${{ steps.format-tag.outputs.replaced }} | .resources[1].url=docker://docker.io/mitre/heimdall2@${{ env.DOCKER_SHA }}' hardening_manifest.yaml
            sed -i s/HEIMDALL_VERSION=\.\*/HEIMDALL_VERSION=${{ steps.format-tag.outputs.replaced }}/ Dockerfile
```

For more examples, check out these workflows: [SAF CLI release]() and [SAF CLI mainline]().

## Contributing, Issues and Support

### Contributing

Please feel free to look through our issues, make a fork, and submit PRs and improvements. We love hearing from our end-users and the community and will be happy to engage with you on suggestions, updates, fixes or new capabilities.

### Issues and Support

Please feel free to contact us by [opening an issue](https://github.com/mitre/ironbank_release_action/issues/new) or emailing us at [saf@mitre.org](mailto:saf@mitre.org) should you have any suggestions, questions, or issues.  Please direct security concerns to [saf-security@mitre.org](mailto:saf-security@mitre.org).
