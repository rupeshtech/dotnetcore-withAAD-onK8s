# dotnetcore-withAAD-onK8s

    This example demonstrates step by step to create and deploy a sample MVC application with AAD in kubernetes.

## Prerequisites

1. You have Visual Studio Code.
2. You have an Azure Subscription. [Free $200 Azure Credit](https://azure.microsoft.com/free)
3. You have an image repository (this example uses Azure container registry)

## Create Sample MVC App with `AAD authentication`
```sh
dotnet new mvc -au SingleOrg --client-id "xxxxxxxxxxxxxxxxxxx" --tenant-id "xxxxxxxxxxxxxxxxxxxxxxxx" --domain "xxxxxxxx"
```
Add below code in startup.cs ---> Configure [program.cs](https://github.com/rupeshtech/dotnetcore-withAAD-onK8s/blob/master/SampleMvcAppWithAad/Program.cs)

```csharp
    app.Use((context, next) =>
    {
        context.Request.Scheme = "https";
        return next();
    });
```
```sh
dotnet run
```
### Build `image`
```sh
touch Dockerfile (see DockerFile from SampleApp)
docker build -t myapp:v1 .
docker image ls
```
[DockerFile from SampleApp](https://github.com/rupeshtech/dotnetcore-withAAD-onK8s/blob/master/SampleMvcAppWithAad/Dockerfile)
### Push image to ACR
```sh
az login
az acr login --name acrname
docker tag myapp:v1 acrname.azurecr.io/myapp:v1
docker push acrname.azurecr.io/myapp:v1
```

## Create and Configure `AKS`
Skip this step, If you already have AKS created and configured
1. Create Azure kubernetes Cluster
```sh
az aks create -g <resourceGroupName> --name <kubernetes-cluster-name>  --service-principal <servicePrincipalId> --client-secret <clientSecret>
```
2. Create a public (static) IP address in the resource group MC_resourceGroupName_location and note the dns name, which gives <fqdn>
3. Configure the route traffic to the ingress controller
```sh
helm install stable/nginx-ingress \
    --namespace ingress-basic \
    --set controller.replicaCount=1 \
	--set controller.image.repository= quay.io/kubernetes-ingress-controller/nginx-ingress-controller  \
    --set controller.service.loadBalancerIP="<your static Ip address>"
```
4. Configure a DNS name: For the HTTPS certificates to work correctly, configure an FQDN for the ingress controller IP address. Update the following script with the IP address of your ingress controller and a unique name that you would like to use for the FQDN. (This step is not always necessary but good to be sure)
```sh
# Public IP address of static ip address
IP="<your static IP>"
# Name to associate with public IP address
DNSNAME="<dns name>"
# Get the resource-id of the public ip
PUBLICIPID=$(az network public-ip list --query "[?ipAddress!=null]|[?contains(ipAddress, '$IP')].[id]" --output tsv)
# Update public ip address with DNS name
az network public-ip update --ids $PUBLICIPID --dns-name $DNSNAME
```
5. Add Reidirect URI in Azure ---> AAD -- AppRegistration 
https://<fqdn>/signin-oidc
### create secret to pull image for Azure container registry
```sh
kubectl create secret docker-registry <secret-name> --docker-server=<youracr.azurecr.io> --docker-username=<acrusername> --docker-password=<acr-password> --docker-email=<youremailaddress>
```

### Kubernetes `ingress`
```sh
kubectl create -f ingress.yaml
```
A few things to note:
1. We've tagged the ingress with the annotation nginx.ingress.kubernetes.io/ssl-redirect: "true".
2. backend serviceName should match the name defined in file app-service.yaml
3. Host should be your FQDN (fqdn from create static ip)

### Kubernetes `deployment`
```sh
kubectl create -f app-deployment.yaml
```
A few things to note:
1. image pull secret should be present if container registry is private
2. image should be of format <youracr.azurecr.io>/myapp:v1  (if you are using Azure container registry)

### Kubernetes `service`
```sh
kubectl create -f kubectl create -f app-service.yaml
```
### Check deployment, services and pods

you can run following commands to check deployment, services and pods
```sh
kubectl get ing -n ingress-basic
kubectl get deployment
kubectl get pods
kubectl logs <pod_name> -f
```