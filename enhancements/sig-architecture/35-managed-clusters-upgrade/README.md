# Managed Cluster upgrade (MCU) enhancement

## Release Sign off Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

Currently OCM does not provide out of the box APIs to upgrade a group of managed clusters and monitoring the upgrade process.
This enhancement propose Managed Cluster Upgrade (MCU) API to allow end user be able to identify group of managed clusters and define the target ocp version for upgrade.
Moreover MCU API will provides the ability to execute upgrade process in batches and select canary clusters for the upgrade process as well as upgrade the installed OLM operators on the selected clusters.

## Motivation

1. Cluster upgrade usually required a sequence of operations such as; 1-Pre upgrade action 2- cluster upgrade 3- post upgrade action. As OCM user I want to apply workload backup action then after backup action succeed, perform cluster upgrade then after cluster upgrade succeed apply post upgrade health check.  
1. OCP cluster upgrade is a progressive process takes at least 40min to 2h. As OCM user I want to perform clusters upgrade and be able to monitor clusters upgrade state.
1. It make sense to define a descriptive API to define the upgrade process for group of managed clusters and monitor the upgrade process for each cluster.
 
### Goals

1. Provide MCU APIs to facilitate the managed clusters upgrade process and monitor the upgrade state per managed cluster.

### Non-Goals


1. MCU APIs does not perform the upgrade in the managed cluster. MCU delivers the upgrade configurations to the managed cluster and reports the upgrade state.

1. MCU APIs does not roll back the upgrade in case of failure.


## Proposal

### User Stories

1. As end User, I want to perform a pre-upgrade action like backup workload data.
1. As end user, I want to upgrade group of managed clusters and be able to monitor the upgrade process per cluster.
1. As end user, I want to approve OLM operator upgrade for a group of managed clusters.
1. As end user, I want to upgrade group of managed clusters in batches and identify canary clusters for the upgrade process.
1. As end user, I want to perform a post-upgrade action like workload health checks. 

### APIs

#### ManagedClustersUpgrade (MCU)

The example below (mcu-upgrade) shows the spec and status of the MCU CR. 

```
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ManagedClustersUpgrade
metadata:
  name: mcu-upgrades
  namespace: default
spec:
  # optional users can refer to a list of pre-defined placements to select the managed clusters
  placement:
  - name: dev-clusters
    namespace: dev
  # optional user can define the match expression to select group of clusters  
  clusterSelector: 
    clusterSets:
    - dev
    matchExpressions:
      - key: name
        operator: In
        values:
          - dev1
          - dev2
          - dev3
          - dev4
  # user define the cluster version attributes that will be applied to the selected clusters
  clusterVersion:
    channel: stable-4.12
    version: 4.12.0
    upstream: ""
    image: ""
  # User can choose to approve all the installed OLM upgrade OR identify which OLM to upgrade
  ocpOperators: 
    approveAllUpgrades: false/true
    include:
      - name: openshift-gitops-operator
        namespace: openshift-operators
    exclude:
      - name: local-storage-operator
        namespace: openshift-operators
  # User can select canary clusters for the upgrade process and define the number of concurrency clusters to upgrade.
  # User can set the timeout for the upgrade process.
  upgradeStrategy: 
    canaryClusters:
      clusterSelector:
        matchExpressions:
          - key: name
            operator: In
            values:
              - dev1
    clusterUpgradeTimeout: 2h
    maxConcurrency: 2
    operatorsUpgradeTimeout: 10m
  # User can define a manifestWork as pre-upgrade action
  preUpgradeAction:
    # same as manifestWork template in manifestWork APIs
    manifestWork:
       ...
  # User can define a manifestWork as post-upgrade action
  postUpgradeAction:
    # same as manifestWork template in manifestWork APIs
    manifestWork:
       ...    
status:
  preUpgradeActionSummary:
    total: 3
    applied: 3
    available: 2
    progressing: 1
    degraded: 0
  canaryClustersSummery:
    total: 1
    initialized: 1
    partialUpgrade: 0
    complete: 1
    failed: 0
    pending: 0
  clustersSummery:
    total: 4
    initialized: 3
    partialUpgrade: 2
    complete: 1
    failed: 0
    pending: 0
  postUpgradeActionSummary:
    total: 1
    applied: 1
    available: 1
    progressing: 0
    degraded: 0
  conditions:
    - lastTransitionTime: '2022-06-02T15:20:08Z'
      message: ManagedClusters upgrade select 3 clusters
      reason: ManagedClustersSelected
      status: 'True'
      type: Selected
    - lastTransitionTime: '2022-06-02T15:20:08Z'
      message: ManagedClusters upgrade applied
      reason: ManagedClustersUpgradeApplied
      status: 'True'
      type: Applied
    - lastTransitionTime: '2022-06-02T15:20:08Z'
      message: ManagedClusters upgrade InProgress
      reason: ManagedClustersUpgradeComplete
      status: 'True'
      type: InProgress
    - lastTransitionTime: '2022-06-02T15:20:38Z'
      message: ManagedClusters upgrade Complete
      reason: ManagedClustersUpgradeComplete
      status: 'False'
      type: Complete
```

The MCU spec has the following fields
1. `placement` refer to the placement CR that will be used to select the managed clusters.
1. `clusterSelector` define the placement fields to select group of clusters based on cluster sets and match expression or label.
1. `clusterVersion` define the required cluster version upgrade configurations.
1. `ocpOperators` define the ocp operators list to upgrade. 
1. `upgradeStrategy` define the select canary clusters, timeout for upgrade process and number of concurrent clusters to perform upgrade.
1. `preUpgradeAction` define a manifestWork to perform pre upgrade action.
1. `postUpgradeAction` define a manifestWork to perform post upgrade action.
  
The MCU status contains pre/post clusters action summery and canaryClusters/clusters upgrade summery.
  - **canary/clustersSummery**
    - **total** is the number of all selected clusters out of the defined placement. Note: for canary total is the number of selected canary clusters. 
    - **initialized** is the number of clusters that have upgrade initialized.
    - **partialUpgrade** is the number of clusters that have upgrade in progress.
    - **complete** is the number of clusters that have upgrade complete.
    - **failed** is the number of clusters that have upgrade failed.
    - **pending** is the number of clusters in pending state based on the defined maxConcurrency in the upgradeStrategy.
  - **pre/postUpgradeActionSummary** 
    - **total** is the number of selected clusters to apply preUpgrade actions reflected by the maxConcurrency in the upgradeStrategy. Note: for postUpgrade actions total is the number of completed upgrade clusters.
    - **applied** is the number of clusters that have pre/post action applied
    - **available** is the number of clusters that have pre/post action complete
    - **progressing** is the number of clusters that have pre/post action in progress
    - **degraded** is the number of clusters that have pre/post action failed
The MCU status condition list has the following conditions;
  - **Selected** condition state is true when the cluster Selector has identified at least one cluster to upgrade. 
  - **Applied** condition state is true when at least one cluster has the preUpgrade action or upgrade configuration applied to it.
  - **InProgress** condition state is true when at least one cluster upgrade is in partial state. 
  - **Complete** condition state is true when all the clusters upgrade is in complete state.
  - **Failed** condition state is true when at least one cluster upgrade is in failed state.

### Implementation Details/Notes/Constraints [optional]

#### Pre/Post upgrade action

MCU API will uses the manifestWork API defined in the open-cluster-management [Work-API](https://github.com/open-cluster-management-io/api/blob/main/work/v1/types.go#L30) to define the pre/post upgrade actions.
The MCU controller will creates a placeManifestWork to apply the pre/post upgrade actions and propagate the created placeManifestWork status to pre/postUpgradeClustersSummery. For each selected cluster the upgrade will not start unless its preUpgrade action is in available state. The post upgrade action will not be applied unless the cluster has upgrade complete state.

#### Cluster upgrade manifestWork

MCU API use a predefined placement CR to select the managed clusters that  will be upgraded. As well as, the clusterSelector field can be used to select group of clusters.  
The MCU controller will use OCM manifestWork APIs to deploy the cluster version upgrade configuration and apply operators upgrades.

The manifestWork CR example below (spoke1-cluster-upgrade) will be created for each cluster to apply the upgrade configurations.
The manifestWork->manifests contain the clusterVersion that is created based on MCU->spec->clusterVersion fields.
The manifestWork feedback Rules report back the upgrade version, state, verified and progress message. 

```
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  name: spoke1-cluster-upgrade
  namespace: spoke1
spec:
  deleteOption:
    propagationPolicy: SelectivelyOrphan
    selectivelyOrphans:
      orphaningRules:
        - group: ''
          name: version
          namespace: ''
          resource: ClusterVersion.config.openshift.io
  manifestConfigs:
    - feedbackRules:
        - jsonPaths:
            - name: version
              path: '.status.history[0].version'
            - name: state
              path: '.status.history[0].state'
            - name: verified
              path: '.status.history[0].verified'
            - name: message
              path: '.status.history[0].message'
          type: JSONPaths
      resourceIdentifier:
        group: config.openshift.io
        name: version
        namespace: ''
        resource: clusterversions
  workload:
    manifests:
      - apiVersion: config.openshift.io/v1
        kind: ClusterVersion
        metadata:
          name: version
        spec:
          clusterID: 94e170c1-7a3c-4ae1-a85d-8bd904ad783f
          channel: stable-4.12
          desiredUpdate:
            force: false
            image: ''
            version: 4.12.0
      - apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRoleBinding
        metadata:
          annotations:
            rbac.authorization.kubernetes.io/autoupdate: 'true'
          name: admin-ocm
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: ClusterRole
          name: cluster-admin
        subjects:
          - kind: ServiceAccount
            name: klusterlet-work-sa
            namespace: open-cluster-management-agent
```

#### Operators upgrade manifestWork

There are two mode of operations for OLM operator;
1. Automatic mode where the operator upgrade will happen automatically without require ocp admin approver.
1. Manual mode where the operator upgrade will not proceed until the OCP admin approve the Install Plan CR.

The MCU controller will only upgrade the manual mode of operations for the OLM operators. 
The MCU controller will create a manifestWork that capable of approves the installPlan CRs of the selected operators.

The manifestWork CR example below (spoke1-operators-upgrade) could be used to approve the subscription installPlans CRs in the selected managed clusters.
The manifestWork feedback rules report the job state which indicate the operators installPlans CR has been approved or failed.

```
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  name: spoke1-operators-upgrade
  namespace: spoke1
spec:
  deleteOption:
    propagationPolicy: Foreground
  manifestConfigs:
    - feedbackRules:
        - jsonPaths:
            - name: succeeded
              path: .status.succeeded
            - name: active
              path: .status.active
            - name: failed
              path: .status.failed              
          type: JSONPaths
      resourceIdentifier:
        group: batch
        name: installplan-approver
        namespace: installplan-approver
        resource: jobs
  workload:
    manifests:
      - apiVersion: v1
        kind: Namespace
        metadata:
          name: installplan-approver
      - apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRole
        metadata:
          name: installplan-approver
        rules:
          - apiGroups:
              - operators.coreos.com
            resources:
              - installplans
              - subscriptions
            verbs:
              - get
              - list
              - patch
      - apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRoleBinding
        metadata:
          name: installplan-approver
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: ClusterRole
          name: installplan-approver
        subjects:
          - kind: ServiceAccount
            name: installplan-approver
            namespace: installplan-approver
      - apiVersion: v1
        kind: ServiceAccount
        metadata:
          name: installplan-approver
          namespace: installplan-approver
      - apiVersion: batch/v1
        kind: Job
        metadata:
          name: installplan-approver
          namespace: installplan-approver
        spec:
          activeDeadlineSeconds: 690
          manualSelector: true
          selector:
            matchLabels:
              job-name: installplan-approver
          template:
            metadata:
              creationTimestamp: null
              labels:
                job-name: installplan-approver
            spec:
              containers:
                - command:
                    - /bin/bash
                    - '-c'
                    - >
                      echo "Continue approving operator installPlan for
                      ${WAITTIME}sec ."

                      end=$((SECONDS+$WAITTIME))

                      while [ $SECONDS -lt $end ]; do
                        echo "continue for " $SECONDS " - " $end
                        oc get subscriptions.operators.coreos.com -A
                        for subscription in $(oc get subscriptions.operators.coreos.com -A -o jsonpath='{range .items[*]}{.metadata.name}{","}{.metadata.namespace}{"\n"}')
                        do
                          if [ $subscription == "," ]; then
                            continue
                          fi

                          if [[ $INCLUDELIST != "" ]]; then
                            if [[ $INCLUDELIST != *$subscription* ]]; then
                              continue
                            else
                              echo "Subscription $subscription Included"
                            fi
                          fi

                          if [[ $EXCLUDELIST != "" ]]; then
                            if [[ $EXCLUDELIST == *$subscription* ]]; then
                              echo "Subscription $subscription Excluded"
                              continue
                            fi
                          fi

                          echo "Processing subscription '$subscription'"
                          n=$(echo $subscription | cut -f1 -d,)
                          ns=$(echo $subscription | cut -f2 -d,)

                          installplan=$(oc get subscriptions.operators.coreos.com -n ${ns} --field-selector metadata.name=${n} -o jsonpath='{.items[0].status.installPlanRef.name}')
                          installplanNs=$(oc get subscriptions.operators.coreos.com -n ${ns} --field-selector metadata.name=${n} -o jsonpath='{.items[0].status.installPlanRef.namespace}')
                          echo "Check installplan approved status"
                          oc get installplan $installplan -n $installplanNs -o jsonpath="{.spec.approved}"
                          if [ $(oc get installplan $installplan -n $installplanNs -o jsonpath="{.spec.approved}") == "false" ]; then
                            echo " - Approving Subscription $subscription with install plan $installplan"
                            oc patch installplan $installplan -n $installplanNs --type=json -p='[{"op":"replace","path": "/spec/approved", "value": true}]'
                          else
                            echo " - Install Plan '$installplan' already approved"
                          fi
                        done
                      done
                  env:
                    - name: EXCLUDELIST
                      value: ""
                    - name: INCLUDELIST
                      value: openshift-gitops-operator,openshift-operator
                    - name: WAITTIME
                      value: '600'
                  image: 'registry.redhat.io/openshift4/ose-cli:latest'
                  imagePullPolicy: IfNotPresent
                  name: installplan-approver
                  resources: {}
              dnsPolicy: ClusterFirst
              restartPolicy: OnFailure
```

### Sequence of operations;

1. The pre upgrade action must be in available state to proceed for cluster upgrade.
1. In case pre upgrade action is in degraded state for some clusters, other clusters should be able to perform cluster upgrade.
1. The placeManifestWork created for pre upgrade action will be deleted after all clusters have upgrade complete state.
1. The selected Canary Clusters must finish successfully the upgrade first in order to proceed with upgrading all clusters.
1. If Canary Clusters upgrade failure the upgrade for selected clusters must not proceed. 
1. The operator upgrade must be done after the cluster platform upgrade.
1. In case of placement Decision changed for the selected clusters while the upgrade is in progress;
 a) if the selected cluster is in Initialized or Partial upgrade state, the upgrade process must proceed.
 b) if the selected cluster is in NotStarted state, the upgrade process can be terminated.
1. The number of clusters that are in initialized or inProgress state must be equal or less than the maxConcurrency number defined under the upgrade strategy. 
1. When the cluster upgrade rich completed state the post upgrade action should be applied.
1. In case post upgrade action is in degraded state for a cluster, the cluster upgrade will be in failed state.
1. The placeManifestWork created for post upgrade action will be deleted after all clusters have upgrade complete state.
 
### Risks and Mitigation

* Security is based on the placement resources which bind to ManagedClusterSets and the namespace where the placement CR is created.

### Test Plan

- Unit tests will cover the controller fundamental functionalities.
- Unit tests will cover the new APIs.
- Integration tests will cover all user stories.
- And e2e tests will cover the following cases.
  - Create/updated/delete managedClusterupgrade.
  - Placement Decision changes while upgrade is happening


### Graduation Criteria
**Note:** *Section not required until targeted at a release.*

#### Tech Preview -> GA
**For non-optional features moving to GA, the graduation criteria must include
end to end tests.**

### Upgrade / Downgrade Strategy

MCU API will be supported on the target release of OCM (open cluster management) and it should not affect the OCM current APIs functionalities or back compatibility.   

## Alternatives

Current alternative does not satisfy the requirements as explained above in the motivation section

