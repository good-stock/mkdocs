# Kustomize

# How to use Kustomize to build separate environments
<p align="center">
    <img src="/assets/images/kustomize/1.png">
</p>
first of all, i stored all config files in the `k8s` folder. and let’s start from the basic `Kustomize` structure
- `base` - this is the blueprint folder that store all the main config file that will be overwritten later

**_This file use to determine what config files are going to use_**
``` yaml title="kustomization.yaml"
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization

    resources:
    - deployment.yaml
    - namespace.yaml
    - config-map.yaml
    - service.yaml
```
- `overlays` - this is the folder for storing the overridden files. we will use this folder to separate environments like `dev, beta, prod` as you seen above.
    - `beta`  - this is an example separated environment folder I will create overridden file
    - **_This file we can use many resources commands from Kustomize to override the config retrospectively current environment_**
    ``` yaml title="kustomization.yaml"
        apiVersion: kustomize.config.k8s.io/v1beta1
        kind: Kustomization

        nameSuffix: -beta

        commonLabels:
        app: tenant-service-beta

        configMapGenerator:
        - behavior: replace
            envs:
            - .env
            name: tenant-service-config

        images:
        - name: tenant-service-image
        newName: goodstockdev/tenant-service
        newTag: 0.1.0-beta.6

        resources:
        - ../../base
    ```
    - `nameSuffix` - will add `-beta` in every places on our resources config such as
    ``` yaml
        apiVersion: v1
        kind: Service
        metadata:
        name: tenant-service >> wil be tenant-service-beta
        spec:
        selector:
            app: tenant-service
        ports:
            - name: http
            port: 80
            targetPort: 3001
    ```
    - note if we provide configMapGenerator as line 8, don’t forget to provide .env that will overwritten
    - actually we can use patches to override a whole config file as below example
    ``` yaml
        patches:
        - path: deployment.yaml
    ```
    - and we just provide the path file which in this case would be `deployment.yaml`
    <p align="center">
        <img src="/assets/images/kustomize/2.png">
    </p>
    - you can take a look more details on this [repo](https://github.com/julie-ng/cloud-architecture-review/blob/main/manifests/overlays/dev/kustomization.yaml)
- after we provided `kustomize` config files already, we can run following commands to build or delete. 
```sh
kustomize build beta | kubectl apply -f -
```
- use this to delete all just build
```sh
kustomize build beta | kubectl delete -f -
```
- don’t forget to enter to the specific environment folder we are going to build in this case would be `overlays->beta`