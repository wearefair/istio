# Why this Fork
In a mutual TLS-disabled multicluster Kubernetes setup, gRPC liveness and readiness probes don't work in remote clusters. However, this behavior could not be replicated on the Kubernetes cluster running the Istio control plane.

We verified that the remote proxy sidecar containers had connectivity to Pilot Discovery. However, we noticed when comparing the logs in the sidecar containers on a remote cluster vs the local cluster that the local cluster sidecars were properly registering liveness and readiness probes, and the remote clusters weren't.

```bash
# GOOD - On the "local" control plane - we expect to see a listener registered for 8080
[2019-03-06 21:44:39.033][15][info][upstream] external/envoy/source/server/lds_api.cc:80] lds: add/update listener '10.21.3.242_8080'<Paste>

# BAD - On the "remote" cluster - where are 3333 and 9999 coming from???
[2019-03-06 22:00:16.701][10][info][upstream] external/envoy/source/common/upstream/cluster_manager_impl.cc:494] add/update cluster inbound|3333||mgmtCluster during init
[2019-03-06 22:00:16.702][10][info][upstream] external/envoy/source/common/upstream/cluster_manager_impl.cc:494] add/update cluster inbound|9999||mgmtCluster during init
```

After diving into the Istio source, we only found that 3333 and 9999 were ports returned from the [Envoy debug registry](https://github.com/istio/istio/blob/1.0.2/pilot/pkg/proxy/envoy/v2/debug.go#L324-L337).

In the Istio logs, we could see that a remote cluster registry was being picked up. However, based off of the server bootup order, it looked like the debug registry would always be loaded before the multicluster ones. Since the debug registry would always return entries for the management ports, it meant that the debug management ports would always be picked up instead of the remote cluster ones.

We added a lot more logs into this fork and deployed it in a test cluster to confirm our suspicious.

We then added a feature flag to disable registering the debug registry to see if it would fix our liveness probes. Sure enough, they do.

## Building and Running

```bash
make build docker.pilot # Builds the Docker container at istio/pilot:<sha>
```

When deploying and enabling the debug registry is desired behavior, set the following ENV var:
```bash
ENABLE_DEBUG_REGISTRY=true
```
