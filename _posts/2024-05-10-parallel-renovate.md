---
layout: post
title: Running renovate in parallel
date: 2024-05-10 06:00:00
description: Automated dependency updates at scale
tags: renovate, gitlabci
categories: devops
featured: true
---

# Setting

[Renovate Bot](https://docs.renovatebot.com/) is an automated tool that
manages software dependencies. It simplifies project maintenance by automatically
updating dependencies to their latest versions through pull requests. This 
automation not only saves time but also enhances security by minimizing 
vulnerabilities associated with outdated software. Users can customize how and
when updates are applied, ensuring minimal disruption to the development workflow.
By maintaining dependency versions consistently and allowing configurable update
rules, Renovate Bot supports stable development environments and efficient project
lifecycles.

When running Renovate on GitLab CI with the [official GitLab CI templates](https://gitlab.com/renovate-bot/renovate-runner/-/tree/8c3e51c90522cdfdf0d504695fd63f3474e2af5a/templates),
the Renovate Job can easily take multiple tens of minutes to go through a project's
repositories. In the example Pipeline below, renovate scanned 52 GitLab repositories
for dependencies from the below managers:

- npm
- kustomize
- dockerfile
- composer
- gitlabci
- gitlabci-include
- terraform
- terraform-version

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/blog/2024-05-10-parallel-renovate/renovate-single-process-length.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

With roughly 30 minutes of execution time, we can schedule this pipeline only
about once an hour. In the project, renovate is not only used to update
dependencies from external projects but also from our own project's application
components. Therefore, once an hour is the maximum acceptable schedule. At this
stage of the project we knew that there were many more repositories to be created.
Thus there was an urge for optimisation of the renovate pipeline.

# Setup

We will have two repositories:

- `renovate-image`, for building a custom renovate container image
- `renovate-bot`, for holding renovate default configuration and this is were
the pipeline runs

## renovate-image

A custom renovate container image is needed for two reasons. First, we need to 
add a `.gitlab-ci.yaml` template to the image to construct a dynamic GitLab CI
Job. Second, we can add extra tools to the image to run renovate
[`postUpgradeTasks`](https://docs.renovatebot.com/configuration-options/#postupgradetasks).

`/Dockerfile`
```Dockerfile
FROM ghcr.io/renovatebot/renovate:37.351.0@sha256:3286096674fc3e5d5d3e74698c144113930ffbc4f900ddc9f99d6c81c682f448
USER 0

# install tools here

WORKDIR /usr/src/app
COPY template/.gitlab-ci.yaml ./template/.gitlab-ci.yml
USER ubuntu
```

`/template/.gitlab-ci.yml`
```yaml
---
include:
  - project: 'renovate-bot'
    ref: main
    file: '.template.gitlab-ci.yml'

renovate:
  extends: .template
  rules:
    - if: $CI_PIPELINE_SOURCE == "parent_pipeline"
  script:
    - renovate $RENOVATE_EXTRA_FLAGS
  cache:
    key: ${CI_COMMIT_REF_SLUG}-renovate
    paths:
      - renovate/cache/renovate/repository/
  resource_group: $RENOVATE_EXTRA_FLAGS
  artifacts:
    when: always
    expire_in: 1d
    paths:
      - '$RENOVATE_LOG_FILE'
  parallel:
    matrix:
      - RENOVATE_EXTRA_FLAGS: ###RENOVATE_REPOS###
```

## renovate-bot

The definition of the `image` is placed into an extra file called `.template.gitlab-ci.yml`.
This way we can, `extend` from the `.template` job by `include`ing the 
`.template.gitlab-ci.yml` file into a pipeline configuration from `local` and
from `project` (see `/template/.gitlab-ci.yml` in [renovate-image](#renovate-image).

`.template.gitlab-ci.yml`
```yaml
---
.template:
  variables:
    RENOVATE_ENDPOINT: $CI_API_V4_URL
    RENOVATE_PLATFORM: gitlab
  image: registry.my-domain.de/renovate-image:v1.1.5@sha256:4a0cfbdd03f691d96955fe41fc8bec49b011afc06519d3122261ead2f69beb6f
```

Moreover, by adapting renovate `gitlab-ci` manager configuration in the `renovate.json`
we can ensure updates to the `image`. The below regex matches both `.gitlab-ci.yml`
and `.template.gitlab-ci.yml` files.

`renovate.json`
```json
{
  "gitlabci": {
    "fileMatch": [
      "(\\..+)?\\.gitlab-ci\\.ya?ml$"
    ]
  }
}
```

Now, the actual renovate pipeline, includes the above template and defines three
jobs.
- `renovate-config-validator` is for validating the renovate configuration.
- `renovate:discover` finds all repositories which dependencies shall be scanned
by renovate. Furthermore, it creates a GitLab CI pipeline file named `.gitlab-renovate-repos.yml`
from the `template/.gitlab-ci.yml` and replaces the placeholder `###RENOVATE_REPOS###`
with the discovered repositories.
- `renovate:repos` uses the `.gitlab-renovate-repos.yml` from the `renovate:discover`
job to trigger the jobs defined in the pipeline configuration.

`.gitlab-ci.yml`
```yaml
---
stages:
  - test
  - build
  - deploy

include:
  - local: "/.template.gitlab-ci.yml"

renovate-config-validator:
  extends: .template
  stage: test
  script:
    - renovate-config-validator $RENOVATE_CONFIG_VALIDATOR_EXTRA_FLAGS
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS
      when: never
    - if: $CI_COMMIT_BRANCH
    - if: $CI_COMMIT_TAG != null
    - if: $CI_PIPELINE_SOURCE == "schedule" || $CI_PIPELINE_SOURCE == "web"

renovate:discover:
  extends: .template
  stage: build
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule" || $CI_PIPELINE_SOURCE == "web"
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  variables:
    # A Personal Access Token with authorization as described https://docs.renovatebot.com/modules/platform/gitlab/#authentication
    # Add the token as a CI/CD variable with name PAT_RENOVATE
    RENOVATE_TOKEN: $PAT_RENOVATE
    RENOVATE_AUTODISCOVER: 'true'
    RENOVATE_AUTODISCOVER_FILTER: '["group1/subgroup1/**", "group2/**"]' # define your repos here
  script:
    - renovate --write-discovered-repos=renovate-repos.json
    - sed "s~###RENOVATE_REPOS###~$(cat renovate-repos.json)~" /usr/src/app/template/.gitlab-ci.yml > .gitlab-renovate-repos.yml
  artifacts:
    paths:
      - renovate-repos.json
      - .gitlab-renovate-repos.yml

renovate:repos:
  stage: deploy
  needs:
    - renovate:discover
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule" || $CI_PIPELINE_SOURCE == "web"
    - if: $CI_PIPELINE_SOURCE == "pipeline"
  inherit:
    variables: true
  variables:
    # base config from https://gitlab.com/renovate-bot/renovate-runner/-/blob/main/templates/renovate.gitlab-ci.yml?ref_type=heads
    # must be a writable container path as no project is cloned in child pipeline
    RENOVATE_BASE_DIR: /tmp/renovate
    RENOVATE_OPTIMIZE_FOR_DISABLED: 'true'
    RENOVATE_REPOSITORY_CACHE: 'enabled'
    RENOVATE_LOG_FILE: renovate-log.ndjson
    RENOVATE_LOG_FILE_LEVEL: debug

    # custom config
    RENOVATE_TOKEN: $PAT_RENOVATE
    RENOVATE_GIT_AUTHOR: 'Renovate Bot <no-reply@mydomain.de>'
    RENOVATE_OSV_VULNERABILITY_ALERTS: 'true'
    RENOVATE_FORK_PROCESSING: 'enabled'
    RENOVATE_ONBOARDING: 'false'
    RENOVATE_ONBOARDING_CONFIG: '{"$$schema": "https://docs.renovatebot.com/renovate-schema.json", "extends": ["local>renovate-bot:renovate.json"] }'
    RENOVATE_GITHUB_TOKEN_WARN: 'false'
    RENOVATE_REQUIRE_CONFIG: 'ignored'
    RENOVATE_CONFIG_FILE: 'renovate.json'
    LOG_LEVEL: debug
  trigger:
    forward:
      yaml_variables: true
      pipeline_variables: true
    include:
      - job: renovate:discover
        artifact: .gitlab-renovate-repos.yml
```

# Results

55 GitLab repositories are now scanned for dependencies (and updated) by renovate
in 8 Minutes.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/blog/2024-05-10-parallel-renovate/renovate-single-parallel-length-1.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/blog/2024-05-10-parallel-renovate/renovate-single-parallel-length-2.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>