<img src="https://raw.githubusercontent.com/elbb/bb-buildingblock/master/.assets/logo.png" height="200">

# (e)mbedded (l)inux (b)uilding (b)lock template

This code serves as a template for the creation of further building blocks with the purpose of giving all blocks a uniform structure and usage.

## Prerequisites

-   [docker](https://docs.docker.com/install/)
-   [dobi](https://github.com/dnephin/dobi) (downloaded if not in `PATH`)
-   [concourse](https://concourse-ci.org/) (ci/cd)

## Howto use bb-buildingblock template

### Initialize your project
Create a new repository and merge bb-buildingblock, e.g.:
```bash
git init myrepo
cd myrepo
git remote add bb-buildingblock https://github.com/elbb/bb-buildingblock.git
git fetch --no-tags bb-buildingblock
git merge bb-buildingblock/master
```
**Important: This template uses `bb-gitversion` to version itself on the basis of git tags. Therefore it is important that you don't fetch tags from the git remote `bb-buildingblock`. If you don't use the git cli, verify that your git tool didn't fetch tags.** \
Possibly delete local tags:
```bash
git tag -d $(git tag -l)
```

### Merge from upstream

If you want to merge changes from upstream:
```bash
git fetch --no-tags bb-buildingblock
git merge bb-buildingblock/master
```

**Important: If you don't use the git cli, verify that your git tool didn't fetch tags.** \
Possibly delete local tags and fetch tags from origin:
```bash
git tag -d $(git tag -l)
git fetch --tags origin
```

### Toolchains

Follow "[Using dobi for local build](#using-dobi-for-local-build)" to create your `build`, `test` and `deploy` jobs for your local development environment.

Follow "[Using concourse CI for a CI/CD build](#using-concourse-ci-for-a-cicd-build)" to adapt your CI/CD pipeline configuartion.

## Using dobi for local build

`dobi` should only be used via the `dobi.sh` script. The script sets environment variables and includes checks.

You can find the upstream documentation at [https://dnephin.github.io/dobi/](https://dnephin.github.io/dobi/).

By default three dobi resources are predefined (but not implemented):

```sh
./dobi.sh build  # build the building block
./dobi.sh test   # run all tests
./dobi.sh deploy # deploy the building block
```

These point to the resources defined in dobi.yaml.
The separation between meta.yaml and dobi.yaml is necessary to integrate the building block into another building block via dobi.

Version information is generated automatically from git history by using building block [bb-gitversion](<https://github.com/elbb/bb-gitversion>).
You can reference this version information by accessing it e.g. via `{env.GitVersion_BranchVersion}`. See "[Versioning](#versioning)" for more information.

Example job:
```yaml
image=debian:
  image: debian:sid-slim
  pull: once

job=echo-version
  use: debian
  command: bash -c "echo ${VERSION}"
  env:
    - "VERSION={env.GitVersion_BranchVersion}"
```

### Default project variables

Edit `./default.env` to set default project variables.

### Local project variables

If you want to override project variables, copy `./local.env.template` to `./local.env` and edit `./local.env` accordingly.<br>
`./local.env` is ignored by git via `./.gitignore`.

## Using concourse CI for a CI/CD build

We provide an example concourse CI/CD pipeline in `./example/ci`. See the [integration documentation](README_CICD_INTEGRATION.md) on how to modify the `pipeline.yaml` and config files for your building block.

## Versioning

This template building block uses [bb-gitversion](https://github.com/elbb/bb-gitversion) to compute the semantic version of itself from the git history on the bases of git tags.

Using `dobi.sh` the version is updated when necessary and published in environment variables. You can use these environment variables in your dobi targets.
See `./gen/gitversion/env/gitversion.env` for a detailed list of version environment variables.
These key-value pairs are also generated into other representations:
```
./gen/gitversion/cpp/elbb/version.h    #c++ header
./gen/gitversion/json/gitversion.json  #json reprentation
./gen/gitversion/plain/*               #Filename==Key, Content==Value
```
`GitVersion_BranchVersion` is a special key. This version information has different content for `master` and `non-master` branches.
It is identically to `FullSemVer` in case of using the master branch. On all other branches it is identically to `InformationalVersion`.

### Examples

You can find an example how to use `GitVersion_BranchVersion` in a dobi job at "[Using dobi for local build](#using-dobi-for-local-build)"

You can find an example how to use `GitVersion_BranchVersion` in a cicd pipeline at [./example/ci/pipeline.yaml](./example/ci/pipeline.yaml)

# What is embedded linux building blocks

embedded linux building blocks is a project to create reusable and
adoptable blueprints for highly recurrent issues in building an internet
connected embedded linux system.

# License

Licensed under either of

-   Apache License, Version 2.0, (./LICENSE-APACHE or <http://www.apache.org/licenses/LICENSE-2.0>)
-   MIT license (./LICENSE-MIT or <http://opensource.org/licenses/MIT>)

at your option.

# Contribution

Unless you explicitly state otherwise, any contribution intentionally
submitted for inclusion in the work by you, as defined in the Apache-2.0
license, shall be dual licensed as above, without any additional terms or
conditions.

Copyright (c) 2020 conplement AG
