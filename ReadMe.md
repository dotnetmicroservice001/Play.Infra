# Play.Infra 
Play Economy Infrastructure components
```bash
export owner=[GIHHUB-PERSONAL-OR-ORG-NAME]"dotnetmicroservice001"
export gh_pat=[YOUR_PERSONAL_ACCESS_TOKEN]"ghp_"
export RG=[RESOURCE-GROUP-NAME-HERE]rg-playground
export COSMOS=[DB-NAME-HERE]"cosmos-pe"
export SB=[SERVICE-BUS-HERE]"sb-pf-pos"
export ACR=[CONTAINER-REGISTRY-HERE]"acrpfpos"
export AKS=[AKS-NAME-HERE]"aks-playground"
export KV=[KEY-VAULT-NAME]"kv-pf-pos"
```



## Add GitHub Package source 

```bash 
dotnet nuget add source \
  --username "$owner" \
  --password "$gh_pat" \
  --store-password-in-clear-text \
  --name github \
  "https://nuget.pkg.github.com/$owner/index.json"
``` 

## Creating Azure Resource Group 
```bash
az group create --name $RG --location westus
```

## Creating CosmosDB Account 
```bash
az cosmosdb create --name $COSMOS --resource-group $RG --kind MongoDB --enable-free-tier
```

## Creating the Service Bus Namespace 
```bash 
az servicebus namespace create --name $SB --resource-group $RG --sku Standard
```
## Creating Container Registry 
```bash
az acr create --name $ACR --resource-group $RG --sku Basic
```

## Creating AKS cluster
```bash 
az aks create -n $AKS -g $RG --node-vm-size Standard_B2s --node-count 2 --attach-acr $ACR \
   --enable-oidc-issuer --enable-workload-identity --generate-ssh-keys

az aks get-credentials --name $AKS --resource-group $RG
```
## Creating Azure Key Vault 
```bash 
az keyvault create -n $KV -g $RG 
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

export appname=myapp
export namespace=emissary
helm install emissary-ingress datawire/emissary-ingress --set service.annotations."service\.beta\.kubernetes\.io/azure-dns-label-name"=$appname --namespace $namespace && \
kubectl -n $namespace wait --for condition=available --timeout=90s deploy -lapp.kubernetes.io/instance=emissary-ingress

kubectl rollout status deployment/emissary-ingress -n $namespace -w
kubectl get svc -w  --namespace emissary emissary-ingress
```

## Configuring Emissary-ingress routing
```bash 
kubectl apply -f ./emissary-ingress/listener.yaml -n $namespace
kubectl apply -f ./emissary-ingress/mappings.yaml -n $namespace
```

## Installing Cert Manager 
```bash
helm repo add jetstack https://charts.jetstack.io --force-update

helm install \
  cert-manager jetstack/cert-manager \
  --namespace $namespace \
  --version v1.18.2 \
  --set crds.enabled=true
```

## Creating Cluster Issure
```bash
kubectl apply -f ./cert-manager/cluster-issuer.yaml -n "$namespace"
kubectl apply -f ./cert-manager/acme-challenge.yaml -n "$namespace"
```
## Creating TLS certificate 
```bash 
kubectl apply -f ./emissary-ingress/tls-certificate.yaml -n "$namespace"
```
## Enabling TLS and HTTPS 
```bash
kubectl apply -f ./emissary-ingress/host.yaml -n "$namespace"
```

## Packaging and publishing microservice helm chart 
```bash 
helm package ./helm/microservice

helmUser="00000000-0000-0000-0000-000000000000"
helmPassword=$(az acr login --name $ACR --expose-token --output tsv --query accessToken)
helm registry login $ACR.azurecr.io --username $helmUser --password $helmPassword 
version=1.0.2
helmchart=playflow-microservice
helm push $helmchart-$version.tgz oci://$ACR.azurecr.io/helm

```

## Create GitHub service principle 
```bash
export appId=$(az ad sp create-for-rbac -n "Github" --query appId --output tsv)
```

## Deploying Seq to AKS 
```bash
helm repo add datalust https://helm.datalust.co
helm repo update 
export pass="[admin password here]"
helm install seq datalust/seq -n observability --create-namespace --set firstRunAdminPassword=$pass

```

### Deploy Jaeger to AKS
```bash
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update 
helm upgrade jaeger jaegertracing/jaeger --values jaeger/values.yaml -n observability --install

```

## Deploy Prometheus and Grafana
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update 
helm upgrade prometheus prometheus-community/kube-prometheus-stack --values ./prometheus/values.yaml -n observability --install
```