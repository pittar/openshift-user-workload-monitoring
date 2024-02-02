# OpenShift User Workload Monitoring

This repository contains sample configuration of [OpenShift User Workload Monitoring](https://docs.openshift.com/container-platform/4.9/monitoring/enabling-monitoring-for-user-defined-projects.html), including:
* Enabling User Workload Monitoring
* Deploying the Grafana operator
* Deploying and configuring Grafana
* Deploying custom dashboards.

## Enable User Workload Monitoring

To [enable user workload monitoring](https://docs.openshift.com/container-platform/4.9/monitoring/enabling-monitoring-for-user-defined-projects.html), you need to edit the *cluster-monitoring-config* ConfigMap in the *openshift-monitoring* namespace, or create the ConfigMap if it doesn't exist.  The important line is `enabableUserWorkload: true` in the `config.yaml` stanza.  It should look something like this (although, it's possible the config might be much longer if other aspects of monitoring have been configured):

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
```

Check to see if this ConfigMap already exists in your cluster:

```
oc get cm cluster-monitoring-config -n openshift-monitoring
```

If it does, then edit the ConfigMap and add the `enableUserWorkload: true` line.  If it doesn't exist, you can create the ConfigMap:

```
oc apply -f manifests/01-config -n openshift-monitoring
```

This will trigger the deployment of a second Prometheus instance, as well as other infrastructure (e.g. Thanos Ruler) to support user workload monitoring.

## Deploy Grafana Operator

Next, deploy the Grafana operator.  This is a *community* operator that is maintained by Red Hat.  It is *not* a supported component of OpenShift.

You can install the Grafana operator through the OpenShift Administrator UI, or, you can execute the following command to deploy the operator into a namespace called "workload-monitoring".  

```
oc apply -k manifests/02-grafana-operator
```

You can deploy this operator into whatever namespace you like, but the example in this repository will use a new namespace called `workload-monitoring`.

Creating the new namespace and installing the Grafana operator will take a minute or two.  Once the operator has been deployed, you can move on to the next step.

## Deploy Grafana Instance and Prometheus DataSource

Next, deploy a `GrafanaDataSource` that is configured to pull data from the Thanos Querier that was deployed as part of Workload Monitoring.  Since this Data Source will require a the token of a `ServiceAccount` with read access to OpenShift Monitoring, the next step does the following:

1. Defines RBAC to allow the Grafana `ServiceAccount` to read metrics.
2. Creates a `GrafanaDataSource` to pull data from Thanos Querier.
3. Creates the `Grafana` instance.
4. Creates a `Secret` that injects the Grafana service account token into the `GrafanaDataSource` config.

To deploy all of the above, run the following command:

```
oc apply -k manifests/03-grafana-instance
```

In a minute or two, you will have an instance of Grafana running that can pull metrics from Prometheus.

## Deploy Grafana Dashboards

Finally, deploy sample Spring Boot and Quarkus dashboards:

```
oc apply -k manifests/04-dashboards
```

In a minute, you should see there are now two dashboards available under the "workload-monitoring" folder (Home -> Workload Monitoring).

If there are `ServiceMonitor` instances setup in developer namespaces and properly configured to scrap metrics from the apps that are deployed, then these dashboards should be able to display those metrics.

In these example dashboards, the Spring Boot dashboard assumes metrics are "tagged" with the key/value pair of "runtime=springboot".  For example:
https://github.com/pittar/spring-petclinic/blob/gitops-tekton/src/main/resources/application.properties#L18

For the Quarkus dashboard, a similar tag is required to set the runtime tag value to "quarkus".  For example:

* https://github.com/pittar-gameoflife/web-ui/blob/main/src/main/resources/application.properties#L3
* https://github.com/pittar-gameoflife/web-ui/blob/main/src/main/java/ca/pitt/demo/gameoflife/ui/config/CustomConfiguration.java#L28

## Example ServiceMonitor

An example `ServiceMonitor` that can scrap metrics from a Spring Boot app might look like this:

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: server
spec:
  selector:
    matchLabels:
      prometheus-metrics: "true"
  endpoints:
  - port: 8080-tcp
    path: /actuator/prometheus
    scheme: http
```

This would assume the `Service` for the application includes the `prometheus-metrics: true` annotation and the port name is `8080-tcp`.

For example: https://github.com/pittar-demos/roadshow/blob/main/gitops/developer/petclinic/base/02-petclinic-svc.yaml

## Example Applications

If you want to deploy a demo app in your cluster that is pre-configured to work with this monitoring stack, you have two options:

### Quarkus

Deploy a Quarkus app that is configured to use the Micrometer Prometheus registry.  The app will expose prometheus metrics at the `/q/metrics` path.  The configuration includes a pre-configured `ServiceMonitor`.  The app will be deployed to the `quarkus-dev` namespace.  Metrics should start appearing in the Quarkus dashboard a minute or two after the app starts running.

```
oc apply -k manifests/05-demo-apps/quarkus/overlays/dev
```

If you also want a second "test" namespace (to see how the 'namespace' drop down works in the dashboard), you can create a `quarkus-test` instance like so:

```
oc apply -k manifests/05-demo-apps/quarkus/overlays/test
```

Application GitHub repo: [Quarkus Prometheus Demo](https://github.com/pittar/quarkus-prometheus-demo)

### Spring Boot

Deploy a Spring Boot app that is configured to use the Micrometer Prometheus registry.  The app will expose prometheus metrics at the `/actuator/prometheus` path.  The configuration includes a pre-configured `ServiceMonitor`.  The app will be deployed to the `petclinic-dev` namespace.  Metrics should start appearing in the Quarkus dashboard a minute or two after the app starts running.

```
oc apply -k manifests/05-demo-apps/springboot/overlays/dev
```

If you also want a second "test" namespace (to see how the 'namespace' drop down works in the dashboard), you can create a `petclinic-test` instance like so:

```
oc apply -k manifests/05-demo-apps/springboot/overlays/test
```

Application GitHub repo: [Spring Pet Clinic - prometheus branch](https://github.com/pittar/spring-petclinic/tree/prom)
