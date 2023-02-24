---
title: "Autoscaling Kuberenetes workloads with custom schedules"
date: 2023-02-22T00:00:00+00:00
publishDate: 2023-02-22T00:00:00+00:00
image: "/images/containers.jpg"
tags: ['ADO', 'pipelines', 'automation', 'kubernetes', 'FinOps', 'OpenShift']
comments: false
type: post
category: blog post
author: Anthony Gibbons
authorImage: "/images/anthony.jpg"
---

# Autoscaling Kuberenetes workloads with custom schedules

## Context

Would you like to know how we cut annual cloud costs by over 45% for one of our major financial services clients? Read on!

Following on from a previous blog post where we discovered how to make use of [custom schedules](https://frontierdigital.net/blog/azure-devops-pipelines-with-multiple-schedules/) in ADO pipelines, we decided to use the same mechanism as part of our FinOps practices for one of our major clients. 

This particular client has a large development studio with a route to live consisting of multiple test environments prior to production. Each environment has at least 1 RedHat OpenShift cluster running a complex variety of Kubernetes workloads. 

## The challenge

The challenge was to reduce the compute costs for worker nodes across the non-production whilst maintaining a high degree of environment availability for development and testing teams. We saw an opportunity to take advantage of the OpenShift/Kubernetes machine autoscaler by scaling down workloads out of hours.
 
#### Reduce workloads > reduce OpenShift worker nodes > reduce cloud compute costs

Whilst it was understood that *some* workloads could be scaled down out of hours, not *all* workloads could be scaled down. Some more critical or time sensitive workloads were to remain running 24/7. Furthermore, there would be different requirements per project and per environment and the development studio would require final control over the autoscaling configuration of each environment and the projects within.

## The solution

At Frontier Digital, we have always maintained an `everything as code` approach to all of the platforms we develop. With this mindset in place, we quickly realised that the autoscaling configuration each environment would have to be maintained in source control. We developed an easily consumable config mechanism in YAML. One YAML file per environment with the OpenShift projects defined as follows:

```yaml
projects:   
  - name: core-services
    exclusions:
      - account-batch-service
      - account-registration-service
  - name: payment-services
    exclusions:
      - payment-schedule-service
      - payment-transfer-service
      - transfer-automation-service
```

First of all, this is an `opt-in` approach to autoscaling. If the project is not specifically called out in configuration, it is ignored by the autoscaler pipeline. Here we can see that we have opted the `core-services` and `payment-services` projects into autoscaling. However, each of thes projects has a number of `exclusions`. Exclusions are called out in config to tell the autoscaler to scale every deployment in the project *except* those called out in the exclusions array. This allows developers and testers to be in full control of which workloads are subject to autoscaling. 

## In practice

Taking the example of the `core-services` project, we can see that there are a number of different deployments with differing replica counts. 

![Payment Services](/images/ocp-1.png)

The configuration for this test environment calls out `core-services` as a candidate project for the autoscaler. However, the following services are to be excluded:

* `account-batch-service`
* `account-registration-service`

When we run the autoscaler pipeline, the config is consumed from the repository: 

![Pipeline Logs](/images/ocp-2.png)

Finally, when the pipeline has finished executing, we can see that the deployments have been scaled to zero, leaving the excluded deployments alone:

![Payment Sevices](/images/ocp-3.png)

## Keeping track

As we have previously noted, we cannot rely on a standard replica count across our different deployments. Many deployments have differing replica counts. Our goals are to reduce cost whilst maintaining stability for our development and testing teams across the studio. When the autoscaler scales deployments back up, scaling to a pod count that is too high or too low would endager one or both of these goals. In order to keep track of the previous state of the deployments, the autoscaler annotates the deployment upon scaledown:

![Annotations](/images/ocp-4.png)

Information is captured such as the original replica count to aid the scale up action and some additional metadata such as the URL of the ADO pipeline run that executed the scale down. 

## Scheduling

All this is useful but there has to be some mechanism to control each environment and the different scheduling requirements. Going back to the [custom schedule trick](https://frontierdigital.net/blog/azure-devops-pipelines-with-multiple-schedules/) we are able to define many different schedules in our ADO pipeline:

```yaml
schedules:
  - cron: "0 20 * * *"
    displayName: FAT shutdown
    branches:
      include:
        - master
    always: true

  - cron: "0 8 * * *"
    displayName: FAT restore
    branches:
      include:
        - master
    always: true

  - cron: "0 20 * * Fri"
    displayName: SIT shutdown
    branches:
      include:
        - master
    always: true

  - cron: "0 3 * * Mon"
    displayName: SIT restore
    branches:
      include:
        - master
    always: true

  - cron: "0 1 * * Sat"
    displayName: UAT shutdown
    branches:
      include:
        - master
    always: true

  - cron: "0 20 * * Sun"
    displayName: UAT restore
    branches:
      include:
        - master
    always: true
```

Since we can query the ADO API to determine which schedule kicked the pipeline off, the pipeline can make use of this information in a Pre Flight step to determine which action to take:

![Pipeline](/images/ocp-5.png)

## Summary

This exercise has resulted in a huge cost saving for the development studio. They are no longer burning money on expensive compute when it is simply not in use. This, combined with a number of FinOps practices has allowed us to cut their annual cloud costs by over 45%. Get in touch if you'd like to find out more.