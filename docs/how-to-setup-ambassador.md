# Setup ambassador edge stack

## Add the Repo:
- `helm repo add datawire https://app.getambassador.io`
- `helm repo update`
 
## Create Namespace and Install:
- `kubectl create namespace ambassador && \`
- `kubectl apply -f https://app.getambassador.io/yaml/edge-stack/3.7.2/aes-crds.yaml`
- `kubectl wait --timeout=90s --for=condition=available deployment emissary-apiext -n emissary-system`
- `helm install edge-stack --namespace ambassador datawire/edge-stack && \`
- `kubectl -n ambassador wait --for condition=available --timeout=90s deploy -lproduct=aes`

- go to [Ambassador Dashboard](https://app.getambassador.io/cloud/home/dashboard)
- go to service menu
- click  add service
- name token
- save token and save to secure place
- create `values.yaml` file
``` yaml title="values.yaml"
emissary-ingress:
createDefaultListeners: true
agent:
    cloudConnectToken: "OTIxOGY2YjQtZTEwMS00NzQxLWIwZTAtZGE3NjQ0YWE4MDU5OklrcmVwM0dGQnBFNGlrZmlSSk1IdkxYTExncndiaWtTTDVFQw=="
resources:
    requests:
    cpu: "100m"
    memory: "100Mi"
    limit:
    cpu: "200m"
    memory: "300Mi"
replicaCount: 1
```
- run this commands 
``` sh
helm upgrade edge-stack --namespace ambassador datawire/edge-stack --values values.yaml
```
``` sh
kubectl -n ambassador wait --for condition=available --timeout=90s deploy -lproduct=aes
```
- when token has been synced, we visit to the [abbassador service](https://app.getambassador.io/cloud/services)
- we should see the list of ambassador services here
- and we can run this `kubectl -n ambassador get svc edge-stack` to get our gateway IP.

## Setup TLS
- map dns by going to your domain provider that you bought 
- and set up something like this
```
pi.goodsstock.app       A      1 hour    139.59.221.20
```
- run this command
``` sh
kubectl apply -f https://github.com/jetstack/cert-manager/releases/latest/download/cert-manager.crds.yaml
```
- prepare needed files following below
- `certificate-issuer.yaml` (note as I was developed `cert-manager.io/v1alpha2` not working so I kept `cert-manager.io/v1` instead)
``` yaml title="certificate-issuer.yaml"
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
name: letsencrypt-prod
spec:
acme:
    # Replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: goodstockdev@gmail.com
    # ACME URL, you can use the URL for Staging environment to Issue untrusted certificates
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
    # Secret resource that will be used to store the account's private key.
    name: issuer-account-private-key
    solvers:
    # Define the solver to perform HTTP-01 challenge
    - http01:
        ingress:
        class: nginx
    selector: {}
```
``` yaml title="certificate.yaml"
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
name: api.goodsstock.app	
# Cert-manager will put the resulting Secret in the same Kubernetes 
# namespace as the Certificate. You should create the certificate in 
# whichever namespace you want to configure a Host.
spec:
secretName: api.goodsstock.app	
issuerRef:
    # Name of ClusterIssuer
    name: letsencrypt-prod
    kind: ClusterIssuer
dnsNames:
- api.goodsstock.app	
```
``` yaml title="acme-challenge.yaml"
apiVersion: getambassador.io/v2
kind: Mapping
metadata:
name: acme-challenge-mapping
spec:
prefix: /.well-known/acme-challenge/
rewrite: ""
service: acme-challenge-service
---
apiVersion: v1
kind: Service
metadata:
name: acme-challenge-service
spec:
ports:
- port: 80
    targetPort: 8089
selector:
    acme.cert-manager.io/http01-solver: "true"
```
``` yaml title="host.yaml"
apiVersion: getambassador.io/v2
kind: Host
metadata:
name: api.goodsstock.app
spec:
hostname: api.goodsstock.app
acmeProvider:
    authority: 'https://acme-v02.api.letsencrypt.org/directory'
    email: goodstockdev@gmail.com
tlsSecret:
    name: api.goodsstock.app # The secretName defined in your Certificate resource
```
- run `kubectl apply -f ${all of above config}` command to apply config above
- you can use `k9s` to debug if it works
    - search host by this `:host`
    - search certificate by this `:certificates` 
    - search certificate request by this `:certificaterequest`

## Mapping Gateway
- assume that you apply app config files already such as `deployment.yaml`, `service.yaml`, `config-map.yaml` and etc.
- consider the `service.yaml` file. you have to make sure this file run because gateway will be used it
``` yaml
ports:
    - name: http
        port: 80
        targetPort: 3001
```
    - note  port:80  is needed since our gateway listen to it
- go to [Ambassador service page](https://app.getambassador.io/cloud/services)
- select the service you want to map
- input the host matching which default to `*`
- input the path matching eg. `/tenant-service/`
- automatic retry, rate-limit can be set here as well (which i didn’t try)
- click generate mapping
- select YAML tab
- copy all of code and paste to your mapping file eg. `tenant-mapping.yaml`
- run 
``` sh 
kubectl apply -f tenant-mapping.yaml
```
- and re-check on the [Ambassador service](https://app.getambassador.io/cloud/services) console.
