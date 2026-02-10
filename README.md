
# Making Intel GPUs available in your kubernetes cluster

Getting Intel GPUs to work in your Kubernetes cluster is fairly straightforward; two helm charts are required one not needing any configuring and the second is extremely simple to configure. There is a hard dependency on [cert-manager](https://artifacthub.io/packages/helm/cert-manager/cert-manager) and an optional dependency on [Node Feature Discovery](https://artifacthub.io/packages/helm/node-feature-discovery/node-feature-discovery)

## The background

For this tutorial I assume you have the following:

 * A Working K3S cluster
 * At least one node with some sort of Intel GPU
 * Helm installed
 * Cert Manager installed

This tutorial has only been tested on Debian 13 as of February 9, 2026.

## Installing needed packages

In order to effectively use your Intel GPU for quicksync we need to install a package, `intel-media-va-driver-non-free`. Since I use Debian's cloud images the step to enable to correct repositories is slightly different for some reason:

```
$sudo sed -i 's/^Components: main$/Components: main non-free non-free-firmware/' /etc/apt/sources.list.d/debian.sources
```

If you installed Debian normally you would want to edit `/etc/apt/sources.list`, the first line is likely the correct one to edit:

```
$sudo nano /etc/apt/sources.list
deb http://mirror.csclub.uwaterloo.ca/debian/ bookworm main
```

Stick `non-free non-free-firmware` onto the end so that it looks something like this:

```
deb http://mirror.csclub.uwaterloo.ca/debian/ bookworm main non-free-firmware contrib
```

Press Ctrl+X to save, then Y to confirm and then enter.

```
$sudo apt-get update
$sudo apt-get install intel-media-va-driver-non-free
```

Optionally you can install `vainfo` to see what codecs your GPU supports:

```
$vainfo
error: can't connect to X server!
libva info: VA-API version 1.17.0
libva info: Trying to open /usr/lib/x86_64-linux-gnu/dri/iHD_drv_video.so
libva info: Found init function __vaDriverInit_1_17
libva info: va_openDriver() returns 0
vainfo: VA-API version: 1.17 (libva 2.12.0)
vainfo: Driver version: Intel iHD driver for Intel(R) Gen Graphics - 23.1.1 ()
vainfo: Supported profile and entrypoints
VAProfileNone                   : VAEntrypointVideoProc
VAProfileNone                   : VAEntrypointStats
VAProfileMPEG2Simple            : VAEntrypointVLD
VAProfileMPEG2Simple            : VAEntrypointEncSlice
VAProfileMPEG2Main              : VAEntrypointVLD
VAProfileMPEG2Main              : VAEntrypointEncSlice
VAProfileH264Main               : VAEntrypointVLD
VAProfileH264Main               : VAEntrypointEncSlice
VAProfileH264Main               : VAEntrypointFEI
VAProfileH264Main               : VAEntrypointEncSliceLP
VAProfileH264High               : VAEntrypointVLD
VAProfileH264High               : VAEntrypointEncSlice
VAProfileH264High               : VAEntrypointFEI
VAProfileH264High               : VAEntrypointEncSliceLP
VAProfileVC1Simple              : VAEntrypointVLD
VAProfileVC1Main                : VAEntrypointVLD
VAProfileVC1Advanced            : VAEntrypointVLD
VAProfileJPEGBaseline           : VAEntrypointVLD
VAProfileJPEGBaseline           : VAEntrypointEncPicture
VAProfileH264ConstrainedBaseline: VAEntrypointVLD
VAProfileH264ConstrainedBaseline: VAEntrypointEncSlice
VAProfileH264ConstrainedBaseline: VAEntrypointFEI
VAProfileH264ConstrainedBaseline: VAEntrypointEncSliceLP
VAProfileVP8Version0_3          : VAEntrypointVLD
VAProfileVP8Version0_3          : VAEntrypointEncSlice
VAProfileHEVCMain               : VAEntrypointVLD
VAProfileHEVCMain               : VAEntrypointEncSlice
VAProfileHEVCMain               : VAEntrypointFEI
VAProfileHEVCMain10             : VAEntrypointVLD
VAProfileHEVCMain10             : VAEntrypointEncSlice
VAProfileVP9Profile0            : VAEntrypointVLD
VAProfileVP9Profile2            : VAEntrypointVLD
```

The codecs that have `VAEntrypointEncSlice` are the ones your gpu can encode, in my case I can encode up to and including H265 10bit. Your results may vary.

## Tagging your nodes

The Intel GPU Device Plugin chart we'll be installing will create a DaemonSet with a node selector of `intel.feature.node.kubernetes.io/gpu: true` so that any pods created by the DaemonSet will only run on nodes with Intel GPUs. Here you have two options:

 * Manually label nodes
 * Use Node Feature Discovery to label appropriate nodes

## Manual tagging

Manually tagging your nodes is simple:

```
$kubectl label nodes NODE-NAME intel.feature.node.kubernetes.io/gpu=true
```

Do that for each node that has an Intel GPU.

In `values.yaml` you'll need to uncomment the `nodeFeatureRule` line. By default the intel device plugins helm chart uses NFD

And that's it. 

## Node Feature Discovery

Node Feature Discovery uses rules to determine what to label and why. The Intel GPU Device Plugin that we'll be installing comes with pre-made rules to automatically label your nodes.

### Installing NFD

```
$helm repo add node-feature-discovery https://kubernetes-sigs.github.io/node-feature-discovery/charts
$helm repo update

$helm upgrade --install node-feature-discovery node-feature-discovery/node-feature-discovery
```

## Intel GPU Plugin

```
$helm repo add intel https://intel.github.io/helm-charts/
$helm repo update

$helm upgrade --install device-plugin-operator intel/intel-device-plugins-operator
$helm upgrade --install gpu-device-plugin intel/intel-device-plugins-gpu --values values.yaml
```

The values.yaml file is quite sparse, having only one key-pair: `sharedDevNum: 4`. Setting sharedDevNum to N where N is greater than 1 allows N pods to use the same GPU on a node. Uncommenting `nodeFeatureRule: false` disables the usage of Node Feature Discovery rules for detecting various Intel GPUs

And that's it! You are now ready to use your Intel GPU in a pod.

## Test deployment

To make sure it works run the test pod in vainfo-pod.yaml:

```
$kubectl apply -f vainfo-pod.yaml
```

Give the cluster a few seconds to start the pod then check its output:

```
$kubectl logs vainfo
libva info: VA-API version 1.22.0
libva info: User environment variable requested driver 'iHD'
libva info: Trying to open /usr/local/lib/dri/iHD_drv_video.so
libva info: Found init function __vaDriverInit_1_22
libva info: va_openDriver() returns 0
Trying display: drm
vainfo: VA-API version: 1.22 (libva 2.22.0)
vainfo: Driver version: Intel iHD driver for Intel(R) Gen Graphics - 24.3.3 (9876014)
vainfo: Supported profile and entrypoints
        VAProfileNone                   : VAEntrypointVideoProc
        *snip*
```

It works!

Remove the deployment:

```
$kubectl delete -f vainfo-pod.yaml
```

## Jellyfin

I have set up [Jellyfin](https://github.com/UntouchedWagons/K3S-Cluster-Setup/blob/929678afa738c9527a838c8735cb917b10c4a521/production/default/jellyfin/service.yaml) to use the GPU of the node it's running on for transcoding. Configuring a container is simple:

    spec:
      nodeSelector:
        intel.feature.node.kubernetes.io/gpu: "true"
      containers:
        - name: jellyfin
          image: jellyfin/jellyfin:10.8.13-1
          resources:
            limits:
              gpu.intel.com/i915: "1"
            requests:
              gpu.intel.com/i915: "1"

Apply the manifest then go into the Playback settings of Jellyfin and choose Intel QuickSync.

## Changelog

 * February 9, 2026 - Spring cleaning, added test pod, adjusted valus.yaml slightly
 * March 5, 2024 - Initial commit

Sources:

 * https://www.reddit.com/r/selfhosted/comments/121vb07/plex_on_kubernetes_with_intel_igpu_passthrough/