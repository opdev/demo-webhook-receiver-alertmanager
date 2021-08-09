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
$ oc -n openshift-user-workload-monitoring get pod
NAME                                   READY   STATUS    RESTARTS   AGE
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

Get the Cluster IP of the webhook service:

```shell
$ oc get svc my-webhook
NAME         TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
my-webhook   NodePort   172.30.188.12   <none>        8080:31635/TCP   2d16h
```

In the OpenShift console, under "Administration/Cluster Settings", in the
"Global Configuration" tab, click "AlertManager" then "Create Receiver":
- Receiver Name: "my-webhook"
- Receiver Type: "Webhook"
- URL: "http://172.30.188.12:8080/webhook"
- Routing labels:
  - Name: "app"
  - Value: "my-webhook"

![webhook_receiver_config](https://user-images.githubusercontent.com/1616123/128550188-08be8b17-7d6f-429c-a2af-4bda9ed35939.png)

## Observe Result

```shell
$ export POD_NAME=`oc get po | awk '$1 ~ /my-webhook-/ {print $1}'`
$ oc logs -f $POD_NAME
2021/08/06 16:09:26 listening on: :8080

```

For debugging purposed, one can speed up the frequency of alerts being sent
with the `Repeat Interval` and `Group Interval` in the AlertManager config:

![image](https://user-images.githubusercontent.com/1616123/128550417-7c7d35dc-db55-4486-a6c9-e5d9249dab1d.png)



## Helpful resources

- https://zhimin-wen.medium.com/custom-notifications-with-alert-managers-webhook-receiver-in-kubernetes-8e1152ba2c31
- https://prometheus.io/docs/alerting/latest/configuration/
- https://docs.openshift.com/container-platform/4.8/post_installation_configuration/configuring-alert-notifications.html

