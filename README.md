# SRVKP-3136

## Issue
After removing a namespace resource with pruner [annotations](https://docs.openshift.com/container-platform/4.11/cicd/pipelines/automatic-pruning-taskrun-pipelinerun.html#annotations-for-automatic-pruning-taskruns-pipelineruns_automatic-pruning-taskrun-pipelinerun), the operator breaks down and for any new namespace with pruner annotations, the related CronJob is not created.

## How-To-Reproduce
- Create the `namespace-one` [Namespace](https://raw.githubusercontent.com/gmeghnag/SRVKP-3136/main/namespace-one/namespace-one.yaml), with the required [annotations](https://docs.openshift.com/container-platform/4.11/cicd/pipelines/automatic-pruning-taskrun-pipelinerun.html#annotations-for-automatic-pruning-taskruns-pipelineruns_automatic-pruning-taskrun-pipelinerun) for tekton resources automatic pruning, along with a simple `Task` and  `Pipeline`, by executing:

  ```
  git clone https://github.com/gmeghnag/SRVKP-3136
  cd SRVKP-3136
  oc apply -f namespace-one
  ```
  Verify that the pruner `CronJob` has been created for the namespace `namespace-one`:
  ```
  oc get cj -n openshift-pipelines -o json | jq '.items[0] | select(.metadata.labels."tektonconfig.operator.tekton.dev/pruner.ns"=="namespace-one").metadata.name'
  ```
- Then, delete the namespace `namespace-one`, then you will see that the associated `CronJob` resource in the namespace `openshift-pipelines` will not be deleted:
  ```
  oc delete ns namespace-one
  ```

- Now create another namespace `namespace-two` identical to `namespace-one`, you will see that the `CronJob` for that namespace will not be created and even if you start some pipelines, these will not be pruned:
  ```
  oc create -f namespace-two
  oc get cj -n openshift-pipelines -o json | jq '.items[0] | select(.metadata.labels."tektonconfig.operator.tekton.dev/pruner.ns"=="namespace-two").metadata.name'
  ```
  
## Error logs :red_circle:
Check for the reconciler error inside the `openshift-pipelines-operator lifecycle` container:
  ```
  $ oc logs -n openshift-operators $(oc get po -n openshift-operators -l app=openshift-pipelines-operator -o jsonpath='{.items[0].metadata.name}') -c openshift-pipelines-operator lifecycle | jq -R 'fromjson? | select(.level=="error") | select(.msg |contains("not found"))'
  ...
  {
    "level": "error",
    "logger": "tekton-operator-lifecycle",
    "caller": "tektonconfig/tektonconfig.go:142",
    "msg": "namespaces \"namespace-one\" not found",
    "commit": "282cdb3",
    "knative.dev/pod": "openshift-pipelines-operator-5c66f65888-mn28l",
    "knative.dev/controller": "github.com.tektoncd.operator.pkg.reconciler.shared.tektonconfig.Reconciler",
    "knative.dev/kind": "operator.tekton.dev.TektonConfig",
    "knative.dev/traceid": "bc69da1b-95b4-4a5f-bed0-2b8f9ea91899",
    "knative.dev/key": "config",
    "stacktrace": "github.com/tektoncd/operator/pkg/reconciler/shared/tektonconfig.(*Reconciler).ReconcileKind\n\t/go/src/github.com/tektoncd/operator/pkg/reconciler/shared/tektonconfig/tektonconfig.go:142\ngithub.com/tektoncd/operator/pkg/client/injection/reconciler/operator/v1alpha1/tektonconfig.(*reconcilerImpl).Reconcile\n\t/go/src/github.com/tektoncd/operator/pkg/client/injection/reconciler/operator/v1alpha1/tektonconfig/reconciler.go:235\nknative.dev/pkg/controller.(*Impl).processNextWorkItem\n\t/go/src/github.com/tektoncd/operator/vendor/knative.dev/pkg/controller/controller.go:542\nknative.dev/pkg/controller.(*Impl).RunContext.func3\n\t/go/src/github.com/tektoncd/operator/vendor/knative.dev/pkg/controller/controller.go:491"
  }
  ```

## Workaround
Manually remove the `CronJob` for the missing namespace:
```
 // replace <NAMESPACE_NOT_FOUND> with the removed (not found) namespace
 oc delete cj -n openshift-pipelines $(oc get cj -n openshift-pipelines -o json | jq '.items[0] | select(.metadata.labels."tektonconfig.operator.tekton.dev/pruner.ns"=="<NAMESPACE_NOT_FOUND>").metadata.name')
```

## Environment

Tested on version:
```
$ tkn version
Client version: 0.22.0
Pipeline version: v0.41.1
Triggers version: v0.22.2

$ oc version
Client Version: 4.10.30
Server Version: 4.11.36
Kubernetes Version: v1.24.11+af0420d

$ oc get sub -n openshift-operators
NAME                              PACKAGE                           SOURCE             CHANNEL
openshift-pipelines-operator-rh   openshift-pipelines-operator-rh   redhat-operators   pipelines-1.9
```
