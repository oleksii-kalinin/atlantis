# Custom Workflows

Custom workflows can be defined to override the default commands that Atlantis
runs.

## Usage

Custom workflows can be specified in the Server-Side Repo Config or in the Repo-Level
`atlantis.yaml` files.

**Notes:**

* If you want to allow repos to select their own workflows, they must have the
`allowed_overrides: [workflow]` setting. See [server-side repo config use cases](server-side-repo-config.md#allow-repos-to-choose-a-server-side-workflow) for more details.
* If in addition you also want to allow repos to define their own workflows, they must have the
`allow_custom_workflows: true` setting. See [server-side repo config use cases](server-side-repo-config.md#allow-repos-to-define-their-own-workflows) for more details.

## Use Cases

### .tfvars files

Given the structure:

```plain
.
└── project1
    ├── main.tf
    ├── production.tfvars
    └── staging.tfvars
```

If you wanted Atlantis to automatically run plan with `-var-file staging.tfvars` and `-var-file production.tfvars`
you could define two workflows:

```yaml
# repos.yaml or atlantis.yaml
workflows:
  staging:
    plan:
      steps:
      - init
      - plan:
          extra_args: ["-var-file", "staging.tfvars"]
    # NOTE: no need to define the apply stage because it will default
    # to the normal apply stage.

  production:
    plan:
      steps:
      - init
      - plan:
          extra_args: ["-var-file", "production.tfvars"]
    apply:
      steps:
        - apply:
            extra_args: ["-var-file", "production.tfvars"]
    import:
      steps:
        - init
        - import:
            extra_args: ["-var-file", "production.tfvars"]
    state_rm:
      steps:
        - init
        - state_rm:
            extra_args: ["-lock=false"]
```

Then in your repo-level `atlantis.yaml` file, you would reference the workflows:

```yaml
# atlantis.yaml
version: 3
projects:
# If two or more projects have the same dir and workspace, they must also have
# a 'name' key to differentiate them.
- name: project1-staging
  dir: project1
  workflow: staging
- name: project1-production
  dir: project1
  workflow: production

workflows:
  # If you didn't define the workflows in your server-side repos.yaml config,
  # you would define them here instead.
```

When you want to apply the plans, you can comment

```shell
atlantis apply -p project1-staging
```

and

```shell
atlantis apply -p project1-production
```

Where `-p` refers to the project name.

### Adding extra arguments to Terraform commands

If you need to append flags to `terraform plan` or `apply` temporarily, you can
append flags on a comment following `--`, for example commenting:

```shell
atlantis plan -- -lock=false
```

If you always need to do this for a project's `init`, `plan` or `apply` commands
then you must define a custom workflow and set the `extra_args` key for the
command you need to modify.

```yaml
# atlantis.yaml or repos.yaml
workflows:
  myworkflow:
    plan:
      steps:
      - init:
          extra_args: ["-lock=false"]
      - plan:
          extra_args: ["-lock=false"]
    apply:
      steps:
      - apply:
          extra_args: ["-lock=false"]
```

If [policy checking](policy-checking.md#how-it-works) is enabled, `extra_args` can also be used to change the default behaviour of conftest.

```yaml
workflows:
  myworkflow:
    policy_check:
      steps:
      - show
      - policy_check:
          extra_args: ["--all-namespaces"]
```

### Custom init/plan/apply Commands

If you want to customize `terraform init`, `plan` or `apply` in ways that
aren't supported by `extra_args`, you can completely override those commands.

In this example, we're not using any of the built-in commands and are instead
using our own.

```yaml
# atlantis.yaml or repos.yaml
workflows:
  myworkflow:
    plan:
      steps:
      # If you want to hide command output from Atlantis's PR comment, use
      # the output option on the run step's expanded form.
      - run:
          command: terraform init -input=false
          output: hide

      # If you're using workspaces you need to select the workspace using the
      # $WORKSPACE environment variable.
      - run: terraform workspace select $WORKSPACE

      # You MUST output the plan using -out $PLANFILE because Atlantis expects
      # plans to be in a specific location.
      - run: terraform plan -input=false -refresh -out $PLANFILE
    apply:
      steps:
      # Again, you must use the $PLANFILE environment variable.
      - run: terraform apply $PLANFILE
```

### CDKTF

Here are the requirements to enable [CDKTF](https://developer.hashicorp.com/terraform/cdktf)

* A custom image with `CDKTF` installed
* Add `**/cdk.tf.json` to the list of Atlantis autoplan files.
* Set the `atlantis-include-git-untracked-files` flag so that the Terraform files dynamically generated
by CDKTF will be add to the Atlantis modified file list.
* Use `pre_workflow_hooks` to run `cdktf synth`
* Optional: There isn't a requirement to use a repo `atlantis.yaml` but one can be leveraged if needed.

#### Custom Image

```dockerfile
# Dockerfile
FROM ghcr.io/runatlantis/atlantis:v0.19.7

USER root
RUN apk add npm && npm i -g cdktf-cli
```

#### Server Config

```bash
# env variables
ATLANTIS_AUTOPLAN_FILE_LIST="**/*.tf,**/*.tfvars,**/*.tfvars.json,**/cdk.tf.json"
ATLANTIS_INCLUDE_GIT_UNTRACKED_FILES=true
```

OR

`atlantis server --config config.yaml`

```yaml
# config.yaml
autoplan-file-list: "**/*.tf,**/*.tfvars,**/*.tfvars.json,**/cdk.tf.json"
include-git-untracked-files: true
```

#### Server Repo Config

Use `pre_workflow_hooks`

`atlantis server --repo-config="repos.yaml"`

```yaml
# repos.yaml
repos:
  - id: /.*cdktf.*/
    pre_workflow_hooks:
      - run: npm i && cdktf get && cdktf synth --output ci-cdktf.out
```

**Note:** don't use the default `cdktf.out` directory that CDKTF uses, as this should be in the `.gitignore` list of the
repo, so that locally generated files are not checked in.

#### Repo Structure

This is the git repo structure after running `cdktf synth`. The `cdk.tf.json` files contain the Terraform configuration
that atlantis can run.

```bash
$ tree --gitignore
.
├── cdktf.json
├── ci-cdktf.out
│   ├── manifest.json
│   └── stacks
│       └── eks
│           └── cdk.tf.json
```

#### Workflow

1. Container orchestrator (k8s/fargate/ecs/etc) uses the custom docker image of atlantis with `cdktf` installed with
the `--autoplan-file-list` to trigger on `cdk.tf.json` files and `--include-git-untracked-files` set to include the
CDKTF dynamically generated Terraform files in the Atlantis plan.
1. PR branch is pushed up containing `cdktf` code changes.
1. Atlantis checks out the branch in the repo.
1. Atlantis runs the `npm i && cdktf get && cdktf synth` command in the repo root as a step in `pre_workflow_hooks`,
generating the `cdk.tf.json` Terraform files.
1. Atlantis detects the `cdk.tf.json` untracked files in a number of directories.
1. Atlantis then runs `terraform` workflows in the respective directories as usual.

### Terragrunt

Atlantis supports running custom commands in place of the default Atlantis
commands. We can use this functionality to enable
[Terragrunt](https://github.com/gruntwork-io/terragrunt).

You can either use your repo's `atlantis.yaml` file or the Atlantis server's `repos.yaml` file.

Given a directory structure:

```plain
.
└── live
    ├── prod
    │   └── terragrunt.hcl
    └── staging
        └── terragrunt.hcl
```

If using the server `repos.yaml` file, you would use the following config:

```yaml
# repos.yaml
# Specify TERRAGRUNT_TFPATH environment variable to accommodate setting --default-tf-version
# Generate json plan via terragrunt for policy checks
repos:
- id: "/.*/"
  workflow: terragrunt
workflows:
  terragrunt:
    plan:
      steps:
      - env:
          name: TERRAGRUNT_TFPATH
          command: 'echo "terraform${ATLANTIS_TERRAFORM_VERSION}"'
      - env:
          # Reduce Terraform suggestion output
          name: TF_IN_AUTOMATION
          value: 'true'
      - run:
          # Allow for targeted plans/applies as not supported for Terraform wrappers by default
          command: terragrunt plan -input=false $(printf '%s' $COMMENT_ARGS | sed 's/,/ /g' | tr -d '\\') -no-color -out $PLANFILE
          output: hide
      - run: |
          terragrunt show $PLANFILE
    apply:
      steps:
      - env:
          name: TERRAGRUNT_TFPATH
          command: 'echo "terraform${ATLANTIS_TERRAFORM_VERSION}"'
      - env:
          # Reduce Terraform suggestion output
          name: TF_IN_AUTOMATION
          value: 'true'
      - run: terragrunt apply -input=false $PLANFILE
    import:
      steps:
      - env:
          name: TERRAGRUNT_TFPATH
          command: 'echo "terraform${DEFAULT_TERRAFORM_VERSION}"'
      - env:
          name: TF_VAR_author
          command: 'git show -s --format="%ae" $HEAD_COMMIT'
      # Allow for imports as not supported for Terraform wrappers by default
      - run: terragrunt import -input=false $(printf '%s' $COMMENT_ARGS | sed 's/,/ /' | tr -d '\\')
    state_rm:
      steps:
      - env:
          name: TERRAGRUNT_TFPATH
          command: 'echo "terraform${DEFAULT_TERRAFORM_VERSION}"'
      # Allow for state removals as not supported for Terraform wrappers by default
      - run: terragrunt state rm $(printf '%s' $COMMENT_ARGS | sed 's/,/ /' | tr -d '\\')
```

If using the repo's `atlantis.yaml` file you would use the following config:

```yaml
version: 3
projects:
- dir: live/staging
  workflow: terragrunt
- dir: live/prod
  workflow: terragrunt
workflows:
  terragrunt:
    plan:
      steps:
      - env:
          name: TERRAGRUNT_TFPATH
          command: 'echo "terraform${ATLANTIS_TERRAFORM_VERSION}"'
      - env:
          # Reduce Terraform suggestion output
          name: TF_IN_AUTOMATION
          value: 'true'
      - run:
          command: terragrunt plan -input=false -out=$PLANFILE
          output: strip_refreshing
    apply:
      steps:
      - env:
          name: TERRAGRUNT_TFPATH
          command: 'echo "terraform${ATLANTIS_TERRAFORM_VERSION}"'
      - env:
          # Reduce Terraform suggestion output
          name: TF_IN_AUTOMATION
          value: 'true'
      - run: terragrunt apply $PLANFILE
```

**NOTE:** If using the repo's `atlantis.yaml` file, you will need to specify each directory that is a Terragrunt project.

::: warning
Atlantis will need to have the `terragrunt` binary in its PATH.
If you're using Docker you can build your own image, see [Customization](deployment.md#customization).
:::

If you don't want to create/manage the repo's `atlantis.yaml` file yourself, you can use the tool [terragrunt-atlantis-config](https://github.com/transcend-io/terragrunt-atlantis-config) to generate it.

The `terragrunt-atlantis-config` tool is a community project and not maintained by the Atlantis team.

### Running custom commands

Atlantis supports running completely custom commands. In this example, we want to run
a script after every `apply`:

```yaml
# repos.yaml or atlantis.yaml
workflows:
  myworkflow:
    apply:
      steps:
      - apply
      - run: ./my-custom-script.sh
```

::: tip Notes

* We don't need to write a `plan` key under `myworkflow`. If `plan`
isn't set, Atlantis will use the default plan workflow which is what we want in this case.
* A custom command will only terminate if all output file descriptors are closed.
Therefore a custom command can only be sent to the background (e.g. for an SSH tunnel during
the terraform run) when its output is redirected to a different location. For example, Atlantis
will execute a custom script containing the following code to create a SSH tunnel correctly:
`ssh -f -M -S /tmp/ssh_tunnel -L 3306:database:3306 -N bastion 1>/dev/null 2>&1`. Without
the redirect, the script would block the Atlantis workflow.
:::

### Custom Backend Config

If you need to specify the `-backend-config` flag to `terraform init` you'll need to use a custom workflow.
In this example, we're using custom backend files to configure two remote states, one for each environment.
We're then using `.tfvars` files to load different variables for each environment.

```yaml
# repos.yaml or atlantis.yaml
workflows:
  staging:
    plan:
      steps:
      - run: rm -rf .terraform
      - init:
          extra_args: [-backend-config=staging.backend.tfvars]
      - plan:
          extra_args: [-var-file=staging.tfvars]
  production:
    plan:
      steps:
      - run: rm -rf .terraform
      - init:
          extra_args: [-backend-config=production.backend.tfvars]
      - plan:
          extra_args: [-var-file=production.tfvars]
```

::: warning NOTE
We have to use a custom `run` step to `rm -rf .terraform` because otherwise Terraform
will complain in-between commands since the backend config has changed.
:::

You would then reference the workflows in your repo-level `atlantis.yaml`:

```yaml
version: 3
projects:
- name: staging
  dir: .
  workflow: staging
- name: production
  dir: .
  workflow: production
```

### Add directory and repo context for aws resources using default tags

This is only available in AWS provider version [5.62.0](https://github.com/hashicorp/terraform-provider-aws/releases/tag/v5.62.0) and higher.

This configuration will create the following tags

* `repository` equal to `github.com/<owner>/<repo>` which can be changed for gitlab or other VCS
* `repository_dir` equal to the relative directory

Other default variables can be added such as for workspace. See below for more available environment variables.

```yaml
workflows:
  terraform:
    plan:
      steps:
        # These env vars TF_AWS_DEFAULT_TAGS_ will work for aws provider 5.62.0+
        # https://github.com/hashicorp/terraform-provider-aws/releases/tag/v5.62.0
        - &env_default_tags_repository
          env:
            name: TF_AWS_DEFAULT_TAGS_repository
            command: 'echo "github.com/${BASE_REPO_OWNER}/${BASE_REPO_NAME}"'
        - &env_default_tags_repository_dir
          env:
            name: TF_AWS_DEFAULT_TAGS_repository_dir
            command: 'echo "${REPO_REL_DIR}"'
    apply:
      steps:
        - *env_default_tags_repository
        - *env_default_tags_repository_dir
```

NOTE:

* Appending tags to every resource may regenerate data sources such as `aws_iam_policy_document` which will cause many resources to be modified. See known issue in aws provider [#29421](https://github.com/hashicorp/terraform-provider-aws/issues/29421).

* To run a local plan outside of terraform, the same environment variables will need to be created.

    ```bash
    tfvars () {
      export terraform_repository=$(git config --get remote.origin.url | sed 's,^git@,,g' | tr ':' '/' | sed 's,.git$,,g')
      export terraform_repository_dir=$(git rev-parse --show-prefix | sed 's,\/$,,g')
    }
    export TF_AWS_DEFAULT_TAGS_repository=$terraform_repository
    export TF_AWS_DEFAULT_TAGS_repository_dir=$terraform_repository_dir
    tfvars
    terraform plan
    ```

    If a colon is used in the tag name, use the `env` command instead of `export`.

    ```bash
    tfvars
    env \
      TF_AWS_DEFAULT_TAGS_org:repository=$terraform_repository \
      TF_AWS_DEFAULT_TAGS_org:repository_dir=$terraform_repository_dir \
      terraform plan
    ```

## Reference

### Workflow

```yaml
plan:
apply:
import:
state_rm:
```

| Key      | Type            | Default                   | Required | Description                           |
|----------|-----------------|---------------------------|----------|---------------------------------------|
| plan     | [Stage](#stage) | `steps: [init, plan]`     | no       | How to plan for this project.         |
| apply    | [Stage](#stage) | `steps: [apply]`          | no       | How to apply for this project.        |
| import   | [Stage](#stage) | `steps: [init, import]`   | no       | How to import for this project.       |
| state_rm | [Stage](#stage) | `steps: [init, state_rm]` | no       | How to run state rm for this project. |

### Stage

```yaml
steps:
- run: custom-command
- init
- plan:
    extra_args: [-lock=false]
```

| Key   | Type                 | Default | Required | Description                                                                                   |
|-------|----------------------|---------|----------|-----------------------------------------------------------------------------------------------|
| steps | array[[Step](#step)] | `[]`    | no       | List of steps for this stage. If the steps key is empty, no steps will be run for this stage. |

### Step

#### Built-In Commands

Steps can be a single string for a built-in command.

```yaml
- init
- plan
- apply
- import
- state_rm
```

| Key                             | Type   | Default | Required | Description                                                                                                                  |
|---------------------------------|--------|---------|----------|------------------------------------------------------------------------------------------------------------------------------|
| init/plan/apply/import/state_rm | string | none    | no       | Use a built-in command without additional configuration. Only `init`, `plan`, `apply`, `import` and `state_rm` are supported |

#### Built-In Command With Extra Args

A map from string to `extra_args` for a built-in command with extra arguments.

```yaml
- init:
    extra_args: [arg1, arg2]
- plan:
    extra_args: [arg1, arg2]
- apply:
    extra_args: [arg1, arg2]
- import:
    extra_args: [arg1, arg2]
- state_rm:
    extra_args: [arg1, arg2]
```

| Key                             | Type                               | Default | Required | Description                                                                                                                                                               |
|---------------------------------|------------------------------------|---------|----------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| init/plan/apply/import/state_rm | map\[`extra_args` -> array\[string\]\] | none    | no       | Use a built-in command and append `extra_args`. Only `init`, `plan`, `apply`, `import` and `state_rm` are supported as keys and only `extra_args` is supported as a value |

#### Custom `run` Command

A custom command can be written in 2 ways

Compact:

```yaml
- run: custom-command arg1 arg2
```

| Key | Type   | Default | Required | Description          |
|-----|--------|---------|----------|----------------------|
| run | string | none    | no       | Run a custom command |

Full

```yaml
- run:
    command: custom-command arg1 arg2
    shell: sh
    shellArgs:
     - "--debug"
     - "-c"
    output: show
```

| Key | Type                                                         | Default | Required | Description                                                                                                                                                                                                                                                                                                                                                                                             |
|-----|--------------------------------------------------------------|---------|----------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| run | map\[string -> string\] | none    | no       | Run a custom command                                                                                                                                                                                                                                                                                                                                                                                    |
| run.command | string                                                       | none | yes      | Shell command to run                                                                                                                                                                                                                                                                                                                                                                                    |
| run.shell | string | "sh" | no | Name of the shell to use for command execution |
| run.shellArgs | string or []string | "-c" | no | Command line arguments to be passed to the shell. Cannot be set without `shell` |
| run.output | string                                                       | "show" | no       | How to post-process the output of this command when posted in the PR comment. The options are<br/>*`show` - preserve the full output<br/>* `hide` - hide output from comment (still visible in the real-time streaming output)<br/> * `strip_refreshing` - hide all output up until and including the last line containing "Refreshing...". This matches the behavior of the built-in `plan` command |

#### Native Environment Variables

* `run` steps in the main `workflow` are executed with the following environment variables:
  note: these variables are not available to `pre` or `post` workflows
  * `WORKSPACE` - The Terraform workspace used for this project, ex. `default`.
      NOTE: if the step is executed before `init` then Atlantis won't have switched to this workspace yet.
  * `ATLANTIS_TERRAFORM_VERSION` - The version of Terraform used for this project, ex. `0.11.0`.
  * `DIR` - Absolute path to the current directory.
  * `PLANFILE` - Absolute path to the location where Atlantis expects the plan to
      either be generated (by plan) or already exist (if running apply). Can be used to
      override the built-in `plan`/`apply` commands, ex. `run: terraform plan -out $PLANFILE`.
  * `SHOWFILE` - Absolute path to the location where Atlantis expects the plan in json format to
      either be generated (by show) or already exist (if running policy checks). Can be used to
      override the built-in `plan`/`apply` commands, ex. `run: terraform show -json $PLANFILE > $SHOWFILE`.
  * `POLICYCHECKFILE` - Absolute path to the location of policy check output if Atlantis runs policy checks.
      See [policy checking](policy-checking.md#data-for-custom-run-steps) for information of data structure.
  * `BASE_REPO_NAME` - Name of the repository that the pull request will be merged into, ex. `atlantis`.
  * `BASE_REPO_OWNER` - Owner of the repository that the pull request will be merged into, ex. `runatlantis`.
  * `HEAD_REPO_NAME` - Name of the repository that is getting merged into the base repository, ex. `atlantis`.
  * `HEAD_REPO_OWNER` - Owner of the repository that is getting merged into the base repository, ex. `acme-corp`.
  * `HEAD_BRANCH_NAME` - Name of the head branch of the pull request (the branch that is getting merged into the base)
  * `HEAD_COMMIT` - The sha256 that points to the head of the branch that is being pull requested into the base. If the pull request is from Bitbucket Cloud the string will only be 12 characters long because Bitbucket Cloud truncates its commit IDs.
  * `BASE_BRANCH_NAME` - Name of the base branch of the pull request (the branch that the pull request is getting merged into)
  * `PROJECT_NAME` - Name of the project configured in `atlantis.yaml`. If no project name is configured this will be an empty string.
  * `PULL_NUM` - Pull request number or ID, ex. `2`.
  * `PULL_URL` - Pull request URL, ex. `https://github.com/runatlantis/atlantis/pull/2`.
  * `PULL_AUTHOR` - Username of the pull request author, ex. `acme-user`.
  * `REPO_REL_DIR` - The relative path of the project in the repository. For example if your project is in `dir1/dir2/` then this will be set to `"dir1/dir2"`. If your project is at the root this will be `"."`.
  * `USER_NAME` - Username of the VCS user running command, ex. `acme-user`. During an autoplan, the user will be the Atlantis API user, ex. `atlantis`.
  * `COMMENT_ARGS` - Any additional flags passed in the comment on the pull request. Flags are separated by commas and
      every character is escaped, ex. `atlantis plan -- arg1 arg2` will result in `COMMENT_ARGS=\a\r\g\1,\a\r\g\2`.
* A custom command will only terminate if all output file descriptors are closed.
Therefore a custom command can only be sent to the background (e.g. for an SSH tunnel during
the terraform run) when its output is redirected to a different location. For example, Atlantis
will execute a custom script containing the following code to create a SSH tunnel correctly:
`ssh -f -M -S /tmp/ssh_tunnel -L 3306:database:3306 -N bastion 1>/dev/null 2>&1`. Without
the redirect, the script would block the Atlantis workflow.
* If a workflow step returns a non-zero exit code, the workflow will stop.
:::

#### Environment Variable `env` Command

The `env` command allows you to set environment variables that will be available
to all steps defined **below** the `env` step.

You can set hard coded values via the `value` key, or set dynamic values via
the `command` key which allows you to run any command and uses the output
as the environment variable value.

```yaml
- env:
    name: ENV_NAME
    value: hard-coded-value
- env:
    name: ENV_NAME_2
    command: 'echo "dynamic-value-$(date)"'
- env:
    name: ENV_NAME_3
    command: echo ${DIR%$REPO_REL_DIR}
    shell: bash
    shellArgs:
      - "--verbose"
      - "-c"
```

| Key             | Type                  | Default | Required | Description                                                                                                     |
|-----------------|-----------------------|---------|----------|-----------------------------------------------------------------------------------------------------------------|
| env | map\[string -> string\] | none    | no       | Set environment variables for subsequent steps                                                                  |
| env.name | string | none | yes | Name of the environment variable                                                                                |
| env.value | string | none | no | Set the value of the environment variable to a hard-coded string. Cannot be set at the same time as `command`   |
| env.command | string | none | no | Set the value of the environment variable to the output of a command. Cannot be set at the same time as `value` |
| env.shell | string | "sh" | no | Name of the shell to use for command execution. Cannot be set without `command` |
| env.shellArgs | string or []string | "-c" | no | Command line arguments to be passed to the shell. Cannot be set without `shell` |

::: tip Notes

* `env` `command`'s can use any of the built-in environment variables available
  to `run` commands.
:::

#### Multiple Environment Variables `multienv` Command

The `multienv` command allows you to set dynamic number of multiple environment variables that will be available
to all steps defined **below** the `multienv` step.

Compact:

```yaml
- multienv: custom-command
```

| Key      | Type   | Default | Required | Description                                                |
|----------|--------|---------|----------|------------------------------------------------------------|
| multienv | string | none    | no       | Run a custom command and add printed environment variables |

Full:

```yaml
- multienv:
    command: custom-command
    shell: bash
    shellArgs:
      - "--verbose"
      - "-c"
    output: show
```

| Key                | Type                  | Default | Required | Description                                                                         |
|--------------------|-----------------------|---------|----------|-------------------------------------------------------------------------------------|
| multienv           | map[string -> string] | none    | no       | Run a custom command and add printed environment variables                          |
| multienv.command   | string                | none    | yes      | Name of the custom script to run                                                    |
| multienv.shell     | string                | "sh"    | no       | Name of the shell to use for command execution                                      |
| multienv.shellArgs | string or []string    | "-c"    | no       | Command line arguments to be passed to the shell. Cannot be set without `shell`     |
| multienv.output    | string                | "show"  | no       | Setting output to "hide" will suppress the message obout added environment variables |

The output of the command execution must have the following format:
`EnvVar1Name=value1,EnvVar2Name=value2,EnvVar3Name=value3`

The name-value pairs in the output are added as environment variables if command execution is successful, otherwise the workflow execution is interrupted with an error and the errorMessage is returned.

::: tip Notes

* `multienv` `command`'s can use any of the built-in environment variables available
  to `run` commands.
:::
