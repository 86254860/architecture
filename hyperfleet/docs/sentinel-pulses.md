# Sentinel pulses, adapter behavior and API resource status

This document proposes changes to the current system design for how API and adapter behaves by introducing new API resource root conditions and detailing status reports by adapters and transitions.

Sentinel behaviour generates pulses when:

- User changes the cluster/nodepool spec
- Last status report from adapter has exceeded some TTL
  - For `status.phase==NonReady` clusters/nodepools, TTL=10sec
  - For `status.phase==Ready` clusters/nodepools, TTL=30min

Pulses publish messages to a queue and are distributed to adapters for them to update their status for the resource.

The adapters performs some actions in their "resources" phase in order to query the state and then reports to the HyperFleet API with at least 3 mandatory conditions:

- Applied: the work to be done has been started
- Available: The status of the resource for that adapter is successful
- Health: Additional info about the health of the probe

The actions performed in the "resources" phase:

- Should be idempotent
- It is a call to an API with a payload
  - k8s API to create k8s objects like CRDs or Job
  - Call Maestro API
  - Call other APIs

After the actions, the state is collected to report to the HyperFleet API

- The adapter work can be asynchronous
  - The API returns before the work has completed.
- In this case the adapter will report with condition Available=Unknown
- e.g. Some k8s resources have generation/observedGeneration the adapter can determine if the status of the object is meaningful or hasn't yet  reconcile

Scenarios for adapter receiving a message to process a resource:

- API State:
  - No Available condition exists for the adapter in the API
  - There is an Available condition with observed_generation < spec.generation
  - There is an Available condition with observed_generation == spec.generation
- Adapter task resource for current generation:
  - Does not exists
  - Exists
    - Is still in progress
    - Finished successful/failure

### Example for adapter task using k8s resources

Applying k8s resources should be an idempotent operation

- On receiving a message and passing the precondition, apply the manifests
- Optimize in preconditions, if condition Available last_updated_time is within the TTL
- Reports Available depending on job status and result:
  - If condition Complete!=True -> Unknown
  - Else suceeded == 1

#### Jobs

The "problem" with applying an already existing and completed job is that k8s will not spin up a new container for it. So, the adapter will get a "cached" status from the status results of the completed job.

Some approaches to solve this:

- Recreate the job in the resources phase (delete and create)
- Remove the job automatically using `ttlSecondsAfterFinished` with a value < 30m
  - This guarantees that the next reconcile event after 30m will not find the job
  - And will recreate it

#### CRDs

Applying a CRD or manifest (like namespace manifest) should be an idempotent operation

The adapter should detect if the work to be done by the adapter task is still in progress for the CRD. For example some k8s objects use `generation` and `status.observedGeneration` to differentiate.

#### API calls

If using an arbitrary API call in the adapter resource phase, the calling service must be also idempotent, and provide a way to determine work in progress, similar to the CRDs case.

### Some edge cases to clarify

1. In progress adapter task response
    1. cluster `Ready` at `gen=1`
    1. At 30m, event to refresh status for `gen=1`
    1. Adapter starts a new adapter task, so it should report `Applied=True`
        - What about `Available`?
        - If it reports `False`, cluster will go `NotReady`
        - So, it either:
          - Doesn't report (we need to add post-conditions to conditionally report)
          - Or reports `Available=Unknown`

2. Mixed generation for Available conditions
    1. cluster `Ready` at `gen=1`
    1. User updates spec, `gen=2`
    1. Only one/some adapters report `observed_generation=2` `Available=True`

- Is cluster still `Ready` or has become `NotReady` because of the mixed generations?
- If one adapter reports `Available=False` for `gen=2`
  - Cluster should go to `NotReady`
  - But can we consider it still Ready at `gen=1` ?

1. Cluster transitions to Ready but with mixed generations
    1. At one point a cluster is `Ready`
    1. Later, the first adapter reports `Available=false`
        - cluster transitions `Ready -> NotReady` at `gen=1`
    1. User updates `gen=2`
    1. 1st adapter reports `Available=True` `observed_generation=2`
       - Is cluster `Ready` or `NotReady` ?

### API behaviour

We need to disambiguate the meaning of `status.phase=Ready/NotReady`

- Currently it conveys:
  - **“new desired generation not reconciled”**
  - **“system broken”**
    - even though the old generation might still be operating fine.
- We can use the proposal of having 2 conditions at the resource level
  - `Available`: System is running
    - It also contains an `observed_generation` property
    - All adapters report `Available==True` at `observed_generation`

  - `Ready`: System is running at the latest spec generation

The GCP team proposed also to have the logic behind the `Available/Ready` configurable using some type of expressions, eg something like:

```
available=self.items.all(i, 
  i.observed_generation == self.items[0].observed_generation && 
  i.conditions.exists(c, c.type == 'Available' && c.status == 'True')
)
ready= available && self.items.all(i, i.observed_generation == generationN)
```

Some other rules for the API:

1. API should only accept conditions statuses for same or increased condition.observed_generation
    - An adapter update can not replace data from a newer generation
    - e.g. If validation adapter is at gen=2, API only accepts reports of gen>=2
1. API should discard conditions with `Available=Unknown`
    - It can still store in some status log table/file for tracing
    - We can think of optimizing by having a post-condition that discards adapter report
    - It also doesn't update `last_updated_time` for the adapter
1. Available only transitions to True
    - If all the adapters are at the same generation report Available=True
    - Could be that the adapter's reports observed_generation < generation
      - This can happen when updating quickly a spec
      - API will be getting "older" adapter statuses first
      - It can be "Available"" for that generation, but needs reconciliation
      - Eventually it will get newer responses
1. Available transitions to False
    - If any adapter of the `observed_generation` in the condition reports `Available=False`
    - But not for adapters reporting `Available=false` at other `observed_generation`
    - This keeps `Available=true` for `observed_generation` while `Ready=False`
      - Meaning, that the last known generation is still marked available
1. Ready transitions to False for every user spec change
      - Since it will increase generation
      - And all adapters being async have observedGeneration<generation
