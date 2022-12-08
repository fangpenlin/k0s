# Configuration options

## Using a configuration file

k0s can be installed without a config file. In that case the default configuration will be used. You can, though, create and run your own non-default configuration (used by the k0s controller nodes).

k0s supports providing only partial configurations. In case of partial configuration is provided, k0s will use the defaults for any missing values.

1. Generate a yaml config file that uses the default settings.

    ```shell
    mkdir -p /etc/k0s
    k0s config create > /etc/k0s/k0s.yaml
    ```

2. Modify the new yaml config file according to your needs, refer to [Configuration file reference](#configuration-file-reference) below. You can remove the default values if wanted as k0s supports partial configs too.

3. Install k0s with your new config file.

    ```shell
    sudo k0s install controller -c /etc/k0s/k0s.yaml
    ```

4. If you need to modify your existing configuration later on, you can change your config file also when k0s is running, but remember to restart k0s to apply your configuration changes.

    ```shell
    sudo k0s stop
    sudo k0s start
    ```

## Configuration file reference

**CAUTION**: As many of the available options affect items deep in the stack, you should fully understand the correlation between the configuration file components and your specific environment before making any changes.

A YAML config file follows, with defaults as generated by the `k0s config create` command:

```yaml
apiVersion: k0s.k0sproject.io/v1beta1
kind: ClusterConfig
metadata:
  name: k0s
spec:
  api:
    address: 192.168.68.104
    port: 6443
    k0sApiPort: 9443
    externalAddress: my-lb-address.example.com
    sans:
      - 192.168.68.104
    tunneledNetworkingMode: false
    extraArgs: {}
  storage:
    type: etcd
    etcd:
      peerAddress: 192.168.68.104
      extraArgs: {}
  network:
    podCIDR: 10.244.0.0/16
    serviceCIDR: 10.96.0.0/12
    provider: kuberouter
    calico: null
    clusterDomain: cluster.local
    dualStack: {}
    kuberouter:
      mtu: 0
      peerRouterIPs: ""
      peerRouterASNs: ""
      autoMTU: true
      hairpinMode: false
    kubeProxy:
      disabled: false
      mode: iptables
      metricsBindAddress: 0.0.0.0:10249
  telemetry:
    enabled: true
  controllerManager:
    extraArgs: {}
  scheduler:
    extraArgs: {}
  installConfig:
    users:
      etcdUser: etcd
      kineUser: kube-apiserver
      konnectivityUser: konnectivity-server
      kubeAPIserverUser: kube-apiserver
      kubeSchedulerUser: kube-scheduler
  images:
    konnectivity:
      image: quay.io/k0sproject/apiserver-network-proxy-agent
      version: 0.0.32-k0s1
    metricsserver:
      image: registry.k8s.io/metrics-server/metrics-server
      version: v0.6.2
    kubeproxy:
      image: registry.k8s.io/kube-proxy
      version: v1.25.4
    coredns:
      image: docker.io/coredns/coredns
      version: v1.10.0
    calico:
      cni:
        image: docker.io/calico/cni
        version: v3.24.5
      node:
        image: docker.io/calico/node
        version: v3.24.5
      kubecontrollers:
        image: docker.io/calico/kube-controllers
        version: v3.24.5
    kuberouter:
      cni:
        image: docker.io/cloudnativelabs/kube-router
        version: v1.5.1
      cniInstaller:
        image: quay.io/k0sproject/cni-node
        version: 1.1.1-k0s.0
    default_pull_policy: IfNotPresent
  konnectivity:
    agentPort: 8132
    adminPort: 8133
```

## `spec` Key Detail

### `spec.api`

| Element           | Description                                                                                                                                                                                                                 |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `externalAddress` | The loadbalancer address (for k0s controllers running behind a loadbalancer). Configures all cluster components to connect to this address and also configures this address for use  when joining new nodes to the cluster. |
| `address`         | Local address on which to bind an API. Also serves as one of the addresses pushed on the k0s create service certificate on the API. Defaults to first non-local address found on the node.                                  |
| `sans`            | List of additional addresses to push to API servers serving the certificate.                                                                                                                                                |
| `extraArgs`       | Map of key-values (strings) for any extra arguments to pass down to Kubernetes api-server process.                                                                                                                          |
| `port`¹           | Custom port for kube-api server to listen on (default: 6443)                                                                                                                                                                |
| `k0sApiPort`¹     | Custom port for k0s-api server to listen on (default: 9443)                                                                                                                                                                 |
| `tunneledNetworkingMode`     | Whether to tunnel Kubernetes access from worker nodes via local port forwarding. (default: `false`)                                                                                                                                                                 |

¹ If `port` and `k0sApiPort` are used with the `externalAddress` element, the loadbalancer serving at `externalAddress` must listen on the same ports.

### `spec.storage`

| Element            | Description                                                                                                                                                            |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `type`             | Type of the data store (valid values:`etcd` or `kine`). **Note**: Type `etcd` will cause k0s to create and manage an elastic etcd cluster within the controller nodes. |
| `etcd.peerAddress` | Node address used for etcd cluster peering.                                                                                                                            |
| `etcd.extraArgs`   | Map of key-values (strings) for any extra arguments to pass down to etcd process.                                                                                      |
| `kine.dataSource`  | [kine](https://github.com/k3s-io/kine) datasource URL.                                                                                                                 |

### `spec.network`

| Element         | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
|-----------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `provider`      | Network provider (valid values: `calico`, `kuberouter`, or `custom`). For `custom`, you can push any network provider (default: `kuberouter`). Be aware that it is your responsibility to configure all of the CNI-related setups, including the CNI provider itself and all necessary host levels setups (for example, CNI binaries). **Note:** Once you initialize the cluster with a network provider the only way to change providers is through a full cluster redeployment. |
| `podCIDR`       | Pod network CIDR to use in the cluster.                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `serviceCIDR`   | Network CIDR to use for cluster VIP services.                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `clusterDomain` | Cluster Domain to be passed to the [kubelet](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/#kubelet-config-k8s-io-v1beta1-KubeletConfiguration) and the coredns configuration.                                                                                                                                                                                                                                                                           |

#### `spec.network.calico`

| Element                 | Description                                                                                                                                                                                                                                                                                                                                                                                                |
|-------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `mode`                  | `vxlan` (default), `ipip` or `bird`                                                                                                                                                                                                                                                                                                                                                                        |
| `overlay`               | Overlay mode: `Always` (default), `CrossSubnet` or `Never` (requires `mode=vxlan` to disable calico overlay-network).                                                                                                                                                                                                                                                                                      |
| `vxlanPort`             | The UDP port for VXLAN (default: `4789`).                                                                                                                                                                                                                                                                                                                                                                  |
| `vxlanVNI`              | The virtual network ID for VXLAN (default: `4096`).                                                                                                                                                                                                                                                                                                                                                        |
| `mtu`                   | MTU for overlay network (default: `0`, which causes Calico to detect optimal MTU during bootstrap).                                                                                                                                                                                                                                                                                                        |
| `wireguard`             | Enable wireguard-based encryption (default: `false`). Your host system must be wireguard ready (refer to the [Calico documentation](https://docs.projectcalico.org/security/encrypt-cluster-pod-traffic) for details).                                                                                                                                                                                     |
| `flexVolumeDriverPath`  | The host path for Calicos flex-volume-driver(default: `/usr/libexec/k0s/kubelet-plugins/volume/exec/nodeagent~uds`). Change this path only if the default path is unwriteable (refer to [Project Calico Issue #2712](https://github.com/projectcalico/calico/issues/2712) for details). Ideally, you will pair this option with a custom ``volumePluginDir`` in the profile you use for your worker nodes. |
| `ipAutodetectionMethod` | Use to force Calico to pick up the interface for pod network inter-node routing (default: `""`, meaning not set, so that Calico will instead use its defaults). For more information, refer to the [Calico documentation](https://docs.projectcalico.org/reference/node/configuration#ip-autodetection-methods).                                                                                           |
| `envVars`               | Map of key-values (strings) for any calico-node [environment variable](https://docs.projectcalico.org/reference/node/configuration#ip-autodetection-methods).                                                                                                                                                                                                                                              |

#### `spec.network.calico.envVars`

Environment variable's value must be string, e.g.:

```yaml
spec:
  network:
    provider: calico
    calico:
      envVars:
        TEST_BOOL_VAR: "true"
        TEST_INT_VAR: "42"
        TEST_STRING_VAR: test
```

K0s runs Calico with some predefined vars, which can be overwritten by setting new value in `spec.network.calico.envVars`:

```shell
CALICO_IPV4POOL_CIDR: "{{ spec.network.podCIDR }}"
CALICO_DISABLE_FILE_LOGGING: "true"
FELIX_DEFAULTENDPOINTTOHOSTACTION: "ACCEPT"
FELIX_LOGSEVERITYSCREEN: "info"
FELIX_HEALTHENABLED: "true"
FELIX_PROMETHEUSMETRICSENABLED: "true"
```

In SingleStack mode there are additional vars:

```shell
FELIX_IPV6SUPPORT: "false"
```

In DualStack mode there are additional vars:

```shell
CALICO_IPV6POOL_NAT_OUTGOING: "true"
FELIX_IPV6SUPPORT: "true"
IP6: "autodetect"
CALICO_IPV6POOL_CIDR: "{{ spec.network.dualStack.IPv6podCIDR }}"
```

#### `spec.network.kuberouter`

| Element          | Description                                                                                                                                        |
| ---------------- |----------------------------------------------------------------------------------------------------------------------------------------------------|
| `autoMTU`        | Autodetection of used MTU (default: `true`).                                                                                                       |
| `mtu`            | Override MTU setting, if `autoMTU` must be set to `false`).                                                                                        |
| `metricsPort`    | Kube-router metrics server port. Set to 0 to disable metrics  (default: `8080`).                                                                   |
| `peerRouterIPs`  | Comma-separated list of [global peer addresses](https://github.com/cloudnativelabs/kube-router/blob/master/docs/bgp.md#global-external-bgp-peers). |
| `peerRouterASNs` | Comma-separated list of [global peer ASNs](https://github.com/cloudnativelabs/kube-router/blob/master/docs/bgp.md#global-external-bgp-peers).      |
| `hairpinMode`    | Activate hairpinMode (default: `false`) (https://github.com/cloudnativelabs/kube-router/blob/master/docs/user-guide.md#hairpin-mode)               |

**Note**: Kube-router allows many networking aspects to be configured per node, service, and pod (for more information, refer to the [Kube-router user guide](https://github.com/cloudnativelabs/kube-router/blob/master/docs/user-guide.md)).

#### `spec.network.kubeProxy`

| Element          | Description                                                                                                                                        |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `disabled`       | Disable kube-proxy altogether (default: `false`).                                                                                                       |
| `mode`           | Kube proxy operating mode, supported modes `iptables`, `ipvs`, `userspace` (default: `iptables`) |

### `spec.controllerManager`

| Element     | Description                                                                                                             |
| ----------- | ----------------------------------------------------------------------------------------------------------------------- |
| `extraArgs` | Map of key-values (strings) for any extra arguments you want to pass down to the Kubernetes controller manager process. |

### `spec.scheduler`

| Element     | Description                                                                                                |
| ----------- | ---------------------------------------------------------------------------------------------------------- |
| `extraArgs` | Map of key-values (strings) for any extra arguments you want to pass down to Kubernetes scheduler process. |

### `spec.workerProfiles`

Worker profiles are used to set kubelet parameters can for a worker. Each worker profile is then used to generate a config map containing a custom `kubelet.config.k8s.io` object.

For a list of possible kubelet configuration keys: [go here](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/).

The worker profiles are defined as an array of `spec.workerProfiles.workerProfile`. Each element has following properties:

| Property | Description                                                    |
| -------- | -------------------------------------------------------------- |
| `name`   | String; name to use as profile selector for the worker process |
| `values` | Mapping object                                                 |

For each profile, the control plane creates a separate ConfigMap with `kubelet-config yaml`. Based on the `--profile` argument given to the `k0s worker`, the corresponding ConfigMap is used to extract the `kubelet-config.yaml` file. `values` are recursively merged with default `kubelet-config.yaml`

Note that there are several fields that cannot be overridden:

- `clusterDNS`
- `clusterDomain`
- `apiVersion`
- `kind`
- `staticPodURL`

#### Examples

##### Feature Gates

The below is an example of a worker profile with feature gates enabled:

```yaml
spec:
  workerProfiles:
    - name: custom-feature-gate      # name of the worker profile
      values:
         featureGates:        # feature gates mapping
            DevicePlugins: true
            Accelerators: true
            AllowExtTrafficLocalEndpoints: false
```

##### Custom volumePluginDir

```yaml
spec:
  workerProfiles:
    - name: custom-pluginDir
      values:
         volumePluginDir: /var/libexec/k0s/kubelet-plugins/volume/exec
```

##### Eviction Policy

```yaml
spec:
  workerProfiles:
    - name: custom-eviction
      values:
        evictionHard:
          memory.available: "500Mi"
          nodefs.available: "1Gi"
          imagefs.available: "100Gi"
        evictionMinimumReclaim:
          memory.available: "0Mi"
          nodefs.available: "500Mi"
          imagefs.available: "2Gi"
```

##### Unsafe Sysctls

```yaml
spec:
  workerProfiles:
    - name: custom-eviction
      values:
        allowedUnsafeSysctls:
          - fs.inotify.max_user_instances
```

### `spec.images`

Nodes under the `images` key all have the same basic structure:

```yaml
spec:
  images:
    coredns:
      image: quay.io/coredns/coredns
      version: v1.7.0
```

#### Available keys

- `spec.images.konnectivity`
- `spec.images.metricsserver`
- `spec.images.kubeproxy`
- `spec.images.coredns`
- `spec.images.calico.cni`
- `spec.images.calico.flexvolume`
- `spec.images.calico.node`
- `spec.images.calico.kubecontrollers`
- `spec.images.kuberouter.cni`
- `spec.images.kuberouter.cniInstaller`
- `spec.images.repository`¹

¹ If `spec.images.repository` is set and not empty, every image will be pulled from `images.repository`

If `spec.images.default_pull_policy` is set and not empty, it will be used as a pull policy for each bundled image.

#### Example

```yaml
images:
  repository: "my.own.repo"
  konnectivity:
    image: calico/kube-controllers
    version: v3.16.2
  metricsserver:
    image: registry.k8s.io/metrics-server/metrics-server
    version: v0.6.2
```

In the runtime the image names are calculated as `my.own.repo/calico/kube-controllers:v3.16.2` and `my.own.repo/metrics-server/metrics-server:v0.6.2`. This only affects the the imgages pull location, and thus omitting an image specification here will not disable component deployment.

### `spec.extensions.helm`

`spec.extensions.helm` is the config file key in which you configure the list of [Helm](https://helm.sh) repositories and charts to deploy during cluster bootstrap (for more information, refer to [Helm Charts](helm-charts.md)).

### `spec.extensions.storage`

`spec.extensions.storage` controls bundled storage provider.
The default value `external` makes no storage deployed.

To enable [embedded host-local storage provider](storage.md#bundled-openebs-storage) use the following configuration:

```yaml
spec:
  extensions:
    storage:
      type: openebs_local_storage
```

### `spec.konnectivity`

The `spec.konnectivity` key is the config file key in which you configure Konnectivity-related settings.

- `agentPort` agent port to listen on (default 8132)
- `adminPort` admin port to listen on (default 8133)

### `spec.telemetry`

To improve the end-user experience k0s is configured by defaul to collect telemetry data from clusters and send it to the k0s development team. To disable the telemetry function, change the `enabled` setting to `false`.

The telemetry interval is ten minutes.

```yaml
spec:
  telemetry:
    enabled: true
```

## Disabling controller components

k0s allows completely disabling some of the system components. This allows the user to build a minimal Kubernetes control plane and use what ever components they need to fullfill their need for the controlplane. Disabling the system components happens through a commandline flag for the controller process:

```sh
--disable-components strings                     disable components (valid items: api-config,autopilot,control-api,coredns,csr-approver,endpoint-reconciler,helm,konnectivity-server,kube-controller-manager,kube-proxy,kube-scheduler,kubelet-config,metrics-server,network-provider,node-role,system-rbac)

```

If you use k0sctl just add the flag when installing the cluster for the first controller at `spec.hosts.installFlags` in the config file like e.g.:

```yaml
spec:
  hosts:
  - role: controller
    installFlags:
    - --disable-components metrics-server
```

As seen from the component list, the only always-on component is the Kubernetes api-server, without that k0s serves no purpose.