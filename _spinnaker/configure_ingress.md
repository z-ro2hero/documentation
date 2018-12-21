---
layout: post
title: Exposing Spinnaker
order: 30
---
{:.no_toc}
* This is a placeholder for an unordered list that will be replaced with ToC. To exclude a header, add {:.no_toc} after it.
{:toc}

### DNS Preparation

We'll be setting up two CNAME entries in our DNS in this example; we won't be
able to actually configure the DNS until we get an A record from AWS after
creating the LoadBalancer, but we'll need to select the names in order to configure
the LoadBalancer.  For this tutorial, we've selected `demo.armory.io` to be
our Deck service (the UI), and `gate.demo.armory.io` to be our Gate service
(the API).

## Exposing Spinnaker on EKS
### Create a LoadBalancer service

Note: This guide assumes you’re deploying Spinnaker on Kubernetes using the Distributed deployment type with Halyard.

While there are many ways to expose Spinnaker, we find the method described in this post to be the easiest way to get started. If your organization has other requirements, this post may be helpful as you start working through the process.

First, we’ll start by creating LoadBalancer Services which will expose the API (Gate) and the UI (Deck) via a Load Balancer in your cloud provider. We’ll do this by running the commands below and creating the spin-gate-public and spin-deck-public Services.
NAMESPACE is the Kubernetes namespace where your Spinnaker install is located. Halyard defaults to spinnaker unless explicitly overridden.

```
export NAMESPACE={namespace}
kubectl expose service -n ${NAMESPACE} spin-gate --type LoadBalancer \
  --port 8084 \
  --target-port 8084 \
  --name spin-gate-public
kubectl expose service -n ${NAMESPACE} spin-deck --type LoadBalancer \
  --port 443 \
  --target-port 9000 \
  --name spin-deck-public
```

Once these Services have been created, we’ll need to update our Spinnaker deployment so that the UI understands where the API is located. To do this, we’ll use Halyard to override the base URL for both the API and the UI and then redeploy Spinnaker.

```
export NAMESPACE={namespace}
export API_URL=$(kubectl get svc -n $NAMESPACE spin-gate-public -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
export UI_URL=$(kubectl get svc -n $NAMESPACE spin-deck-public -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
hal config security api edit --override-base-url http://${API_URL}:8084
hal config security ui edit --override-base-url http://${UI_URL}:9000
hal deploy apply
```

### Secure with SSL on EKS

This tutorial presumes you've already created a certificate in the AWS Certificate Manager.

First get the certificate arn and run
```
export ACM_CERT_ARN="arn:::::your:cert"
```

Edit the LoadBalancer service `spin-gate-public` and  `spin-deck-public` we will include 3 annotations for each.

```
kubectl annotate svc -n ${NAMESPACE} spin-gate-public service.beta.kubernetes.io/aws-load-balancer-backend-protocol=http
kubectl annotate svc -n ${NAMESPACE} spin-gate-public service.beta.kubernetes.io/aws-load-balancer-ssl-cert=${ACM_CERT_ARN}
kubectl annotate svc -n ${NAMESPACE} spin-gate-public service.beta.kubernetes.io/aws-load-balancer-ssl-ports=80,443

kubectl annotate svc -n ${NAMESPACE} spin-deck-public service.beta.kubernetes.io/aws-load-balancer-backend-protocol=http
kubectl annotate svc -n ${NAMESPACE} spin-deck-public service.beta.kubernetes.io/aws-load-balancer-ssl-cert=${ACM_CERT_ARN}
kubectl annotate svc -n ${NAMESPACE} spin-deck-public service.beta.kubernetes.io/aws-load-balancer-ssl-ports=80,443
```

### Update the Internal URLS in Spinnaker
We’ll need to update the internal URLs (Deck will complain about trying to call out to an HTTP resource from an HTTPS request). Update the URLs like we did before, but changing the protocols to https:

```
  hal config security api edit --override-base-url https://gate.demo.armory.io
  hal config security ui edit --override-base-url https://demo.armory.io
  hal deploy apply
```


### Enabling sticky sessions

If you're Armory Spinnaker installation will be using [authentication](https://docs.armory.io/install-guide/auth/) and you expect to scale the API server (Gate) beyond more than one instance you'll want to enable sticky sessions. This will ensure that clients will connect and authenticate with the same server each time. Otherwise, you may be forced to reauthenticate if you get directed to a new server. To enable sticky sessions, you'll want to enable session affinity on the Gate service created above.

```
kubectl patch -n ${NAMESPACE} service/spin-gate-public --patch '{"spec": {"sessionAffinity": "ClientIP"}}'
```

For more details about session affinity, see the Kubernetes documentation on [Services](https://kubernetes.io/docs/concepts/services-networking/service/).


## Exposing Spinnaker on GKE with Ingress
### Setting up HTTP Load Balancing with Ingress

GKE has a “built-in” ingress controller and that's what we will use.

First create a file called basic-ingress.yaml and paste it the following

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: basic-ingress

spec:
  rules:
  - host: demo.armory.io
    http:
      paths:
      - backend:
          serviceName: spin-deck
          servicePort: 9000
        path: /
  - host: gate.demo.armory.io
    http:
      paths:
      - backend:
          serviceName: spin-gate
          servicePort: 8084
        path: /
```

Then apply this
`kubectl apply -f basic-ingress.yaml`

Find out the external IP address of the load balancer serving your application by running:
`kubectl get ingress basic-ingress`


Output:

```
NAME            HOSTS                                       ADDRESS         PORTS     AGE
basic-ingress   demo.armory.io, gate.demo.armory.io         203.0.113.12    80        2m
```

Note: It may take a few minutes for GKE to allocate an external IP address and set up forwarding rules until the load balancer is ready to serve your application. In the meanwhile, you may get errors such as HTTP 404 or HTTP 500 until the load balancer configuration is propagated across the globe.

You need to update your DNS records to have the demo.armory.io host point to the IP address generated.

After doing that you can visit http://demo.armory.io:9000/ to view spinnaker.

### Secure with SSL on GKE
To enable SSL and configure your certificates you can follow this guide:
[https://cloud.google.com/kubernetes-engine/docs/how-to/ingress-multi-ssl](https://cloud.google.com/kubernetes-engine/docs/how-to/ingress-multi-ssl)