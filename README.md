# sample-rollouts-metric-plugin
This contains an example plugin for use with Argo Rollouts plugin system

### Build

To build a release build run the command below:
```bash
make build-sample-plugin
```

To build a debug build run the command below:
```bash
make build-sample-plugin-debug
```

### Attaching a debugger to debug build
If using goland you can attach a debugger to the debug build by following the directions https://www.jetbrains.com/help/go/attach-to-running-go-processes-with-debugger.html

You can also do this with many other debuggers as well. Including cli debuggers like delve.
## Using a Metric Plugin

There are two methods of installing and using an argo rollouts plugin. The first method is to mount up the plugin executable
into the rollouts controller container. The second method is to use a HTTP(S) server to host the plugin executable.

### Mounting the plugin executable into the rollouts controller container

There are a few ways to mount the plugin executable into the rollouts controller container. Some of these will depend on your
particular infrastructure. Here are a few methods:

* Using an init container to download the plugin executable
* Using a Kubernetes volume mount with a shared volume such as NFS, EBS, etc.
* Building the plugin into the rollouts controller container

Then you can use setup the configmap to point to the plugin executable. Example:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argo-rollouts-config
data:
  plugins: |-
    metrics:
    - name: "prometheus" # name of the plugin uses the name to find this configuration, it must match the name required by the plugin
      location: "file://./my-custom-plugin" # supports http(s):// urls and file://
```

### Using a HTTP(S) server to host the plugin executable

Argo Rollouts supports downloading the plugin executable from a HTTP(S) server. To use this method, you will need to
configure the controller via the `argo-rollouts-config` configmaps `pluginLocation` to an http(s) url. Example:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argo-rollouts-config
data:
  plugins: |-
    metrics:
    - name: "prometheus" # name of the plugin uses the name to find this configuration, it must match the name required by the plugin
      location: "https://github.com/argoproj-labs/sample-rollouts-metric-plugin/releases/download/v0.0.3/metric-plugin-linux-amd64" # supports http(s):// urls and file://
      sha256: "08f588b1c799a37bbe8d0fc74cc1b1492dd70b2c" #optional sha256 checksum of the plugin executable
```

### Sample Analysis Template

An example for this sample plugin below:
```
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 5s
      # NOTE: prometheus queries return results in the form of a vector.
      # So it is common to access the index 0 of the returned array to obtain the value
      successCondition: result[0] >= 8
      failureLimit: 2
      count: 3
      provider:
        plugin:
          argoproj-labs/sample-prometheus:
            address: http://prometheus.local
            step: 1m
            query: |
              machine_cpu_cores
```
