# Opentelemetry Collector For Api Discovery

This document describes the set up details for the OpenTelemetry Collector For Api Discovery based on different datasource types.
## Pre-req: Ensure one of the below datasource type is available on your system to collect traces through the opentelemetry collector

Example configurations for both Istio and nginx are provided here
1. Istio installation details - [Istio-Readme.md](https://github.com/ibm-apiconnect/api-discovery-otel-collector/blob/main/Istio-README.md)
2. nginx Conguration details - [Nginx-Readme.md](https://github.com/ibm-apiconnect/api-discovery-otel-collector/blob/main/Nginx-README.md)

## Pre-req: Ensure helm is installed  

See here for details  
https://helm.sh/docs/intro/install/  

### Use the Helm chart to deploy the collector.  

In the helm directory, update the [values.yaml](apidiscovery/values.yaml) file based on your apiconnect provider org, authentication and istio set up  

The following parameters require updates:  
 - `discovery.datasource_type` - (Optional) field to set the datasource type to decide if the specific telemetry configuration is needed. If the datasource type is istio, the required Istio Telemetry CR will be installed or helm will ignore the telemetry creation.
 - `discovery.apic_host_domain` - Domain name of the ApiConnect instance where discovered APIs will be sent.<br /> &nbsp; Example : `us-east-a.apiconnect.automation.ibm.com`  
 - `discovery.provider_org` - The provider org name of the apiconnect manager  
 - `discovery.apikey` - An API Key can be obtained from the api-manager for the user who has access to post the API.
An API key can be created by logging into the APIC Manager UI and selecting the "My API Keys" option under the profile icon from the top navigation bar. 
The apikey will be added to a kubernetes secret as part of the deployment and then mounted on the collector deployment pod.  
 - `discovery.processor` - The flag to enable or disable the processors. The [configmap](apidiscovery/templates/processor-configmap.yaml) enables the existing batch and memory_limiter processor when `enabled` value is given and will not provide any processors in other cases. The available batch and memory limiter processors are configurable according to the requirements. <br /> &nbsp;
Refer - [Batch processor](https://github.com/open-telemetry/opentelemetry-collector/blob/main/processor/batchprocessor/README.md#batch-processor) and [Memory Limiter processor](https://github.com/open-telemetry/opentelemetry-collector/tree/main/processor/memorylimiterprocessor#configuration)<br />
Keeping the `filter` in the [`service.pipelines.traces.processors`](apidiscovery/templates/processor-configmap.yaml#L21) while enabling processors is recommended, as we have a built-in filter processor that enhances the collector's performance.
- `namespace`: The namespace where istio or nginx has been deployed on your cluster. As required by istio's telemetry integration the collector pod will be deployed here.
- `images.api_discovery_collector`: As new versions of the collectors are released updating this property will enable the upgrade of the collector deployment. Note: collectors will require updates to ensure they remain compatible with the discovery service. Details of these updates will be available in this repository.
- `logging.log_level`: Default log_level is `info`. For debug purposes the log level can be increased to debug if needed.  

**Istio specific parameters**

- `discovery.datasource_name` - (Optional) field to set the datasource name for all APIs discovered by the collector through istio. If datasource name is empty, the namespace of the collected API will be used.
- `telemetry_namespace`: (Optional) This is where the istio Telemetry CR will be deployed. By default the values file sets this to `istio-system` as the root configuration namespace to provide mesh level collection configuration. This can be customized as you wish based on your collection requirements regarding the deployed applications you want to be discovered. See here for further details https://istio.io/latest/docs/reference/config/telemetry/. Not needed for nginx configuration.
    - For example: If there are more APIs discovered and you want to restrict some of the services, then the telemetry CR can be configured with matchLabels selector as mentioned in one of the [examples](https://istio.io/latest/docs/reference/config/telemetry/#examples)

Once you have made the required updates to the values.yaml file you can deploy the collector.

Run the following command to deploy the collector  
```
helm template . | kubectl apply -f -
```
Output should be as follows which indicate that the required Kubernetes Reources have been deployed where telemetry creation is only for Istio

```
helm template . | kubectl apply -f -
secret/api-discovery-secret created
service/management-api-discovery-otel-collector created
deployment.apps/management-api-discovery-otel-collector created
telemetry.telemetry.istio.io/api-discovery-otel created
```

Once the collector has been deployed any istio or nginx instructmented pods which have traffic running through them will begin to be discovered and sent to the apiconnect discovery service.

## Upgrading the collector

The upgrade of the otel collector mainly refers to the changes in the [values.yaml](apidiscovery/values.yaml), especially with the `images.api_discovery_collector` parameter.

The changes may include introducing a new parameter, changing existing parameters

It is recommended to upgrade the collector from time to time.

Make sure values.yaml file is based on your apiconnect provider org, authentication and datasource are set up.

Run the following command to upgrade the collector with the new changes

```
helm template . | kubectl apply -f -
```

## Uninstalling the collector

The collector can be uninstalled using the following comman.  

```
helm template . | kubectl delete -f -
```
