# How to set default context to connect Digitalocean cluster

## Install
install `doctl` by following commands
``` sh
    cd ~
    wget https://github.com/digitalocean/doctl/releases/download/v1.94.0/doctl-1.94.0-linux-amd64.tar.gz
    tar xf ~/doctl-1.94.0-linux-amd64.tar.gz
    sudo mv ~/doctl /usr/local/bin
```

## Create an API Token
- go to digitalocean console
- go to API [page](https://cloud.digitalocean.com/account/api/tokens?i=ccedef)
- click `Generate New Token`
- enter token name
- don’t forget to tick Write scope
- click `Generate Token`
- enter this command
```sh
   doctl auth init --context <NAME>
```
    - specific Name to se for uaccess cluste
    - enter the API token that generated above
- switch to created context
``` sh
    doctl auth list
    doctl auth switch --context <NAME>
```
- use `doctl` to load the kube config file from our cluster by
```sh
    doctl kubernetes cluster kubeconfig save use_your_cluster_name
```
> cluster name can be found in digitalocean Kubernetes console and looking for the CLUSTER ID eg. 27fcf299-1242-402f-bb08-01f967a20b4
- after downloaded the kube config file you can validate by
    - to see current context used
    ``` sh
        kubectl config current-context
    ```
    - to get list context
    ``` sh
        kubectl config get-contexts
    ```
    - to set default context
    ``` sh
        kubectl config use-context do-sfo2-example-cluster-01
    ```

## References
- [How to Install and Configure doctl | DigitalOcean Documentation](https://docs.digitalocean.com/reference/doctl/how-to/install/)
- [How to Connect to a DigitalOcean Kubernetes Cluster | DigitalOcean Documentation](https://docs.digitalocean.com/products/kubernetes/how-to/connect-to-cluster/)
