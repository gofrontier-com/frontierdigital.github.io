---
title: "Azure DevOps Pipelines with multiple schedules"
date: 2022-12-04T00:00:00+00:00
publishDate: 2022-12-04T00:00:00+00:00
image: "/images/pipelines.jpg"
tags: ['ADO', 'pipelines', 'automation']
comments: false
draft: true
type: post
category: blog post
author: Neil Cowlin
authorImage: "/images/neil.jpg"
---
# Azure DevOps Pipelines with multiple schedules

## Context

A client is aware that significant cost savings could be realised by a scheduled manipulation of some of their deployed resources and services.  Their working week leads to predictable hot and cold periods in the utilisation of their build services and pre-production environments in which scaling up and down could take place.

The resources in question don't have any other suitable means to scale based on demand.

The client wants an Azure DevOps native solution, and everything is deployed by ADO pipelines.

## The goal

I would like to provide a single pipeline, which executes on a defined schedule, and applies some settings appropriate for that time. Sort of like an old programmable thermostat.

Simplistically, it's likely to be "run a scale up" or "run a scale down".

ADO provides scheduled triggers, defined using a cron syntax. You can set a pipeline to run whenever you want. You can set multiple cron schedules for a single job.

I'm picturing a pipeline something like this:

```yaml
# Two trigger definitions
schedules:
  - cron: "0 18 * * Mon-Fri"
    displayName: Every weekday at 1800
    branches:
      include:
        - master
    always: true

  - cron: "0 8 * * Mon-Fri"
    displayName: Every weekday at 0800
    branches:
      include:
        - master
    always: true

stages:
  - stage: Run_task
    jobs:
      - job: Task
        pool: hubAgents
        steps:
          - bash: # do different stuff depending on which cron schedule triggered us
```

## ADO Limitations

However, there’s a painful limitation: you cannot - within the YAML schema - set parameters or variables per trigger.  Moreover, a running pipeline doesn’t "know" which particular cron schedule triggered it.

ADO exposes many [built in variables](https://learn.microsoft.com/en-us/azure/devops/pipelines/build/variables?view=azure-devops&tabs=yaml#agent-variables-devops-services), but it strangely seems that inspecting the trigger is one of them.  

You could create multiple pipelines - one for each schedule - with some hardcoded set of arguments to pass to your pipeline payload template, but that creates duplication, and will get unmanageable quickly.  I am also working in a [Frontier Digital](https://frontierdigital.net/) ADO framework where GitOps rules all, and there’s automation both upstream and downstream creating and consuming the config - even *this* pipeline will be programmatically created!

## REST API to the rescue

There is however the trusty ADO REST API. It has all the trigger information - what we need is for our pipeline to query the API for information about itself as it's running.

We can see in [the docs](https://learn.microsoft.com/en-us/rest/api/azure/devops/build/builds/get?view=azure-devops-rest-6.1#build) that the `builds/build` endpoint returns an object containing what we need:

Here's an example of the API response from the endpoints that we're interested in

```json
{
  // ...
  "triggerInfo": {
    "scheduleName": "Every weekday at 1800"
  },
  // ...
}
```

Where `scheduleName` contains the `displayName` for the triggering schedule.

This can be done with a relatively simple `bash` task using `curl`, piping the returned JSON into `jq` to allow us to pull out what we’re after and set a variable.

This task is completely portable with no parameters needed as all settings can be pulled from built in variables!

### The task

```yaml
steps:
  - bash: |
      set -euo pipefail
      schedule_name=$( \
        curl \
          -H "Content-Type: application/json" \
          -s \
          -u ":$(System.AccessToken)" `# authenticate with Access Token` \
          "$(System.CollectionUri)/$(System.TeamProject)/_apis/build/builds/$(Build.BuildId)?api-version=7.0" |
        jq --raw-output '.triggerInfo.scheduleName' \
      )
      echo "##vso[task.setvariable variable=scheduleName]$schedule_name"
      echo "##vso[task.setvariable variable=scheduleName;isOutput=true]$schedule_name"
    name: getScheduleName
```

#### Building up the API endpoint

- `System.CollectionUri`: `https://dev.azure.com/my-org/`
- `System.TeamProject`: `my-project`
- `Build.BuildId`: The Id for *this* running pipeline

#### jq options

- `--raw-output`: removes quotes around the returned value

The task sets a variable `$schedule_name` (and an output variable too, for multi stage goodness), which can be passed on to whatever the main payload of the pipeline is.

In my situation, I use it to filter a configuration file.  The payload itself doesn't need to know if it's scaling up or down, it's just applying whatever the filtered configuration is!

Unfortunately this variable is obviously only available at run time, so can’t be used as a template variable for ADO native template conditions etc. You have to make some switching within your main payload.

### Non-scheduled pipeline runs

Manual runs of the pipeline won't have a `scheduleName` set.  The returned object looks like this:

```json
{
  // ...
  "triggerInfo": {},
  // ...
}
```

In this instance, our task returns a string with the literal value `null`.  We can handle this special case in our code.

## Summary

It would be great if ADO just exposed this information for us, but at least this way we can work around it and have one scheduled pipeline that handles all the cases.
