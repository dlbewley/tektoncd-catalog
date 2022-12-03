# Red Hat Advanced Cluster Security Generate NetworkPolicy Task

Generate suggested NetworkPolicy artifacts given source Kubernetes manifests using `roxctl`.

**This is a Technology Preview feature**
Technology Preview features are not supported with Red Hat production service level agreements (SLAs) and might not be functionally complete. Red Hat does not recommend using them in production. These features provide early access to upcoming product features, enabling customers to test functionality and provide feedback during the development process. For more information about the support scope of Red Hat Technology Preview features, see <https://access.redhat.com/support/offerings/techpreview/>

## Prerequisites

This task requires an active installation of [Red Hat Advanced Cluster Security (RHACS)](https://www.redhat.com/en/resources/advanced-cluster-security-for-kubernetes-datasheet).  It also requires configuration of secrets for the Central endpoint and an API token with at least CI privileges.

<https://www.redhat.com/en/technologies/cloud-computing/openshift/advanced-cluster-security-kubernetes>

## Install the Task

```bash
kubectl apply -f https://api.hub.tekton.dev/v1/resource/tekton/task/rhacs-networkpolicy/3.73/raw
```

## Parameters

- **`manifests`**: Directory holding deployment manifests. May be relative to workspace root or fully qualified. Examples: _"k8s"_
- **`insecure-skip-tls-verify`**: Skip verification the TLS certs of the Central endpoint and registry. Examples: _"true", **"false"**_.
- **`output_file`**: File to store generated policies in. Examples: _networkpolicy.yaml_ (optional)
- **`output_dir`**: Directory to store generated policies in. Examples: _network-policies_ (optional)
- **`rox_central_endpoint`**: Secret containing the address:port tuple for StackRox Central. Default: _**rox-central-endpoint**_
- **`rox_api_token`**: Secret containing the StackRox API token with CI permissions. Default: _**rox-api-token**_

## Workspaces

- **source**: A [Workspace](https://github.com/tektoncd/pipeline/blob/main/docs/workspaces.md) containing the application manifests.

## Usage

Create secrets for authentication to RHACS Central endpoint and supply filesystem path to deployment manifest for checking.

Run this task after checking out application manifest source code.


**Example secret creation:**

```bash
kubectl create secret generic rox-api-token \
  --from-literal=rox_api_token="$ROX_API_TOKEN"
kubectl create secret generic rox-central-endpoint \
  --from-literal=rox_central_endpoint=central.stackrox.svc:443
```

**Example task use:**

```yaml
  tasks:
    - name: generate-networkpolicy
    taskRef:
      name: rhacs-networkpolicy
      kind: Task
    workspaces:
    - name: source
      workspace: shared-workspace
    params:
    - name: manifests
      value: $(workspaces.source.path)/k8s
    runAfter:
    - fetch-repository
```

**Samples:**

* [secrets.yaml](samples/secrets.yaml) example secret
* [pipeline.yaml](samples/pipeline.yaml) demonstrates use in a pipeline.
* [pipelinerun.yaml](samples/pipelinerun.yaml) demonstrates use in a pipelinerun.

# Known Issues

* Skipping TLS Verify is currently required. TLS trust bundle not working for quay.io etc.
* Even with empty values, `--output-dir` and `--output-file` may not be supplied simultaneously. Given that the entrypoint for our container image is `/roxctl` and there is no `/bin/sh` this is problematic.
  > _ERROR:    Flags [-d|--output-dir, -f|--output-file] cannot be used together_
  * **TODO:** _RFE_ 
    * Permit both flags simultaneously.
    * In case of _non-null_ values for both, `exit 1`
    * In case of _non-null_ values for one, write to that location
    * In case of _null_ values for both, write to `STDOUT`
