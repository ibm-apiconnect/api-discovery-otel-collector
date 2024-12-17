# Istio installation For Opentelemetry Collector in Api Discovery
## Pre-req: Ensure istio is installed on your cluster 

Follow the istio getting started guide if you do not have a cluster with istio already installed.   

Deploy both the istio mesh and the sample application to test with.  
https://istio.io/latest/docs/setup/getting-started/  

The collector has been tested with `istioctl` version `1.19.x` & `1.20.x` 

Istio can be deployed either as Istio telemetry or Istio Envoyfilter depending on the collector_mode in values.yaml. The Istio Envoyfilter can be configured when the request and response body need to be collected and API discovered from the installed application. Whereas it is not available in the Istio telemetry collects only other data like Method, Path, Host, status_code, content_type, and so on.

## Istio Telemetry
## Configuring Istio to support the collector  

### Update the Istio mesh config  
Firstly a reference to the discovery collector (which will be deployed in the suqsequent steps) will need to be registered in the istio MeshConfig to send trace data to the discovery service.  
See the Provider Selection section for some details: https://istio.io/latest/docs/tasks/observability/telemetry/#provider-selection    
There is an example of the patch that will need to be configured on the mesh in the [apidiscovery/patches](apidiscovery/patches) directory.   
This is a complete example of what the `istio` configmap could look like but the salient lines from the example which need to be added to your own configuration under extensionProviders are the following   
```
    extensionProviders:
      - name: "discoveryOtelCollector"
        opentelemetry:
          service: "management-api-discovery-otel-collector.istio-system.svc.cluster.local"
          port: 5555
```
These can be added by doing an edit on the `istio` configmap which will be in the namespace where istio is deployed.  
```
e.g.
kubectl edit configmap istio -n <istio-system>
```
Or if you have deployed istio via the IstioOperator you can configure the same via the MeshConfig  
See here for more details: https://istio.io/v1.10/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig 

## Istio EnvoyFilter
## Configuring Istio logs

1. Deploy any rest api app deployment and service and make sure the Istio VirtualService is deployed.<br />
    1.1 As part of the Istio deployment where the application is deployed will have been annotated so that all pods in that namespace will have the istio-proxy envoy sidecar deployed on them.
2. The EnvoyFilter Istio CR [EnvoyFilter.yaml](https://github.com/ibm-apiconnect/api-discovery-otel-collector/blob/main/apidiscovery/templates/envoy-filter.yaml) has been installed when the collector_mode is envoyfilter in [values.yaml](apidiscovery/values.yaml).<br />
    2.1 `istioctl proxy-config` allows you to see all the envoy configurations applied to a pod. It can be filtered to see the configured envoy clusters, listeners, routes, etc.<br /> &nbsp;
    e.g. Running the `istioctl proxy-config` against the created rest api pod will be used to validate that the Envoy Filter has been applied
    ```
    istioctl pc listener <podname> -n <namespace> -o yaml 
    ```
3. Change the envoy logging level on the rest api pod so that the debug logs used in the wasm filter show up
    ```
    istioctl pc log <podname> -n <namespace> --level debug  
    ```
Example output : 
```
wasmfiltertest-6df8655d7b-plg7w.default:
active loggers:
  admin: debug
  alternate_protocols_cache: debug
  aws: debug
  assert: debug
  backtrace: debug
  basic_auth: debug
  cache_filter: debug
  client: debug
  config: debug
  connection: debug
  conn_handler: debug
  compression: debug
  credential_injector: debug
  ...
```
4. Optionally, at this point any API calls which run through the rest api app will be collected by the envoy wasm filter to send to the Otel collector every 3 minutes and are discovered. <br /> &nbsp;
5. The istio-proxy logs on the pod can be seen to check the wasm filter logs 
    ```
    kubectl logs <podname> -n <namespace> -c istio-proxy
    ```
## Uninstalling the Envoy filter

The Envoy Filter can be uninstalled with the following steps
1. Run `kubectl delete EnvoyFilter wasm-envoy-filter -n istio-system`
2. Restart all the pods that are configured in any namespaces to collect traces through EnvoyFilter.
