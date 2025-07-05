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