# Helm Resource for Concourse

Deploy to [Kubernetes Helm](https://github.com/kubernetes/helm) from [Concourse](https://concourse.ci/).

## Installing

Add the resource type to your pipeline:

```yaml
resource_types:
- name: helm
  type: docker-image
  source:
    repository: aptomi/concourse-helm-resource
```

## Source Configuration

You have two options of specifying connection to the Kubernetes cluster:  
* Single parameter
    * `kubeconfig`: *Required.* Kube config for the cluster
* Separate parameters
    * `cluster_url`: *Required.* URL to Kubernetes Master API service
    * `cluster_ca`: *Optional.* Base64 encoded PEM. Required if `cluster_url` is https.
    * `token`: *Optional.* Bearer token for Kubernetes.  This, 'token_path' or `admin_key`/`admin_cert` are required if `cluster_url` is https.
    * `token_path`: *Optional.* Path to file containing the bearer token for Kubernetes.  This, 'token' or `admin_key`/`admin_cert` are required if `cluster_url` is https.
    * `admin_key`: *Optional.* Base64 encoded PEM. Required if `cluster_url` is https and no `token` or 'token_path' is provided.
    * `admin_cert`: *Optional.* Base64 encoded PEM. Required if `cluster_url` is https and no `token` or 'token_path' is provided.
    
Helm-related parameters are defined as follows:    
* `release`: *Optional.* Name of the release (not a file, a string). (Default: autogenerated by helm)
* `namespace`: *Optional.* Kubernetes namespace the chart will be installed into. (Default: default)
* `helm_init_server`: *Optional.* Installs helm into the cluster if not already installed. (Default: false)
* `tiller_namespace`: *Optional.* Kubernetes namespace where tiller is running (or will be installed to). (Default: kube-system)
* `tiller_service_account`: *Optional* Name of the service account that tiller will use (only applies if helm_init_server is true).
* `repos`: *Optional.* Array of Helm repositories to initialize, each repository is defined as an object with properties `name`, `url` (required) username and password (optional).
* `helm_host`: *Optional* Address of Tiller. Skips helm discovery process. (only applies if `helm_init_server` is false).
* `tls_enabled`: *Optional* Uses TLS for all interactions with Tiller. (Default: false)
* `helm_ca`: *Optional* Private CA that is used to issue certificates for Tiller clients and servers (only applies if tls_enabled is true).
* `helm_cert`: *Optional* Certificate for Client (only applies if tls_enabled is true).
* `helm_key`: *Optional* Key created for Client when doing a secure Tiller install (only applies if tls_enabled is true).
* `tiller_cert`: *Optional* Certificate for Tiller (only applies if tls_enabled and helm_init_server are true).
* `tiller_key`: *Optional* Key created for Tiller when doing a secure Tiller install (only applies if tls_enabled and helm_init_server are true).

## Behavior

### `check`: Check for new releases

Any new revisions to the release are returned, no matter their current state. The release must be specified in the
source for `check` to work.

### `in`: Not Supported

### `out`: Deploy the helm chart

Deploys a Helm chart onto the Kubernetes cluster. Tiller must be already installed
on the cluster.

#### Parameters

* `chart`: *Required.* Either the file containing the helm chart to deploy (ends with .tgz) or the name of the chart (e.g. `stable/mysql`).
* `namespace`: *Optional.* Either a file containing the name of the namespace or the name of the namespace. (Default: taken from source configuration).
* `release`: *Optional.* Either a file containing the name of the release or the name of the release. (Default: taken from source configuration).
* `values`: *Optional.* File containing the values.yaml for the deployment. Supports setting multiple value files using an array.
* `override_values`: *Optional.* Array of values that can override those defined in values.yaml. Each entry in
  the array is a map containing a key and a value or path. Value is set directly while path reads the contents of
  the file in that path. A `hide: true` parameter ensures that the value is not logged and instead replaced with `***HIDDEN***`.
  A `type: string` parameter makes sure Helm always treats the value as a string (uses the `--set-string` option to Helm; useful if the value varies
  and may look like a number, eg. if it's a Git commit hash).
* `version`: *Optional* Chart version to deploy, can be a file or a value. Only applies if `chart` is not a file.
* `delete`: *Optional.* Deletes the release instead of installing it. Requires the `name`. (Default: false)
* `test`: *Optional.* Test the release instead of installing it. Requires the `release`. (Default: false)
* `purge`: *Optional.* Purge the release on delete. (Default: false)
* `replace`: *Optional.* Replace deleted release with same name. (Default: false)
* `force`: *Optional.* Force resource update through delete/recreate if needed. (Default: false)
* `devel`: *Optional.* Allow development versions of chart to be installed. This is useful when wanting to install pre-release
  charts (i.e. 1.0.2-rc1) without having to specify a version. (Default: false)
* `debug`: *Optional.* Dry run the helm install with the debug flag which logs interpolated chart templates. (Default: false)
* `wait_until_ready`: *Optional.* Set to the number of seconds it should wait until all the resources in
    the chart are ready. (Default: `0` which means don't wait).
* `recreate_pods`: *Optional.* This flag will cause all pods to be recreated when upgrading. (Default: false)
* `show_diff`: *Optional.* Show the diff that is applied if upgrading an existing successful release. Will not be used when `devel` is set. (Default: false)
* `exit_after_diff`: *Optional.* Show the diff but don't actually install/upgrade. (Default: false)
* `reuse_values`: *Optional.* When upgrading, reuse the last release's values. (Default: false)

## Example

### Out

Define the resource:

```yaml
resources:
- name: myapp-helm
  type: helm
  source:
    cluster_url: https://kube-master.domain.example
    cluster_ca: _base64 encoded CA pem_
    admin_key: _base64 encoded key pem_
    admin_cert: _base64 encoded certificate pem_
    repos:
      - name: some_repo
        url: https://somerepo.github.io/charts
```

Add to job:

```yaml
jobs:
  # ...
  plan:
  - put: myapp-helm
    params:
      chart: source-repo/chart-0.0.1.tgz
      values: source-repo/values.yaml
      override_values:
      - key: replicas
        value: 2
      - key: version
        path: version/number # Read value from version/number
      - key: secret
        value: ((my-top-secret-value)) # Pulled from a credentials backend like Vault
        hide: true # Hides value in output
      - key: image.tag
        path: version/image_tag # Read value from version/number
        type: string            # Make sure it's interpreted as a string by Helm (not a number)
```
