<img src="https://raw.githubusercontent.com/elbb/bb-buildingblock/master/.assets/logo.png" height="200">

# Concourse pipeline integration guide

**Before continue reading please make sure that you copied the example pipeline to its final destination.**

    $ cp -r example/ci/* .

For using this concourse CI pipeline it is recommended to have two config files:

-   `config.yaml`: for production purposes
-   `config.local.yaml`: for development and testing purposes

The `config.yaml` is used to access the projects production release repository and branch and a real cloud S3 endpoint to store version artifacts.
The `config.local.yaml` is used to develop and test the pipeline for building blocks. It is used to work with your private fork or local git repository, your local docker registry as well as a local S3 endpoint.

# Example concourse CI pipeline

This example is intended to show the integration of a build pipeline in a derived building block.

## Adaptions for your building block

This section describes what needs to be done within the several files.
**Note: `credentials.yaml` is ignored by `.gitignore` and will not be checked in.**

### config.yaml

Open the `config.yaml` and specify the fields that are described.
This config file shows an example for a fictive building block called `bb-example`.

Note 1: If you are using a real S3 endpoint you should enter those data. If you are using the local test environment specify like below.\
Note 2: If you are using e.g. dockerhub as destination, you can specify `bb_docker_namespace: your_docker_username_or_organization`\
Note 3: If you are running the local docker registry, you might add this to the `bb_insecure_registries` section. If you are using e.g. dockerhub or another ssl featured registry you can leave the brackets empty. e.g. `bb_insecure_registries: []`.

    ---
    # General stuff
    # -------------
    ## Specifies the name of the building block
    bb_name: bb-example
    ## Specify the namespace for the building block image. This gets prefixed before <prefix>/<bb_name>
    ### example: bb_docker_namespace: "registry:5000/elbb" results in using "registry:5000/elbb/<bb_name>"
    ### "elbb" means using the docker hub, e.g. "elbb/<bb-name>"
    bb_docker_namespace: "registry:5000/elbb"
    ## Specifies insecure docker registries, format "host:port" or "ip:port"
    bb_insecure_registries: ["registry:5000"]
    ## Specify if 'latest' tags are build for `git_branch`
    bb_release_as_latest: true

    ## Specifies the version used for bb-gitversion
    bb_gitversion_version: 0.5.1

    # Git relevant stuff
    # ------------------
    ## Specify the git repository to work with
    git_source: https://github.com/elbb/bb-example.git
    ## Specify the git branch to work with
    git_branch: example
    ## This enables/disables ssl verification of the git resource
    git_skip_ssl_verification: false

    # S3 relevant stuff
    # -----------------
    ## Specify the S3 endpoint URI
    # Using local environment for min.io
    s3_endpoint: http://minio:9000
    ## This enables/disables the ssl verification for the S3 endpoint
    s3_skip_ssl_verification: true

### credentials.yaml

**Note: Make sure that you have set up ssh access with your ssh-key in your git repository.**

Fill in your ssh private key and your credentials for the docker registry and S3 server in `credentials.example.yaml` and rename it to `credentials.yaml`. Note the indentation of the ssh key as in the example file.

### Email notification

The concourse ci environment can automatically send an e-mail notification about the current build status.
Copy the file `ci/email.template.yaml` to `ci/email.yaml` and enter the email server configuration and email addresses.
For further information how to configure the email notification, see: <https://github.com/pivotal-cf/email-resource>

**Note: `email.yaml` is ignored by `.gitignore` and will not be checked in.**

The provided pipeline.yaml file demonstrates the usage of email notification.

### Upload to concourse

The pipeline file must be uploaded to concourse CI via `fly`.
Before setting the pipeline you might login first to your concourse instance `fly -t <target> login --concourse-url http://<concourse>:<port>`. See the [fly documentation](https://concourse-ci.org/fly.html) for more help.
Upload the pipeline file with fly:

    $ fly -t <target> set-pipeline -n -p bb-buildingblock -l ci/config.yaml -l ci/credentials.yaml -l ci/email.yaml -c pipeline.yaml

After successfully uploading the pipeline to concourse CI login and unpause it. After that the pipeline should be triggered by new commits on the master branch (or new tags if enabled in `pipeline.yaml`).

### Microsoft Teams notification

In addition to email notification, it is possible to send a notification to Microsoft Teams about the current status of the ci build.
Copy the file `ci/msteams.template.yaml` to `ci/msteams.yaml` and enter the webhook url for your  ms teams channel.
For  webhook url generation, see [MS Teams setup](####MS-Teams-setup).

**Note: `msteams.yaml` is ignored by `.gitignore` and will not be checked in.**

The provided pipeline.yaml file demonstrates the usage of ms teams notification.

#### MS Teams setup

- Open the Microsoft Teams UI.
- Identify the channel you wish to post notifications to - ie: #devops....
- Open the "more options" menu of that channel and select "Connectors".
- Select "Incoming Webhook" and respond to the propts for details like the icon and name of the connector.
- Use the webhook url from above in your pipeline source definition.


### pipeline.yaml

We assume that the git repository for bb-example contains parts the following file structure:

    .
    ├── docker
    |   ├── Dockerfile

As a build artifact a docker image is wanted versioned with bb-gitversion.

The example builds the docker image with the Dockerfile located in `docker/Dockerfile` and creates a tag generated by a previous step running `bb-gitversion`.

Adapt the last section in the `pipeline.yaml` to match your needs. The `image-((bb_name))` will be called `image-bb-example` and is needed to define the target docker image. The name and docker namespace is configured in `config.yaml`. This image section does not need to be modified. But you might to modify the job called `Create docker image for ((bb_name)) and push it`.
The only parts you need to modify are in the last job:

-   `name: Create docker image for ((bb_name)) and push it`

There you need to modify these sections

-   `build` (docker build context)
-   `dockerfile` (location of your Dockerfile).

The pipeline can create a docker image with two tags:

1.  For your specified target branch with versioning information, either with tag `FullSemVer` for `master` or `InformationalVersion` for every other branch (see <https://gitversion.net/docs/more-info/variables> for further information on SemVer)

2.  `latest` if `bb_release_as_latest` is set to `true` in `config.yaml`.

`source` specifies the root of your checked out git repository and does not need to be adapted.

    resources:
    ...
      - name: image-((bb_name))
        type: docker-image
        source:
          repository: ((bb_docker_namespace))/((bb_name))
          username: ((registry_user))
          password: ((registry_password))
          insecure_registries: ((bb_insecure_registries))
    ...
    jobs:
    ...
      - name: Create docker image for ((bb_name)) and push it
        public: true
        plan:
          - in_parallel:
              - get: source
              - get: s3-gitversion
                passed: [Generate gitversion and put it on S3]
                trigger: true
                params:
                  unpack: true
          - put: image-((bb_name))
            params:
              build: source/docker
              dockerfile: source/docker/Dockerfile
              tag_as_latest: ((bb_release_as_latest))
              tag_file: s3-gitversion/version

Note: If the S3 bucket for the specific building block does not exist, it will be created. The files that are stored within that bucket contain various versioning information. The name is suffixed with the Version: `gitversion-<Version>.tar.gz` where `<Version>` is a full semantic version for `master` branch (e.g. `bb-example-0.1.0`) and has additional informations for every `other branch` (e.g. `0.1.0-Branch.origin-example-branch.Sha.f48c55b4dea2b666574f03f34e02001123f7993a`).

## Using the pipeline

Use `fly` to setup the pipeline with concourse CI.
Before setting the pipeline you might login first to your concourse instance `fly -t <target> login --concourse-url http://<concourse>:<port>`. See the [fly documentation](https://concourse-ci.org/fly.html) for more help.

Once you've setup your concourse CI server (either hosted or local) upload the pipeline:

    $ cd example/ci
    $ fly -t <target> set-pipeline -n -p bb-example -l ci/config.yaml -l ci/credentials.yaml -c pipeline.yaml
    $
    $ # example with email notification
    $ fly -t <target> set-pipeline -n -p bb-example -l ci/config.yaml -l ci/credentials.yaml -l ci/email.yaml -c pipeline.yaml
    $
    $ # example with ms teams notification
    $ fly -t <target> set-pipeline -n -p bb-example -l ci/config.yaml -l ci/credentials.yaml -l ci/msteams.yaml -c pipeline.yaml

If your build succeeds, overwrite the the `pipeline.yaml` and `ci/` with your modified pipeline from `example/ci`. You are free to remove `example/ci`.
