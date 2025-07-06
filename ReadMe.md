# Play.Infra 
Play Economy Infrastructure components

## Add GitHub Package source 

```bash 
owner="dotnetmicroservice001"
gh_pat="[YOUR_PERSONAL_ACCESS_TOKEN]"

dotnet nuget add source \
  --username "$owner" \
  --password "$gh_pat" \
  --store-password-in-clear-text \
  --name github \
  "https://nuget.pkg.github.com/$owner/index.json"
``` 

## Creating Azure Resource Group 
```bash
export appname=playeconomy-01
az group create --name $appname --location westus
```

## Creating CosmosDB Account 
```bash
export appname=playeconomy-01
export db=playeconomy-01-db
az cosmosdb create --name $db --resource-group $appname --kind MongoDB --enable-free-tier
```

## Creating the Service Bus Namespace 
```bash 
export appname=playeconomy-01
az servicebus namespace create --name $appname --resource-group $appname --sku Standard
```
## Creating Container Registry 
```bash
export appname=playeconomy-01
export acr="playeconomy01acr"
az acr create --name $acr --resource-group $appname --sku Basic
```

## Creating AKS cluster
```bash 
az aks create -n $appname -g $appname --node-vm-size Standard_B2s --node-count 2 --attach-acr $acr \
   --enable-oidc-issuer --enable-workload-identity --generate-ssh-keys

az aks get-credentials --name $appname --resource-group $appname
```
## Creating Azure Key Vault 
```bash 
az keyvault create -n $appname -g $appname 
```

## Installing Emissary Ingress
```bash
# Add the Helm repository for Ambassador (Datawire)
helm repo add datawire https://app.getambassador.io

# Update your local Helm chart repository cache
helm repo update

kubectl create namespace emissary && \
kubectl apply -f https://app.getambassador.io/yaml/emissary/3.9.1/emissary-crds.yaml

kubectl wait --timeout=90s --for=condition=available deployment emissary-apiext -n emissary-system

export namespace=emissary
helm install emissary-ingress datawire/emissary-ingress --set service.annotations."service\.beta\.kubernetes\.io/azure-dns-label-name"=$appname --namespace $namespace && \
kubectl -n $namespace wait --for condition=available --timeout=90s deploy -lapp.kubernetes.io/instance=emissary-ingress

kubectl get svc -w  --namespace emissary emissary-ingress
```

## Configuring Emissary-ingress routing
```bash 
kubectl apply -f ./emissary-ingress/listener.yaml -n $namespace
kubectl apply -f ./emissary-ingress/mappings.yaml -n $namespace
```