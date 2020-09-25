# IllumiDesk Helm Chart

:warning: Draft Status :warning:

## Overview

Use this [helm chart](https://helm.sh/docs/topics/charts/) to install IllumiDesk into your Cluster. This chart depends on the [jupyterhub](https://zero-to-jupyterhub.readthedocs.io/en/latest/).

This setup pulls images defined in the `illumidesk/values.yaml` file from `DockerHub`. To push new versions of these images or to change the image's tag(s) (useful for testing), then follow the instructions in the [build images section](#build-images).  

## Requirements

- [helm >= v3](https://github.com/kubernetes/helm)
- [Standard Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- (Optional)[AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- (Optional)[Amazon EKS vended KUBECTL](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html)
- (Optional)[EKSCTL](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html)
- (Optional) [Docker](https://docs.docker.com/get-docker/)
- (Optional) [Python 3.6+](https://www.python.org/downloads/)
  
## Assumptions

1. You have a cluster and it has been setup properly
2. You have helm and kubectl installed
3. You have updated your kube-config to use the appropriate context
   * To view contexts
     * ```kubectl config get-contexts```
   * To use different context
     * ```kubectl config use-context {name of context}``` 

## Installation of Illumidesk Helm Chart 

1. Verify that helm exists: ```helm list```
2. Create a namespace for your helm chart
   * ```kubectl create namespace $NAMESPACE```
3. Create a copy of _**example-config/values.yaml.example**_ file and update it with your setup. 
    * NOTE to get a token use  ``` openssl rand -hex 32``` 
    * Here is an example of a basic load balancer setup setup
        ```bash 
                jupyterhub:
                    proxy:
                        secretToken: your_token
                        service:
                        type: LoadBalancer
                albIngressController:
                enabled: false
                allowExternalDNS: 
                enabled: false
                allowEFS: 
                enabled: false
        ```
    * Here is another example of a basic setup using nodeport
        ```bash
            jupyterhub:
            proxy:
                secretToken: your_token
                service:
                type: NodePort
                nodePorts:
                    http: 30791
                    https: 30792
            albIngressController:
            enabled: false
            allowExternalDNS:
            enabled: false
            allowEFS:
            enabled: false
        ```
4. (Optional) if you are using using eks you can also provide details to get aws resources

    * If you use aws can add the following values as well:
    * | Key         | Description                                      | How to get value  |
      | ----------- | ------------------------------------------------ | -------------------------- |
      |   awsAccessKey     | Access Key created for your account       | ```aws configure get aws_access_key_id``` |
      | awsSecretToken     | Secret token provided by aws              | ```aws configure get aws_secret_access_key``` |
      | secretToken     | Secret token for proxy generated by oppenssl           | ```openssl rand -hex 32```    |
      | clusterName     | name of EKS cluster created from eksctl            | ```eksctl get cluster ``` |
      | clusterVPC     | VPC ID of vpc generated by eksctl for your cluster         | ```aws eks describe-cluster --name cluster_name --query "cluster.resourcesVpcConfig.vpcId" --output text```
      | awsRegion     | aws region where your cluster is located              | ```aws configure region```
      | subnets     | subnets that are part of your cluster vpc. At least 2 required            | ```aws ec2 describe-subnets --filter "Name=vpc-id,Values=vpc-xxxxxxxxxxxx" "Name=tag:Name,Values=*Public*"  --query "Subnets[*].SubnetId"``` |
      | domainFilter     | your aws route 53 hosted zonezone              | `aws route53 list-hosted-zones --query "HostedZones[*].Name"`|
      | txtOwnerID     | identifies externalDNS instance            | set to a unique value that doesn't change during the lifetime of your cluster |
      | efs     | efs file system url           | ```Go to console->EFS, Find the file system whose mounted targets are in the same vpc as the cluster``` |
      |  certificate_arn | certificate managaged by aws | ``` Go to console->Certificate Manager, Request for a certificate```|
5. Add the following repo and then update helm repos
    * ``` Helm repo add https://illumidesk.github.io/helm-chart/```
    * ``` Helm repo update ```
6. Install the illumidesk helm chart with the following
    * NOTE: $RELEASE and $NAMESPACE are placeholders so make sure to update them  
    * ```helm upgrade --install $RELEASE illumidesk --namespace $NAMESPACE --values path/to/customized values file```
7. Confirm that you can access the site
    * For nodeport you will need to use your one of your node ips and also the port you defined in your values file
        * Use the following command to list out your nodes
            * ``` kubectl get nodes -o wide ```
        * Open up your browser and use the **NODE_IP:NODE_PORT**
    * For load balancer you will need to get the external IP for proxy-public
        * Use this command to view your namespaces
          * ``` kubectl get svc -n $NAMESPACE```
        * Open up your browser and paste in the load balancer dns that is the external ip of proxy-public
    * For Application Load Balancer, you must have specified the host in your values file
        * Verify the dns has propgates your domain
          ``` dig $HOST ```
        * Open up your browser and paste the value for your host


## Cleanup

```bash
    helm delete <release name> --purge
```

## Images

### Singleuser Images

By default this chart sets the `singleuser` image to [illumidesk/base-notebook](https://hub.docker.com/r/illumidesk/base-notebook). However, any image maintained in the [illumidesk/docker-stacks](https://github.com/illumidesk/docker-stacks) repo is compatible with this chart.

The `illumidesk/docker-stacks` images are based off of the `jupyterh/docker-stacks` conventions. You can therefore use any of the images in the `jupyterh/docker-stacks` repo which are [also available in dockerhub](https://hub.docker.com/u/jupyter).

To set an alternate image for end-users, update the `singleuser.image` key in the `illumidesk/values.yaml` file.

### JupyterHub Images

There are two Dockerfiles to create two version of the JupyterHub image (`illumidesk/jupyterhub`):

- `illumidesk/jupyterhub`: standard JupyterHub image that uses Python 3.8 and installs the illumidesk package. The illumidesk package contains customized authenticators and spawners.
- `illumidesk/k8s-hub`: inherits from the above image and defines the `NB_USER`, `NB_UID`, and `NB_GID` to run the container.

#### Quick Build/Push

    make build-push-jhubs

This command creates requirements.txt with `pip-compile`, builds docker images, and pushes them to the DockerHub registry.

Enter `make help` for additional options.

### The Hard Way

1. Setup virtualenv:

```bash
    virtualenv -p python3 venv
    source venv/bin/activate
    python3 -m pip install dev-requirements.txt
```

1. Build requirements.txt:

```bash
    pip-compile images/jupyterhub/requirements.in
```

> **Note**: The above command will overwrite the existing requirements.txt file.

2. Build the base JupyterHub image (illumidesk/jupyterhub:py3.8):

```bash
    docker build -t illumidesk/jupyterhub:py3.8 \
      images/jupyterhub/.
```

3. Build the JupyterHub Kubernetes image (illumidesk/k8s-hub:py3.8):

```bash
    docker build -t illumidesk/k8s-hub:py3.8 -f \
      images/jupyterhub/Dockerfile.k8s \
      images/jupyterhub/.
```

4. Push images to registry (DockerHub by default):

```bash
    docker push illumidesk/jupyterhub:py3.8
    docker push illumidesk/k8s-hub:py3.8
```

5. Update `jupyterhub.image.name` with image name. The image name should include the full image namespace and tag.

6. Install IllumiDesk with ```helm``` as inatructed in the first section.





