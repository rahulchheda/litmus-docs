---
id: version-1.2.0-admin-mode
title: Administrator Mode
sidebar_label: Administrator Mode
original_id: admin-mode
---
------

###  What is Adminstator Mode? 

Admin mode is one of the ways the chaos orchestration is set up in Litmus, wherein all chaos resources (i.e., install time resources like the operator, chaosexperiment CRs, chaosServiceAccount/rbac and runtime resources like chaosengine, chaos-runner, experiment jobs & chaosresults) are setup in a single admin namespace (typically, litmus). In other words, centralized administration of chaos. 
This feature is aimed at making the SRE/Cluster Admins life easier by doing away with setting up chaos pre-requisites on a per namespace basis (which may be more relevant in an autonomous/self-service cluster sharing model in dev environments).
This mode typically needs a "wider" & "stronger" ClusterRole, albeit one that is still just a superset of the individual experiment permissions. In this mode, the applications in their respective namespaces are subjected to chaos while the chaos job runs elsewhere, i.e., admin namespace. 

### How to use Adminstator Mode?

In order to use Admin Mode, you just have to create a ServiceAccount in the *admin* or so called *chaos* namespace (`litmus` itself can be used), which is tied to a ClusterRole that has the permissions to perform operations on Kubernetes resources involved in the selected experiments across namespaces.
Use provide this ServiceAccount in ChaosEngine's `.spec.chaosServiceAccount`. 

### Example

#### Prepare RBAC Manifest 

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: litmus-admin
  namespace: litmus
  labels:
    name: litmus-admin
---
# Source: openebs/templates/clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: litmus-admin-clusterrole
  labels:
    name: litmus-admin-clusterrole
rules:
- apiGroups: ["","apps","batch","extensions","litmuschaos.io","openebs.io","storage.k8s.io"]
  resources: ["chaosengines","chaosexperiments","chaosresults","configmaps","cstorpools","cstorvolumereplicas","events","jobs","persistentvolumeclaims","persistentvolumes","pods","pods/exec","pods/log","secrets","storageclasses",chaosengines","chaosexperiments","chaosresults","configmaps","cstorpools","cstorvolumereplicas","daemonsets","deployments","events","jobs","persistentvolumeclaims","persistentvolumes","pods","pods/eviction","pods/exec","pods/log","replicasets","secrets","services","statefulsets","storageclasses"]
  verbs: ["create","delete","get","list","patch","update"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get","list","patch"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: litmus-admin-clusterrolebinding
  labels:
    name: litmus-admin-clusterrolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: litmus-admin-clusterrole
subjects:
- kind: ServiceAccount
  name: litmus-admin
  namespace: litmus
```

#### Prepare ChaosEngine 

```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: nginx-chaos
  namespace: litmus #Chaos Resources Namespace
spec:
  appinfo:
    appns: 'default' #Application Namespace
    applabel: 'app=nginx'
    appkind: 'deployment'
  # It can be true/false
  annotationCheck: 'true'
  # It can be active/stop
  engineState: 'active'
  #ex. values: ns1:name=percona,ns2:run=nginx
  auxiliaryAppInfo: ''
  chaosServiceAccount: litmus-admin
  monitoring: false
  # It can be delete/retain
  jobCleanUpPolicy: 'delete'
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            # set chaos duration (in sec) as desired
            - name: TOTAL_CHAOS_DURATION
              value: '30'

            # set chaos interval (in sec) as desired
            - name: CHAOS_INTERVAL
              value: '10'
              
            # pod failures without '--force' & default terminationGracePeriodSeconds
            - name: FORCE
              value: 'false'
```

### Create the ChaosEngine Resource

- Create the ChaosEngine manifest prepared in the previous step to trigger the Chaos.

  `kubectl apply -f chaosengine.yml`

### Watch Chaos Engine 

- Describe Chaos Engine for chaos steps.

  `kubectl describe chaosengine nginx-chaos -n litmus`

### Watch Chaos progress

- View pod terminations & recovery by setting up a watch on the pods in the application namespace

  `watch -n 1 kubectl get pods -n default`

### Check Chaos Experiment Result

- Check whether the application is resilient to the pod failure, once the experiment (job) is completed. The ChaosResult resource name is derived like this: `<ChaosEngine-Name>-<ChaosExperiment-Name>`.

  `kubectl describe chaosresult nginx-chaos-pod-delete -n litmus`