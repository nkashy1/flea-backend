# The Flea Circus

This document describes the backend components and architecture for the Flea federated learning
framework.

## Objective

The Flea backend is intended to support the training of machine learning models in a federated
context, where the model is trained independently in multiple client environments that are not
accessible to the environment in which training tasks are being generated (typically, this
is the local environment of a data scientist).

## Requirements

As per the objective above, there are two primary types of users of a Flea backend:

+ *Task generators* - For example, data scientists who register task specifications against the
backend that clients can fulfil (by updating a model according to those specifications and reporting
the results).

+ *Flea clients* - These are devices and environments which are capable of running model update
operations, which are also configured to query the Flea backend for available tasks. If there are
tasks available for them to work on, they can complete the updates as per the task specifications
and write the results to a URL specified when they subscribe to a task.

Given that the models whose training is coordinated by a Flea backend represent potentially
sensitive intellectual property, we also recognize:

+ *Model owners* - These are people or organizations who may wish to restrict access to task
specifications or who may wish to prevent certain clients from subscribing to particular Flea tasks.
For example, an organization may wish to limit clients to one subscription per Flea task so as to
prevent malicious clients from spamming spurious updates for a particular task.

Finally, as the backend needs to be deployed and maintained in production, we also recogize:

+ *Maintainers* - These are people or teams responsible for setting up deployments of the Flea
backend and for maintaining it under load. For example, the doc.ai infrastructure team.

We go through the requirements for each type of user.

### Task generators

Should be able to:

1. **Register tasks:** Specify tasks with a 4-tuple
`(modelId, hyperparametersId, checkpointIds, deadline)` along with a *task bundle*, where `modelId`,
`hyperparametersId`, and `checkpointIds` refer to
[`tensorio-models`](https://github.com/doc-ai/tensorio-models) resources, `deadline` is a UTC date
and time object specified at the level of seconds, and the *task bundle* is a single file containing
all the information required to run a task (assumed to be comprehensible to Flea clients). Each task
dictates updates with parameters specified by the task bundle to any checkpoint for the given
hyperparameters for the given model which is included in the `checkpointIds` list.

1. **Close tasks:** Designate any available task that they are authorized to modify as closed,
thereby preventing any client from commiting to that task or registering updates for that task
specification.

1. **Aggregate submissions for a task:** Once a task has been closed, the generator of that task
should be able to trigger an aggregation job over all updates registered for that task.

1. **Evaluate aggregated submissions for a task:** Once an aggregation has been performed over a
given task, the task generator should be able to trigger an evaluation job for the aggregated
checkpoint, to determine whether or not the aggregation exhibits improved performance over the
checkpoints in the range for the given task.

### Flea clients

Should be able to:

1. **Discover tasks:** Find all open tasks for a given triple of
`(modelId, hyperparametersId, checkpointId)`, where `modelId`, `hyperparametersId`, and
`checkpointId` are signifiers for [`tensorio-models`](https://github.com/doc-ai/tensorio-models)
resources of the corresponding type and a task is considered to be open for a given triple if its
`deadline` has not yet passed (according to the clock on the Flea API server) and the given
`checkpointId` lies in the `checkpointIds` list for the task.

1. **Commit to a task:** Request that they be allowed to submit an update corresponding to the
specification of a given task and, if their request is granted, receive a URL to which they can
upload the update.

1. **Register updates for a task:** Upload updates they have computed for a given task specification
provided that they have previously commited to that task and that the deadline for that task has
not passed.

### Model owners

Should be able to:

1. **Restrict access to models:** Specify which API clients can view and commit to
tasks for which models (based on a `clientId` token passed to the API by the client).

1. **View task commitment and update registration statistics**

### Maintainers

Should be able to:

1. **Deploy a Flea backend**

1. **Update an already deployed Flea backend**

1. **Get the runtime status of a backend they have deployed** (this includes metrics)

## Components

1. Flea API

1. Flea aggregators

1. Flea evaluators

### Flea API
Responsible for all but the aggregation and evaluation requirements for task generators. Will expose
`gRPC` as well as `REST` endpoints following the model of
[`tensorio-models`](https://github.com/doc-ai/tensorio-models). The API will be implemented in
[`go`](https://golang.org). The `REST` API will be generated
from the `gRPC` API using [`grpc-gateway`](https://github.com/grpc-ecosystem/grpc-gateway).

### Flea aggregators and Flea evaluators

Flea aggregators are responsible for aggregation of updates submitted for a given task.

Flea evaluators are responsible for the evluation of aggregations produced by the aggregators.

These are intended to be scripts which can be either triggered manually by data scientists or which
can be triggered automatically (for example, by a `cron` job). Moreover, since they are intended to
work with models, hyperparameters, and checkpoints created by the data scientists who generated the
tasks they correspond to, there is a certain degree of freedom in how they are implemented. It is
expected that most of these scripts will be implemented in Python, but the only requirement is that
the code necessary to run an aggregator or an evaluator be placed in a uniquely named subdirectory
of the `aggregators/` and `evaluators/` directories in root of this repository along with a
`Dockerfile` that can be used to eventually run the script in a container.

The aggregator and evaluator containers will need to be able to access object storage. Initially,
we plan to support only Google Cloud Storage, meaning that the scripts will simply need to accept
a `GOOGLE_APPLICATION_CREDENTIALS` environment variable (as described
[here](https://cloud.google.com/docs/authentication/getting-started)). Generalization of this
pattern is currently out of scope of this design doc, and this directory structure is unassuming and
unconstrained enough that we are not painting ourselves into any corners as far as configuration for
access to arbitrary object storage backends is concerned.

Any deployment of a Flea backend will specify as a deployment parameter which aggregator or
evaluator to use by specifying subdirectories of the `aggregator/` and `evaluator/` directories
respectively.
