# Demo Webhook Receiver for AlertManager in OpenShift

## Goal

- Run a simple webhook receiver which outputs the alerts received to the log
output
- Configure a custom Alert in Prometheus
- Configure a new receiver in AlertManager to handle the alert and send it to a
webhook

## Run a webhook receiver

```shell
$ oc apply -f kubernetes_manifests/webhook-deployment.yaml
deployment.apps/my-webhook created
service/my-webhook created
```

This runs the webhook receiver in Openshift.

### Explanations / details

The code is in `webhook-receiver.go`. It's a simple webserver exposing an
endpoint `/webhook` and listening on port 8080.

It leverages the AlertManager's
[template module]( https://godoc.org/github.com/prometheus/alertmanager/template#Data).
More precisely, the `Data` struct can be used to parse the alerts.

The webhook simply outputs information about the alerts to stdout every time a
POST request is received.

You can edit the code and build your own image by adapting the following
commands:

```shell
$ export QUAY_USERNAME="username"
$ docker build quay.io/$QUAY_USERNAME/my-demo-webhook:0.0.1
$ docker push quay.io/$QUAY_USERNAME=/my-demo-webhook:0.0.1
$ sed -e "s/quay.io\/mgoerens\/demo-webhook-receiver-alertmanager.*$/quay.io\/$QUAY_USERNAME\/my-demo-webhook:0.0.1/g" kubernetes_manifests/webhook-deployment.yaml 
```

## Configuration of Prometheus and AlertManager in OpenShift

Using:

```shell
$ oc version
Client Version: 4.8.2
Server Version: 4.8.0-fc.6
Kubernetes Version: v1.21.0-rc.0+fc33082
```

### [Enable monitoring for user-defined project](https://docs.openshift.com/container-platform/4.8/monitoring/enabling-monitoring-for-user-defined-projects.html)

```shell
$ oc -n openshift-user-workload-monitoring get pod
No resources found in openshift-user-workload-monitoring namespace.
$ oc -n openshift-monitoring apply -f kubernetes_manifests/cluster-monitoring-config.yaml 
configmap/cluster-monitoring-config created
$ oc -n openshift-user-workload-monitoring get podNAME                                   READY   STATUS    RESTARTS   AGE
prometheus-operator-6b56db9975-zbpf8   2/2     Running   0          12s
prometheus-user-workload-0             5/5     Running   1          9s
prometheus-user-workload-1             5/5     Running   1          9s
thanos-ruler-user-workload-0           3/3     Running   0          5s
thanos-ruler-user-workload-1           3/3     Running   0          5s
```

### Add a custom Prometheus Alert

```shell
$ oc apply -f kubernetes_manifests/prometheus-rule.yaml
prometheusrule.monitoring.coreos.com/example-alert created
```

The alert is now visible in the OpenShift console under "Monitoring/Alerting"
in the "AlertRules" tab. Clear all filters, then filter per "Source/User".
Their should be only 1 alert rule called `ExampleAlert` in `Firing` state.

In the "Alerts" tab, the `ExampleAlert` should also appear as it's in `Firing`
state.

### Add a custom Receiver

Expose the webhook service and retrieve its external endpoint:

```shell
$ oc expose svc my-webhook
route.route.openshift.io/my-webhook exposed
$ oc get route
NAME         HOST/PORT                                         PATH   SERVICES     PORT   TERMINATION   WILDCARD
my-webhook   my-webhook-default.apps.my-cluster.opl.opdev.io          my-webhook   http                 None
```

In the OpenShift console, under "Administration/Cluster Settings", in the
"Global Configuration" tab, click "AlertManager" then "Create Receiver":
- Receiver Name: "my-webhook"
- Receiver Type: "Webhook"
- URL: "my-webhook-default.apps.my-cluster.opl.opdev.io"
- Routing labels:
  - Name: "app"
  - Value: "my-webhook"

## Observe Result

```shell
$ export POD_NAME=`oc get po | awk '$1 ~ /my-webhook-/ {print $1}'`
$ oc logs -f $POD_NAME
2021/08/06 16:09:26 listening on: :8080

```

## Helpful resources

- https://zhimin-wen.medium.com/custom-notifications-with-alert-managers-webhook-receiver-in-kubernetes-8e1152ba2c31
- https://prometheus.io/docs/alerting/latest/configuration/
- https://docs.openshift.com/container-platform/4.6/post_installation_configuration/configuring-alert-notifications.html

