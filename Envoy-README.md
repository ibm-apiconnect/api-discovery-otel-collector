# Envoy configuration for Opentelemetry Collector in Api Discovery

**Pre-req:** Ensure istio is installed as per [istio-readme](https://github.com/ibm-apiconnect/api-discovery-otel-collector/blob/main/Istio-README.md)

1. Deploy any rest api app deployment and service and make sure the Istio VirtualService is deployed.<br />
    1.1 As part of the Istio deployment where the application is deployed will have been annotated so that all pods in that namespace will have the istio-proxy envoy sidecar deployed on them.
2. Verify the EnvoyFilter Istio CR [EnvoyFilter.yaml](https://github.com/ibm-apiconnect/api-discovery-otel-collector/blob/main/apidiscovery/templates/envoy-filter.yaml) is installed.<br />
    2.1 `istioctl proxy-config` allows you to see all the envoy configurations applied to a pod. It can be filtered to see the configured envoy clusters, listeners, routes, etc.<br /> &nbsp;
    e.g. Running the `istioctl proxy-config` against the created rest api pod will be used to validate that the Envoy Filter has been applied
    ```
    istioctl pc listener <podname> -n <namespace> -o yaml 
    ```
3. Change the envoy logging level on the rest api pod so that the debug logs used in the wasm filter show up
    ```
    istioctl pc log <podname> -n <namespace> --level info  
    ```
4. Run some API calls through the rest api app so that the envoy filter will collect the API calls to send to the Otel collector. <br /> &nbsp;
**Note**: The Envoy wasm filter will send a trace to the otel collector when the count reaches a minimum of 200 valid API calls.
5. Inspect the istio-proxy logs on the pod to validate that the wasm filter logs can be seen  
    ```
    kubectl logs <podname> -n <namespace> -c istio-proxy
    ```
