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