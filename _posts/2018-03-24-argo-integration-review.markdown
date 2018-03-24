---
layout: post
title:  "Argo Kubernetes workflow engine integration review"
date:   2018-03-24 15:48:33 +0000
---

My team recently completed a project that integrated a Kubernetes workflow
engine, [Argo](https://github.com/argoproj/argo). I've tried to summarise our
usage at a high-level and record the various issues we encountered that are
worth revisiting as the project evolves.

We started using Argo in the v2.0.0-alpha3 timeframe and have shipped using
Argo 2.0.0.

We implemented:

* A service owning the domain model and storing the status and outputs of
  tasks.
* A workflow service that submits workflows, choosing from available workflows
  based on the domain model and setting global parameters based on user
  configuration. The service then watches the workflow and updates the domain
  model service.

The workflows use global parameters, S3 for input artifacts and make heavy use
of artifact passing. Otherwise we just use the core container / steps template
features.

Overall the experience was very positive, particularly considering the early
stage of the Argo project. We went looking for Kubernetes CI/workflow tooling,
as it seemed that newer projects building directly on Kubernetes could have a
significant benefit in reduced complexity. It was encouraging to find Argo and
Brigade in that space.

The scope of Argo made it easy to understand how it could be embedded in our
application. I really appreciated the separation between Argo and Argo CI. We
needed a domain-specific application model, rather than trying to fit with an
existing structure.

The declarative nature of the configuration and the minimal effort with which
we got a multi-step workflow with artifact passing working were significant
factors in selecting Argo.


## Issues to revisit

- We wrote a straightforward Java model of the Workflow CRD for use with the
  Java Fabric8 Kubernetes client. This predated the availability of the OpenAPI
  definitions, and I hope to revisit this to make use of them (though they seem
  to be missing the `status` part of the CRD). A well documented CRD model
  would have helped significantly in communicating how Argo worked within the
  team. I often found myself referring to the Go type definitions.

- The ability to add metadata to steps (or DAG tasks) that would appear in the
  status node tree would be helpful. I'll try to propose something specific.
  At the moment, we're transforming step names to unique IDs to let us
  trivially map back step results to our identifiers. This works fine, but can
  be unhelpful when debugging. It might be possible to extend the work done on
  [PR 789](https://github.com/argoproj/argo/pull/798).

- Testing workflows with `argo submit` would benefit from better CLI
  support for passing input artifacts and getting output artifacts
  ([issue 695](https://github.com/argoproj/argo/issues/695), [issue
  524](https://github.com/argoproj/argo/issues/524)).

- `argo install` was great for getting started, but it has led to Argo being
  managed separately from our Helm-based deployment process. We came across
  `argo-helm` a little late to switch to it. Having Helm as a first class
  option in the getting started docs might help adoption.

- We've ended up with a separate cluster for our Argo deployment vs other
  services, mostly because we couldn't resource limit Argo within a namespace
  without it failing the workflows at submission time (whereas, cluster-wide
  we'll just have temporarily unschedulable pods). Solutions in this area all
  look non-trivial, but we may have the opportunity to investigate further
  depending on the usage of the project. There's been helpful discussion from
  the Argo team on issues (e.g. [issue
  740](https://github.com/argoproj/argo/issues/740)).

- Support for periodic cleanup of completed workflows, potentially also
  including their S3 artifacts, would have saved some effort and might be a
  good fit for the controller. For workflows scheduled via our API, we ended up
  using a Kubernetes CrobJob to delete them, but we still ended up manually
  cleaning up workflows and S3 data from ad hoc Argo use. Some kind of clean up
  seems likely to be essential for all users (at least of the Kubernetes
  resources).

- It would be helpful to improve the accuracy of the "in-progress" status
  [issue 525](https://github.com/argoproj/argo/issues/525) covers this, though
  it would presumably be possible to work around this by also watching pods.

## Possible future enhancements / rethinks

- We plan to switch to DAG-based workflows as soon as 2.1.0 ships. It proved
  non-trivial to author workflows in order to ensure maximum parallelism with
  the step template syntax. We've tried the DAG syntax in 2.1.0-alpha1 and it
  will simplify workflow authoring considerably.

- Future work will probably require some ability to configure the workflows we
  run beyond simple parameterisation, by enabling/disabling steps or related
  groups of steps based on user choices. I think this will end up bespoke, but
  Argo features such as templates with optional input artifacts could help
  build this ([issue 805](https://github.com/argoproj/argo/issues/805)). The
  DAG model should be easier to manipulate in this way than steps templates.

- We used the built-in artifact passing, and synced the artifacts to separate
  permanent storage as the workflow completed. An alternative (but more
  ambitious) approach would have been to look at making Argo's artifact passing
  pluggable, and directly use our storage APIs. Initially the HTTP option
  seemed tempting here, but it didn't support upload or the tar.gz handling.
