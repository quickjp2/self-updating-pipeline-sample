# Self Updating Pipeline Sample

Automation is great, but wouldn't it be cool if you could automate the updates to your automation? With concourse, you can! This sample will walk you through setting up the initial pipeline, what resources are used, and how to update the pipeline to enable it to push an app.

## The app

The sample app we'll use is a base python flask app. The app only has one route, and not much in the way of a database, but it will suit our needs. You'll find the code [here](./application).

## The pipelines

Before we get started, I should mention some gotchas that could cause a problem with the self-update mechanism:

- resource / resource-type name changes
- changes that cause the concourse resource or pipeline definition resource to become unavailable

Due to some of these gotchas, we'll start with a very simple pipeline, with only the self-update job in it. This will enable us to troubleshoot any problems with the self-update job, and ensure that it is completely stable before adding additional jobs or resources.

### Key concepts

Before moving on, take a breif look at the [starting-pipeline.yml](./ci/pipelines/starting-pipeline.yaml). We'll go over what the pieces mean.

#### Concourse pipelines resource type

```yaml
resource_types:
- name: concourse-pipelines
  type: docker-image
  source:
    repository: concourse/concourse-pipeline-resource
```

Theoretically, you can update multiple pipelines at once. Thus, the resource type is called `concourse-pipelines`.

#### Concourse pipelines resource

```yaml
resources:
- name: <concourse>-<team>-pipelines
  type: concourse-pipelines
  source:
    target: https://<concourse_url>
    insecure: "false"
    teams:
    - name: <concourse_team_name>
      username: ((CONCOURSE_USERNAME))
      password: ((CONCOURSE_PASSWORD))
```

The major thing here is that you can push to multiple teams if desired. For this example, we will only specify one team, as we will only be updating one pipeline.

#### Pipeline definition

```yaml
- name: app-pipeline-source
  check_every: 12h
  webhook: <app_name>-pipeline
  type: git
  source:
    uri: <app_repo_url>
    branch: master
    private_key: ((GIT_PRIVATE_KEY))
    paths: [ci/*]
```

A standard git resource, with one addition: the `paths` param. This is where you can limit what changes will actually trigger a new resource version. In this case, we only want this resource to trigger a new version when we update concourse related files. Since best practices dictate that all concourse files should be in a `ci` folder, we can safely limit our resource to files in that folder.

Another point to make about the resource is the two check variables: `chech_every` and `webhook`. The `chech_every` param overrides the default check of 1m and this example sets it to 12h. The `webhook` param allows the upstream resource to make an api call to concourse, which will inform concourse to check the resource. Together, these variables are used to make the pipeline follow a push methodology, while ensuring that any changes not caught via the push are still caught during a periodic pull.

#### Updater job

```yaml
jobs:
- name: set-<app_name>-pipeline
  build_log_retention:
    builds: 10
  plan:
  - get: resource
    resource: app-pipeline-source
    trigger: true
  - put: <concourse>-<team>-pipelines
    params:
      pipelines:
      - name: <app_name>-pipeline
        team: <concourse_team_name>
        config_file: resource/ci/pipelines/pipeline.yml
        unpaused: true
```

This is where the magic happens. The get should be fairly self-explanitory, so let's focus on the put. Here, you could theoretically push multiple pipelines to multiple teams (the teams are specified in the resource), but for the app, we only need to push the one pipeline. Since our resource will only trigger when the `ci` folder is updated, we can safely say that the job will only run when needed.

Another thing to point out is the `build_log_retention` variable; this reduces the amount of build logs that concourse retains for the job. It is used to help keep the concourse installment's storage clean.

### Starting pipeline

To start with, you'll fill out the starting pipeline with the correct details: [starting-pipeline.yml](./ci/pipelines/starting-pipeline.yaml)

If you use a credential store, you can reference the credentials by using the `((var))` notation.
