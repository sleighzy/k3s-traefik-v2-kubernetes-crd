# Kubernetes Traefik Ingress Controller CRD

[Kubernetes](https://kubernetes.io/) (k8s) provides the ability for
[Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
to be deployed for directing traffic to services. This means that services can
be exposed outside of the cluster without requiring a new loadbalancer for each
one. An ingress can be used instead that routes traffic to services based on
routing rules, e.g. hostname, path, headers, etc.

[Traefik](https://containo.us/traefik/) is a Cloud Native Edge Router and
reverse proxy that can direct traffic between services based on routing rules.
Traefik provides a Ingress Controller that can be deployed into Kubernetes
clusters for these purposes. Traefik introduced a Kubernetes Custom Resource
Definition (CRD) for
[Ingress Routes](https://docs.traefik.io/providers/kubernetes-crd/), which is
what the configuration in this repository is based on.

## K3s and K3d

[k3s](https://k3s.io/) is a lightweight, certified Kubernetes distribution, for
production workloads from Rancher Labs. k3s installs Traefik, version 1.7, as
the Ingress Controller, and a service loadbalancer (klippy-lb) by default so
that the cluster is ready to go as soon as it starts up. The instructions below
will be deploying a k3s cluster _without_ the default Traefik 1.7 as we want to
deploy this ourselves so that we can use the latest Traefik v2 Kubernetes
Ingress Controller installation.

[k3d](https://github.com/rancher/k3d) is tool developed by the folk at Rancher
to deploy k3s nodes into Docker containers. This provides the means to deploy
server and multiple worker nodes on your local machine, taking up very little
resource, each running within its own container.

k3s (using k3d) will be used as the Kubernetes distribution for the examples in
this repository.

## Deploy k3s cluster using k3d

Run the below command (`sleighzy` is my cluster name) to deploy a 2 work node
cluster. This performs the following:

- mounts a directory on the host machine to
  `/var/lib/rancher/k3s/server/manifests` so that k8s manifest files can be
  dropped in here for automatic deployment.
- mounts a directory from the host machine to `/var/lib/rancher/k3s/storage` as
  this is the default directory k3s stores data in. We can create k8s
  [Persistent Volume Claims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
  and they will be created here on the host machine.
- (optional) publish ports 80 and 443 to the host machine so that we can send
  external web traffic (http and https) to the cluster
- the `--server-arg` arguments will pass `--no-deploy traefik` to k3s when the
  cluster is created so that the default Traefik 1.7 ingress controller is not
  installed

```sh
$ k3d create \
  --name sleighzy \
  --volume /mnt/f/k3s/manifests:/var/lib/rancher/k3s/server/manifests \
  --volume /mnt/f/data/k3s/storage:/var/lib/rancher/k3s/storage \
  --api-port 6550 \
  --publish 80:80 \
  --publish 443:443 \
  --workers 2 \
  --server-arg --no-deploy \
  --server-arg traefik

$ export KUBECONFIG="$(k3d get-kubeconfig --name='sleighzy')"

$ kubectl cluster-info
```

## Install Traefik Kubernetes CRD Ingress Controller

k3s ships with Traefik 1.7 by default so we need to install Traefik 2 separately
using the manifests in this repository as the `--no-deploy traefik` arguments we
used mean that Traefik is not installed.

Apply the manifests in order (prefixed by number) to install the secrets, k8s
CRDs, service, and deployment for Traefik v2. The Traefik documentation for the
new Kubernetes CRD Ingress Controller is somewhat incomplete is places, their
largest example is integrating with LetsEncrypt, and some errors were output to
the logs around missing CRDs. I have fixed this up and the configuration here
should work as expected.

Note that the secrets, and their usage in the deployment, will be required if
you are using Traefik and the integration with LetsEncrypt for automatic
creation of certificates for https services. The configuration here is also for
integrating with GoDaddy for the https certiricates so the configuration may be
different for your provider so refer to the Traefik documentation on this. If
you are not using https and integration with LetsEncrypt then you do not need to
apply the `./002-secrets.yaml` file, and can remove the mounting of those
secrets from the `./005-deployment.yaml` file. The later sections in this README
file will cover the HTTPS integration in greater depth.

- [001-rbac.yaml](./001-rbac.yaml) - CRDs and cluster roles
- [002-secrets.yaml](./002-secrets.yaml) - this is optional, but is needed if
  integrating with LetsEncrypt (depending on your mechanism) as per the examples
  further down, for API keys etc. for your DNS provider
- [003-pvc.yaml](./003-pvc.yaml) - this is optional, but is used when
  integrating with LetsEncrypt as this creates a persistent volume on the host
  machine that is used to store the certificates
- [004-service.yaml](./004-service.yaml) - exposes the container ports for
  traefik
- [005-deployment.yaml](./005-deployment.yaml) - the deployment of the Traefik
  container with the associated mounts for secrets and persistent volume if
  integrating with LetsEncrypt for https certificates

### Traefik Dashboard

Traefik provides a number of dashboards for viewing your services, routes,
middleware, etc. In the current deployment configuration this is not being
exposed outside of the cluster. You can port forward `8080` (the admin port)
from the docker host to the traefik service and then you can navigate to
<http://localhost:8080/dashboard> in your browser to view the Traefik dashboard.

```sh
kubectl port-forward --address 0.0.0.0 service/traefik 8080:8080 -n kube-system
```

## Deploy the whoami service and use Traefik 2 IngressRoute

The Traefik documentation and a number of other resources use the `whoami`
service as an example deployment. Apply the below configuration to deploy the
whoami app and associated service.

```yaml
---
kind: Deployment
apiVersion: apps/v1
metadata:
  namespace: default
  name: whoami
  labels:
    app: whoami

spec:
  replicas: 1
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
        - name: whoami
          image: containous/whoami
          ports:
            - name: web
              containerPort: 80
```

For the Traefik `IngressRoute` the below configuration can be applied to route
http traffic to the `whoami` service if the host header is `whoami.mydomain.io`.

```yaml
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: whoami
  namespace: default

spec:
  entryPoints:
    - web
  routes:
    - match: Host(`whoami.mydomain.io`)
      kind: Rule
      services:
        - name: whoami
          port: 80
```

### Invoke whoami service exposed via Traefik

Running the below command from the Docker host should hit the `whoami` service
as the default port `80` for http traffic is being routed to the k3s server node
(docker container), and the Traefik ingress will be matching the
`whoami.mydomain.io` host header rule. Note, while the Traefik pod will not be
running on the server node (but the docker host port 80 is bound to the k3s server docker
container) k3s will still route this to the Traefik pod as expected due to the
klippy-lb service loadbalancer that is installed by k3s by default.

```sh
$ curl http://localhost/ -H "host:whoami.mydomain.io"
Hostname: whoami-bd6b677dc-56zpq
IP: 127.0.0.1
IP: ::1
IP: 10.42.1.7
IP: fe80::d4b0:c1ff:fe23:dcc3
RemoteAddr: 10.42.2.14:53048
GET / HTTP/1.1
Host: whoami.mydomain.io
User-Agent: curl/7.58.0
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 10.42.1.6
X-Forwarded-Host: whoami.mydomain.io
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: traefik-7c8b9b949f-cws5b
X-Real-Ip: 10.42.1.6
```

## HTTPS with LetsEncrypt

Traefik v2 introduced automated generation of certificates for services when
integrating with [LetsEncrypt](https://letsencrypt.org/). When calling services
with routers that reference the configured certificate resolver for Traefik it
will automatically attempt to generate certificates using LetsEncrypt. Traefik
provides a number of ACME challenger options, and a large number of supported
providers for [DNS-01](https://docs.traefik.io/https/acme/#dnschallenge)
challengers. Traefik will generate new certificates for the services when they
expire.

The configuration in this repository can be used to integrate with GoDaddy for
dns challenges as this is my DNS provider. The `...caserver` argument in for
Traefik in the `./004-deployment.yaml` is for the LetsEncrypt Staging server,
currently commented out, and can be used for initial testing purposes. Note that
staging server will require you to add an intermediate certificate as it is not
a completely trusted chain. The `...acme.storage` argument states where Traefik
should write the certificate information to. The deployment configuration here
uses the persistent volume mounted on the host.

Traefik requires the certificates file to have permissions of 600. If running on
Windows with WSL (Windows Subsystem Linux) directories and files created on the
Windows filesystem will have permissions of 777 and so this will fail. You will
need to update (add if it doesn't exist) your /etc/wsl.conf file to add metadata
to mounted filesystems so that the correct permissions can be set on the file in
WSL. See <https://www.turek.dev/post/fix-wsl-file-permissions/> for more
information.

### Prerequisites for certificate generation

The hostname that you will be calling needs to have been added to your
provider's DNS so that https traffic is routed to your server and onto Traefik.
Traefik will use its APIs to create DNS entries for the certificate challenge,
e.g. `TXT` entry `_acme-challenge.whoami` for the whoami service.

Get the API keys and API secrets may be required by your DNS provider, they are
for my GoDaddy account, so that the APIs can be called by Traefik to create the
required DNS entries for the certificate challenge.

Create base64 encoded strings from the API keys and secrets, and then update the
`./002-secrets.yaml` file with them before applying the file.

```sh
$ echo -n '<my api key>' | base64
xxxxxxxxxxx=
$ echo -n '<my secret>' | base64
xxxxxxxxxx==

# update 002-secrets.yaml and then apply
$ kubectl apply -f 002-secrets.yaml
```

### Route https traffic to the whoami service

Apply the below yaml to create an `IngressRoute` that performs the following:

- accepts traffic from the `websecure` entry point, which was configured as the
  port 443 address when starting Traefik
- uses tls and the godaddy certificate resolver, that was configured using the
  Traefik arguments when starting it, to request an https certificate from
  Godaddy if a certificate does not already exist, or is about to expire
- routes all traffic to the `whoami` service for requests with a host header of
  `whoami.mydomain.io` and a path of `/tls`

Ensure that the prerequistes have been set up first as Traefik will attempt to
retrieve certificates as soon as the `IngressRoute` is created.

```yaml
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: whoami-tls
  namespace: default
spec:
  entryPoints:
    - websecure
  routes:
    - kind: Rule
      match: Host(`whoami.mydomain.io`) && PathPrefix(`/tls`)
      services:
        - name: whoami
          port: 80
  tls:
    certResolver: godaddy
```

Confirm that the service and ingress route have been created.

```sh
$ kubectl get svc,ep,ingress,ingressroute -o wide
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE     SELECTOR
service/kubernetes   ClusterIP   10.43.0.1       <none>        443/TCP   2d19h   <none>
service/whoami       ClusterIP   10.43.226.213   <none>        80/TCP    2d19h   app=whoami

NAME                   ENDPOINTS         AGE
endpoints/kubernetes   172.25.0.2:6550   2d19h
endpoints/whoami       10.42.1.7:80      2d19h


NAME                                             AGE
ingressroute.traefik.containo.us/whoami    2d19h
ingressroute.traefik.containo.us/whoami-tls   2d16h
```

Accessing <https://whoami.mydomain.io/tls> in your browser should now display
similar information to that shown when using the previous http url.

```sh
Hostname: whoami-bd6b677dc-56zpq
IP: 127.0.0.1
IP: ::1
IP: 10.42.1.7
IP: fe80::d4b0:c1ff:fe23:dcc3
RemoteAddr: 10.42.2.14:48566
GET / HTTP/1.1
Host: whoami.mydomain.io
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.102 Safari/537.36 Edge/18.19041
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.5
Cookie: _forward_auth=jRmJ5On5wAvIdKJ-B22_gFu0J9mMXnaxPQqUozvQYxw=|1584999044|
Upgrade-Insecure-Requests: 1
X-Forwarded-For: 10.42.1.6
X-Forwarded-Host: whoami.mydomain.io
X-Forwarded-Port: 443
X-Forwarded-Proto: https
X-Forwarded-Server: traefik-7c8b9b949f-cws5b
X-Forwarded-User:
X-Real-Ip: 10.42.1.6
```
