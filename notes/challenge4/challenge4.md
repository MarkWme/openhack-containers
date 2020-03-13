* Create a Key Vault instance to store secrets
```
az keyvault create -g teamResources -n vault200313
```
* Add the required SQL Server secrets to Key Vault
```
az keyvault secret set --vault-name vault200313 --name sql-server --value <sql server name>.database.windows.net
az keyvault secret set --vault-name vault200313 --name sql-user --value <sql user name>
az keyvault secret set --vault-name vault200313 --name sql-password --value <sql password>
```
* Configure Key Vault FlexVol for Kubernetes
```
kubectl create -f https://raw.githubusercontent.com/Azure/kubernetes-keyvault-flexvol/master/deployment/kv-flexvol-installer.yaml
```
* Create a service principal
```
az ad sp create-for-rbac --name keyvaultflexvolsp 
```
* Create a Kubernetes secret to store the service principal credentials. Remember to create the secrets in the same namespace as the applications (api-dev)
```
kubectl create secret generic kvcreds --from-literal clientid=<client id> --from-literal clientsecret=<client secret> --type=azure/kv
```
* Assign key vault permissions to your service principal
```
az keyvault set-policy -n <key vault name> --key-permissions get --spn <client id>
az keyvault set-policy -n <key vault name> --secret-permissions get --spn <client id>
az keyvault set-policy -n <key vault name> --certificate-permissions get --spn <client id>
```
* Create two Kubernetes namespaces
```
kubectl create ns api-dev
kubectl create ns web-dev
```
* Deploy the applications and services into the api-dev and web-dev namespaces
* Set up the web-dev and api-dev users to have access to the correct namespaces ```dev-roles.yaml```
* Test to make sure the web-dev and api-dev users have the correct access levels 
* Deploy nginx ingress
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/provider/cloud-generic.yaml
```
* Deploy Ingress configuration YAML files
* Start the simulator and confirm it is generating data
* Use curl to check endpoints
```
❯ curl http://51.104.136.125/api/poi/healthcheck
{
  "message": "POI Service Healthcheck",
  "status": "Healthy"
}

❯ curl http://51.104.136.125/api/trips/healthcheck
{"message":"Trip Service Healthcheck","status":"Healthy"}

❯ curl http://51.104.136.125/api/user/healthcheck
{"message":"healthcheck","status":"healthy"}

❯ curl http://51.104.136.125/api/user-java/healthcheck
{"message":"healthcheck","status":"healthy"}
```