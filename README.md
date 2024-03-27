# Installing Knative Serving

This example demonstrates how to install Knative serving on a Kubernetes cluster with Kourier as its networking layer.

This demo accompanies the presentation given at the Austin Kubernetes meetup on 2024-03-28:

https://docs.google.com/presentation/d/15QwuM3w88Ij-zaaGGa70tP8ti5n8ox0SP6ii5vsI170/edit?usp=sharing

## Prerequisites

- A Kubernetes cluster with `type: LoadBalancer` support for `Service` objects (e.g. GKE, EKS, AKS, etc.) Kind / Minikube could work locally with MetalLB or similar
- `kubectl`
- `cert-manager` supported DNS provider (for automating `DNS01` challenges) https://cert-manager.io/docs/configuration/acme/dns01/#supported-dns01-providers

Note: All versions of Knative and cert-manager will likely change after this document is written. Please consult the official documentation for the most up-to-date information. As of this writing, the versions used are:

- Knative: v1.13.3
- `cert-manager`: v1.14.4

Links:

- Knative: https://knative.dev/docs/
- `cert-manager`: https://cert-manager.io/docs/

## Steps

### Install Knative Operator

The Knative operator is a Kubernetes operator that manages the lifecycle of Knative components. It'll enable you to install Knatve and configure the install with custom resources. To install the Knative operator, run the following command:

```bash
kubectl apply -f https://github.com/knative/operator/releases/download/knative-v1.13.3/operator.yaml
```


### Use the Knative Operator to install Knative Serving

Once the operator is in place, you can use it to install Knative serving. To do this we'll create a `KnativeServing` resource with Kourier support.

```yaml
apiVersion: operator.knative.dev/v1beta1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  ingress:
    kourier:
      enabled: true
  config:
    network:
      ingress-class: "kourier.ingress.networking.knative.dev"
```

Save the above in a file named `knative-serving.yaml` and apply it to the cluster:


```bash
kubectl apply -f knative-serving.yaml
```

### Install `cert-manager`

Next, we'll want to install `cert-manager`.

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml
```

### Setup a Custom Domain

Once Knative is installed, we'll setup a custom domain for serving our Knative services. Since Knative services take the form:

`service.namespace.example.com`

We'll use `example.com` as our domain, and we'll also assume a single namespace named `preview` will contain our Knative services.

#### Create a Wildcard DNS Record
First, we'll create a wildcard DNS record for our domain to point to the IP address of the Kourier Load Balancer. Grab the external IP address from the `Service` with `kubectl`:

```
❯ kubectl get service  -n knative-serving -o wide kourier
NAME      TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                      AGE   SELECTOR
kourier   LoadBalancer   10.0.0.20        192.168.1.10    80:31407/TCP,443:32432/TCP   34h   app=3scale-kourier-gateway
```

In this example, the IP is `192.168.1.10`. So we'll setup the dns record `*.preview.example.com` to point to Kourier:

```
*.preview.example.com. 300 IN	A	192.168.1.10
```

#### Configure Knative Serving to use the Custom Domain

Next we'll configure Knative Serving to use `example.com` as our custom domain. To do this, edit the `config-domain` config map in the `knative-serving` namespace:

```bash
kubectl patch --namespace knative-serving configmap config-domain -p '{"data": {"example.com": ""}}'
```


### Configure `cert-manager` to Issue Certificates for the Custom Domain

We'll now configure Knative and `cert-manager` to issue wildcard certificates per-namespace for our custom domain. We can thus use a single `DNS01` challenge and one certificate to cover any service in the `preview` namespace.

#### Configure `cert-manager` to Issue Certificates with `DNS01` Challenges

First, we'll configure `cert-manager` to issue certificates using the `DNS01` challenge. To do this, we'll create a `ClusterIssuer` resource. `cert-manager` supports several different DNS providers for automating `DNS01` challenges. In this example, we'll describe a setup for DigitalOcean. Consult the `cert-manager` documentation for other providers.

Here's an example `ClusterIssuer` resource for DigitalOcean:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: digitalocean-dns
  namespace: cert-manager
data:
  access-token: "MY TOKEN HERE"
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns-issuer
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: YOUR EMAIL HERE
    privateKeySecretRef:
      name: letsencrypt-dns-issuer
    solvers:
    - dns01:
        digitalocean:
          tokenSecretRef:
            name: digitalocean-dns
            key: access-token
```

Modify the above example to suit your needs, save it into a file called `lets-encrypt-dns-issuer.yaml`, and apply it to the cluster:

```bash
kubectl apply -f lets-encrypt-dns-issuer.yaml
```

#### Install Knative `net-certmanager` Controller

Once we have `cert-manager` up and running, we can install Knative's `net-certmanager` controller. This controller links Knative with our `ClusterIssuer` resource which allows Knative to request certificates on our behalf.

```
kubectl apply -f https://github.com/knative/net-certmanager/releases/download/knative-v1.13.0/release.yaml`
```

#### Configure Knative Serving to Use `cert-manager`

Finally, we'll configure Knative Serving to use `cert-manager` to issue certificates for our custom domain. To do this, we'll edit the `config-certmanager` `ConfigMap` in the `knative-serving` namespace:

```bash
kubectl edit --namespace knative-serving configmap config-certmanager
```

This `ConfigMap` will contain lots of defaults. What we want to do is add our domain config in the `data:` section:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-certmanager
  namespace: knative-serving
  labels:
    networking.knative.dev/certificate-provider: cert-manager
data:
  _example.com: |
    lots_of_stuff: "here"
  issuerRef: |
    kind: ClusterIssuer
    name: letsencrypt-dns-issuer
```

Save the `ConfigMap` and Knative will now know to use `cert-manager` for issuing certificates. We'll next want to identify the namespaces for which we would like to issue certs. This is highly configurable, but for now we'll simply tell Knative to only issue certs for the `preview` namespace. This will be done with the `config-network` `ConfigMap`:

```bash
kubectl patch --namespace knative-serving configmap config-network -p '{"data": {"namespace-wildcard-cert-selector": "{\"matchExpressions\": [{\"key\":\"kubernetes.io/metadata.name\", \"operator\": \"In\", \"values\":[\"preview\"]}]}"}}'
```

Consult the Knative documentation for more advanced rules and `Namespace` selectors.

#### Enable Automatic Certificate Provisioning

Finally, we'll enable automatic certificate provisioning. We'll also redirect all HTTP requests to HTTPS. To do this, we'll once again edi the `config-network` `ConfigMap` in the `knative-serving` namespace:

```bash
kubectl edit --namespace knative-serving configmap config-network
```

Add the `external-domain-tls` and `http-protocol` to the `data:` section:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-network
  namespace: knative-serving
data:
  external-domain-tls: Enabled
  http-protocol: Redirected
  # LOTS of example and default config here!
  ...
```

### Deploy a Knative Service

Now that we have everything in place, we can deploy a Knative service. Knative provides a simple service that does
nothing but return 200s, but it's a good starting point to verify that everything is working as expected.

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: autoscale-go
  namespace: preview
spec:
  template:
    metadata:
      annotations:
        # Target 10 in-flight-requests per pod.
        autoscaling.knative.dev/target: "10"
    spec:
      containers:
      - image: ghcr.io/knative/autoscale-go:latest
```

Save the above in a file called `autoscale-go.yaml` and apply it to the cluster:

```bash
kubectl apply -f autoscale-go.yaml
```

### Verify Everything is Working


We can look at the `Routes` objects in the `preview` namespace to see the URL for our service:

```bash
kubectl get routes.serving.knative.dev -n preview

❯ kubectl get routes.serving.knative.dev -n preview
NAME           URL                                             READY   REASON
autoscale-go   https://autoscale-go.preview.example.com        True
```


We can then use `curl` to verfiy that our service is up and running:

```bash
curl https://autoscale-go.preview.example.com -vv
```

We can also verify that Knative scaled up pods to handle our requests

```bash
❯ kubectl get po -n preview
NAME                                            READY   STATUS    RESTARTS   AGE
autoscale-go-00001-deployment-f84477d8f-b87pz   2/2     Running   0          10s
```

And after some time we can verify that Knative scaled down the pods:

```bash
sleep 80
❯ kubectl get po -n preview
NAME                                            READY   STATUS        RESTARTS   AGE
autoscale-go-00001-deployment-f84477d8f-b87pz   2/2     Terminating   0          84s

sleep 20

❯ kubectl get po -n preview
No resources found in preview namespace.
```