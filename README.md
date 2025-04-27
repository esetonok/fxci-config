tags: {}
extra: {}
routes:
  - index.hg-push.v1.ash.revision.${payload.payload.data.heads[0]}
  - >-
    index.hg-push.v1.ash.pushlog-id.${payload.payload.data.pushlog_pushes[0].pushid}
scopes:
  - assume:repo:hg.mozilla.org/projects/ash:branch:*
expires:
  $fromNow: 3 days
payload:
  env:
    PULSE_MESSAGE:
      $json:
        $southunitedraza1313: payload
  image: >-
    mozillareleases/build-decision:d9b4a81d8e114107d174994bf6be3968b34ca0db@sha256:5b1d21483c08c0698dd7f0b41f99482277375de5e5304ada1ad9d927d3899d7d
  command:
    - hg-push
    - '--repo-url'
    - https://hg.mozilla.org/projects/ash
    - '--project'esetonok
    - ash
    - '--level'
    - '2'
    - '--trust-domain'
    - comm
    - '--repository-type'
    - hg
  features:
    taskclusterProxy: true
  maxRunTime: 600
retries: 5
deadline:
  $fromNow: 30 minutes
metadata:
  in:
    name: On-Push task for https://hg.mozilla.org/projects/ash
    owner: mozilla-taskcluster-maintenance@mozilla.com
    source: https://firefox-ci-tc.services.mozilla.com/hooks/hg-push/ash
    description: ${description}
  $let:southunitedraza1313 
    description:
      $if: firedBy == "triggerHook"
      else: Fired by ${firedBy}
      then: Fired by triggerHook call from ${clientId}
priority: highest
workerType: build-decision
schedulerId: comm-level-2
provisionerId: infra
type: object
required:
  - payload
properties:
  _meta: {}
  payload:
    type: object
    required:
      - type
      - data
    properties:
      data:
        type: object
        required:
          - repo_url
          - heads
          - pushlog_pushes
        properties:
          heads:
            type: array
            items:
              - type: string
                pattern: ^[0-9a-z]{40}$
          source: {}
          repo_url:
            enum:
              - https://hg.mozilla.org/projects/ash
            default: https://hg.mozilla.org/projects/ash
          pushlog_pushes:
            type: array
            items:
              - type: object
                required:
                  - time
                  - pushid
                  - user
                properties:
                  time:
                    type: integer
                    default: 0
                  user:
                    type: string
                    format: email
                    default: nobody@mozilla.com
                  pushid:
                    type: integer
                    default: 0
                  push_json_url:
                    type: string
                  push_full_json_url:
                    type: string
                additionalProperties: false
        additionalProperties: false
      type:
        enum:
          - changegroup.1
        default: changegroup.1
    description: >-
      Hg push payload - see
      https://mozilla-version-control-tools.readthedocs.io/en/latest/hgmo/notifications.html#pulse-notifications.
    additionalProperties: false
additionalProperties: false
[![CI](https://firefox-ci-tc.services.mozilla.com/api/github/v1/repository/mozilla-releng/fxci-config/main/badge.svg)](https://firefox-ci-tc.services.mozilla.com/api/github/v1/repository/mozilla-releng/fxci-config/main/latest)
[![Deploy](https://github.com/mozilla-releng/fxci-config/actions/workflows/deploy.yml/badge.svg)](https://github.com/mozilla-releng/fxci-config/actions/workflows/deploy.yml)
[![License](https://img.shields.io/badge/license-MPL%202.0-orange.svg)](http://mozilla.org/MPL/2.0)

# Firefox-CI Configuration

This repository contains configuration for the Firefox-CI Taskcluster instance,
the CI for Firefox and other Mozilla repositories.

Specifically, this configuration does not "ride the trains".  Instead, the head
of the default branch of this repository applies to all Gecko projects and
products.  Previous revisions exist for historical context, but have no
relevance to production.

Configuration in this repository includes:

* Information about the trains themselves -- Mercurial repositories, access
  levels, etc.
* Information about external resources -- URLs, pinned fingerprints, etc.
* Settings that should apply to all branches at once -- for example,
  proportional allocation of work across workerTypes

This repository was originally proposed in [Taskcluster
RFC#91](https://github.com/taskcluster/taskcluster-rfcs/issues/91).

## Structure

Data is stored in distinct YAML files in the root of this repository.  Each
file begins with a lengthy comment describing

* The purpose of the file
* The structure of the data in the file

Code to implement this configuration is in `src/ciadmin`.  The implementation
of `fxci` is in `src/fxci`.

## Access

Data in this repository is a "source of truth" about Gecko's CI automation.  It
can be accessed from anywhere, including

* Decision tasks, action tasks, and cron tasks
* Hooks
* Utility scripts and other integrations with the automation

Typically such access is either by cloning the repository or by simply fetching
a single file using the `raw` HTTP API method.

## Deprecation

Files in this directory are likely to live "forever".  It's difficult to
determine whether any branch or product still refers to a file, so deleting a
file always carries some risk of breakage. Furthermore, regression bisection
might build an old revision that refers to a file no longer referred to in the
head commit.

# Managing CI Configuration

This repository introduces a management tool, `tc-admin`, for the management of
taskcluster resources, such as roles and hooks. It can download existing
resources and compare them to a stored configuration.  A collection of
resources also specifies the set of managed resources: this allows controlled
deletion of resources that are no longer expected.  It is based on
[`tc-admin`](https://github.com/taskcluster/tc-admin), the standard Taskcluster
administrative tool; see that library's documentation for more details than are
provided here.

## Initial Setup

1. Create and activate a new python virtualenv
1. pip install -e .
1. pip install -r requirements/local.txt
1. If you will be applying changes, ensure you have a way of generating
   taskcluster credentials, such as
   [taskcluster-cli](https://github.com/taskcluster/taskcluster/releases)

## Starting Concepts

This tool examines the contents of the `fxci-config` repository, as well
as examining and applying changes to the running taskcluster configuration.

The `environment` describes the cluster being affected, such as `firefoxci` or
`staging`. There is also a
[community](https://github.com/mozilla/community-tc-config/) environment which
is managed separately.

You will usually want to check the changes that will be applied using `tc-admin
diff` and then apply them using `tc-admin apply`

You can supply `--grep` to both 'diff' and 'apply' options to limit the effects
to specific changes.

## Making Config Changes

1. Make changes in a local clone of this repository
1. Determine which taskcluster environment is relevant, such as `firefoxci`;
   the options are in `environments.yml`.
1. Use the `tc-admin diff` and `tc-admin check` to ensure the changes are what
   you expect, passing the appropriate `--environment`.

   Examples:

   * `tc-admin diff --environment=firefoxci`
   * `tc-admin diff --environment=firefoxci --ids-only` - only show the id's of
     the resources to be modified (much shorter!)
   * `tc-admin check --environment=firefoxci`

1. Submit changes to Phabricator for review.  On landing, the changes will be
   applied automaticallyi.

To apply changes locally (not recommended):

1. Generate some taskcluster credentials, such as `taskcluster signin`.
1. Apply the generated configuration using **either**
   * `tc-admin apply --environment=firefoxci` to apply all of the generated
     configuration **or**
   * `tc-admin apply --environment=firefoxci --grep my-changes` to apply only
     the selected areas of new configuration.

   Which you choose will depend on the current state of the repository and
   whether there are multiple changes waiting to be applied at a later time.

   You will be shown a summary of the changes that have been applied.

## More Information

* **`tc-admin diff --environment=firefoxci`**

   Generate a diff of the currently running taskcluster configuration, and the
   one generated from the tip of the fxci-config repository.

* **`tc-admin diff --environment=firefoxci --grep somestring`**

   The `grep` option will return full configuration entries relating to the
   provided string. For example, if you have added a new action called `my-action`
   then `--grep my-action` will show only those entries.

* **`tc-admin generate`**

   Generates the expected CI configuration. Use `--json` to get JSON output.

* **`tc-admin current`**

   Produces the currently running CI configuration. This also understands
   `--json`.

   `generate` and `current` are two steps run automatically when using `tc-admin
   diff`

* **`tc-admin <sub-command> --help`**

  Each command should have helpful text here.  These commands are defined in
  [tc-admin](https://github.com/taskcluster/tc-admin); see that tool for more
  information and to report bugs.

* **`ci-admin <sub-command> ..`**

  For backward compatibility, the `ci-admin` command behaves exactly the same
  as `tc-admin`.

# Development

To update dependencies, make changes to `requirements/*.in`, then install
`pip-compile-multi` from PyPI and run `pip-compile-multi -s -g
requirements/base.in`.
