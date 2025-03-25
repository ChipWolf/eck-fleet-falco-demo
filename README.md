# eck-fleet-falco-demo

ECK + Fleet + Falco - Demo

## Running the demo locally

Spin up a kind cluster:

```bash
kind create cluster --config kind-config.yaml
```

This demo uses the [k8saudit](https://github.com/falcosecurity/plugins/tree/main/plugins/k8saudit) Falco plugin.

Note from our [kind config](kind-config.yaml) that our cluster has audit logs enabled as per [audit-policy.yaml](audit-policy.yaml) which is tailored for the k8saudit plugin, and defines the rules about what events should be recorded. The rules shipped with the k8saudit plugin rely on those events. The [webhook-config.yaml](webhook-config.yaml) provides the configuration to send events to a Falco Webhook service. For demonstration purposes only, we use a `NodePort` for the `falco-k8saudit-webhook` service.

Install the Elastic Operator, and wait for it to be ready:

```bash
kubectl create -f https://download.elastic.co/downloads/eck/2.16.1/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.16.1/operator.yaml
kubectl wait --timeout 60s -n elastic-system --for=condition=Ready pod/elastic-operator-0
```

Install the Elastic Stack, and wait for all resources to be green (this may take a few minutes):

```bash
kubectl apply -f manifests/
kubectl wait --timeout 1220s --for='jsonpath={.status.health}=green' kibanas.kibana.k8s.elastic.co/kibana-quickstart
kubectl wait --timeout 1220s --for='jsonpath={.status.phase}=Ready' elasticsearches.elasticsearch.k8s.elastic.co/elasticsearch-quickstart
kubectl wait --timeout 1220s --for='jsonpath={.status.health}=green' agents.agent.k8s.elastic.co/elastic-agent-quickstart
kubectl wait --timeout 1220s --for='jsonpath={.status.health}=green' agents.agent.k8s.elastic.co/fleet-server-quickstart
```

Install Falco and Falcosidekick, configuring the `k8saudit` plugin:

```bash
helm install falco falcosecurity/falco -f falco/values.yaml
```

Get the password for the `elastic` user:

```bash
kubectl get secret elasticsearch-quickstart-es-elastic-user -o json | jq .data.elastic | sed 's/"//g' | base64 -d
```

Go to <https://localhost:30080> in your browser, ignore the self-signed certificate warning, and log into the kibana UI with the above password, and a username of "elastic". Create a data view with an index pattern of `logs-falco*`.

Note the custom rule that has been created in the [falco values file](falco/values.yaml). This rule triggers if a container is spun up with an image that doesn't come from an approved registry. As this is only for demonstration purposes, the rule is limited to containers in the `test` namespace. Let's trigger this rule by spinning up a pod which should be allowed, and one which should be blocked if this rule were enforced at admission:

```bash
kubectl apply -f test-pods/
```

After a while, observe in Kibana that there is a `Detect Unauthorized Container Registry` event in Elastic.

## Teardown

```bash
kind delete cluster
```
