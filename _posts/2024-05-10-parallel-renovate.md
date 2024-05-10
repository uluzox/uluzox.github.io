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
<script src="https://gist.github.com/uluzox/aa26e82b11b6e5093ae59d9e09e9e0c1.js"></script>

`/template/.gitlab-ci.yml`
<script src="https://gist.github.com/uluzox/bf1a1c574369e411b89aa46adf8486fc.js"></script>

## renovate-bot

The definition of the `image` is placed into an extra file called `.template.gitlab-ci.yml`.
This way we can, `extend` from the `.template` job by `include`ing the 
`.template.gitlab-ci.yml` file into a pipeline configuration from `local` and
from `project` (see `/template/.gitlab-ci.yml` in [renovate-image](#renovate-image).

`.template.gitlab-ci.yml`
<script src="https://gist.github.com/uluzox/5424aeabf41ebb3cdf61f07450214ba6.js"></script>

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
<script src="https://gist.github.com/uluzox/0f62f7a318e391b3265dcf148f75ca61.js"></script>

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