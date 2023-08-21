---
title: "Clearing the complexity : From Kubernetes to GCP Cloud Run"
date: 2023-08-21T14:36:17+01:00
---
I’m working in an IT service company, where I’ve been introduced to a new project last year.
The stack is not so boring, every services use Typescript with Hasura as a GraphQL API and PostgreSQL as a PaaS on GCP.

All of the services were deployed across two environments on two different GKE clusters and a dedicated PostgreSQL instance.

These clusters had each two nodes. With the costs of the Kubernetes Engine, Compute Engine (with a total of 4 VMs where the nodes are installed), and Cloud SQL (2 PostgreSQL instances). The monthly billing was averaging 600 euros.

![Before](/images/cr_before.png#center)

This infrastructure cost is huge for an application serving no more than 100 users on working days. But this is not the real problem.

In an IT service company, it can be hard for junior developers to understand how their code is running on a GKE cluster. So, they have problems understanding how to read logs and understand networking concepts inside a pod. And the worst is the fear created by this because they have a fear to operate on the [Kustomize](https://kustomize.io/) files (e.g. inserting a new secret inside a service). So the team needed to get a specific “DevOps” engineer (who does not know how the app works of course) to get the work done.

After working several months on the projects, I told my manager that we need to entirely change our infrastructure because of the previous problems.

As a beginner in the Cloud ecosystem, I learned about GCP Cloud Run on Hacker News and wanted to do a proof of concept to analyze if it's a fit.

## The pros and cons

Our services only work through HTTP (except the database connection) and we didn’t use a specific mechanism from Kubernetes that is not available on Cloud Run.

Furthermore, the actual clusters were not scaling down to 0, but a Cloud Run service can simply do it when there are no requests for 15 minutes (which is greatly reducing the infrastructure costs during the nights and the week-end).

Estimating the future Cloud Run bills of the two environments was impossible for me hence the Cloud calculator is available on GCP.

## Migrating and getting an estimation

The manager accepted my wish to do the proof of concept. I started to use Terraform to create IAM / Cloud Run / networking resources onto GCP.

I chose to stick with the building phase on Gitab CI with [kaniko](https://github.com/GoogleContainerTools/kaniko). Hence, the Cloud Build from GCP seems to work fine for building/caching and pushing onto the Artifact Registry. I don’t want to fall into the pitfall of a Cloud Provider and stay as much as possible Cloud Agnostic.

Unfortunately, it’s not possible to deploy Docker images from our Gitlab registry, they must be stored in the Google Artifact Registry which has its fees. Surprisingly, the Artifact Registry doesn’t publicly support an image cleanup policy so we developed a small bash script to do the job through a scheduled pipeline.

Once the domain mapping and the secrets migration are done, I migrate a service from the test environment.

It was working like a charm and now, the deployment is instantaneous, the control of every revision is really clear and can be manipulated by every developer of the team. So, in case of emergency rollback, we are much more in control than with the former Kubernetes setup.

The proof of concept was accepted and I continued to migrate the test environment service by service. Once it was done, an estimation of the future bills was possible.

The only flaw of Cloud Run I observed for my use case was the random delay of the internal configuration of the ingress to serve our service on our domain. (from 5 minutes to 1 hour). So when we decided to migrate our production, we suffered from one hour of downtime.

Today, the daily average GCP billing has been cut by 3 and every developer from the team can operate on the infrastructure.

![After](/images/cr_after.png#center)
