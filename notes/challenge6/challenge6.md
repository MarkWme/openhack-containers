Challenge 6 requires a number of steps
* Deploy a new AKS cluster with support for Azure AD integration, API Server Authorised Ranges, Pod Security Policy and Network Policy.
* Install and configure Azure AD Pod Identity, accounting for pod security policy
* Install and configure Key Vault Flex Volume, accounting for pod security policy
* Update the application deployment manifests with service accounts and update FlexVol configuration to work with Pod Identity

### Deploy a new AKS cluster
To use API Server Authorised Ranges, a new cluster is needed, so you need to repeat the steps from challenge 3 to create an Azure AD enabled cluster, plus the additional steps to enable pod security policies, network policy and authorised IP address ranges.

* Use the ```secureaks.sh``` script to set up a new AKS cluster with all of the required components enabled. Make sure you update the ```--api-server-authorized-ip-ranges``` value to match the IP address you'll be running the ```kubectl``` commands from.
* Once the deployment is complete, you should be able to connect to the cluster and list pods. Then try from a device with a different IP address and it should fail. The error may be something like:
```
Unable to connect to the server: dial tcp 51.104.137.120:443: connectex: A connection attempt failed because the connected party did not properly respond after a period of time, or established connection failed because connected host has failed to respond.
```
### Deploy Key Vault using Managed Pod Identity
* Install AAD Pod Identity
```
kubectl apply -f https://raw.githubusercontent.com/Azure/aad-pod-identity/master/deploy/infra/deployment-rbac.yaml
```
* Allow Pod Identity to work with Pod Security Policies by deploying the YAML file available at https://github.com/Azure/aad-pod-identity/blob/master/examples/psp-podidentity.yaml (a copy of that file is also in this folder)
* Create a Managed Identity.
```
az identity create -g challenge6 -n secure-aks-id -o json
```
* Get the ID of the Service Principal your AKS cluster is using
```
az aks show -g challenge6 -n secureaks --query servicePrincipalProfile.clientId -o tsv
```
* Assign the ```Managed Identity Operator``` role to the AKS cluster's service principal so it can manage the identity you created earlier
```
az role assignment create --role "Managed Identity Operator" --assignee <aks cluster service principal id>  --scope <full id of the managed identity>
```
* Configure roles for Key Vault. Use the Key Vault we created in challenge 4
```
az role assignment create --role Reader --assignee 39d5a900-c68d-43c6-bf99-076917936f4d --scope subscriptions/4ec6b1b9-7c2b-4e51-9ccc-0d3f132a8fb4/resourceGroups/teamResources/providers/Microsoft.KeyVault/vaults/vault200313
az keyvault set-policy -n vault200313 --key-permissions get --spn 39d5a900-c68d-43c6-bf99-076917936f4d
az keyvault set-policy -n vault200313 --secret-permissions get --spn 39d5a900-c68d-43c6-bf99-076917936f4d
az keyvault set-policy -n vault200313 --certificate-permissions get --spn 39d5a900-c68d-43c6-bf99-076917936f4d
```
* Apply the ```aadpodidentity.yaml``` file
* Apply the ```aadpodidentitybinding.yaml``` file

* Deploy nginx ingress, including the additional line to enable support for Pod Security Policies
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/docs/examples/psp/psp.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/provider/cloud-generic.yaml
```

* Create two Kubernetes namespaces with labels. The labels are used by network policy to select namespaces
```
kubectl create ns api-dev
kubectl label namespace/api-dev environment=api-dev
kubectl create ns web-dev
kubectl label namespace/web-dev environment=web-dev
```
* Create a Kubernetes service account for the applications running in the web-dev and api-dev namespaces
* Create a ClusterRoleBinding to bind those service accounts to the ClusterRole that's linked to a psp that allows deployment
* Update the deployment YAML files for the tripinsights applications to use service accounts and managed identity.
  * Add ```aadpodidbinding: "key-vault-pod-identity"``` to the deployment's labels section
  * Add ```serviceAccountName: service-account-api``` to the pod's spec
  * Remove the ```secretRef``` section from ```flexVolume```
  * Change ```usepodidentity``` to ```true```
* Deploy the updated manifests to get the application up and running again. 

### Network Policy

* Connect to any pod and confirm you can access another pod. For example, connect to the tripviewer pod 

 az identity create -g challenge6 -n secure-aks-id -o json
{
  "clientId": "39d5a900-c68d-43c6-bf99-076917936f4d",
  "clientSecretUrl": "https://control-northeurope.identity.azure.net/subscriptions/4ec6b1b9-7c2b-4e51-9ccc-0d3f132a8fb4/resourcegroups/challenge6/providers/Microsoft.ManagedIdentity/userAssignedIdentities/secure-aks-id/credentials?tid=423cb5ae-47a2-444f-a224-f2722e3180f3&oid=bf1287ae-5d74-4dfc-a8b5-b5728857e9f7&aid=39d5a900-c68d-43c6-bf99-076917936f4d",
  "id": "/subscriptions/4ec6b1b9-7c2b-4e51-9ccc-0d3f132a8fb4/resourcegroups/challenge6/providers/Microsoft.ManagedIdentity/userAssignedIdentities/secure-aks-id",
  "location": "northeurope",
  "name": "secure-aks-id",
  "principalId": "bf1287ae-5d74-4dfc-a8b5-b5728857e9f7",
  "resourceGroup": "challenge6",
  "tags": {},
  "tenantId": "423cb5ae-47a2-444f-a224-f2722e3180f3",
  "type": "Microsoft.ManagedIdentity/userAssignedIdentities"
}

### Hints
Pod Security Policy locks down deployments to the cluster. If you see errors similar to the following:
```
api-dev     26m         Warning   FailedCreate        replicaset/poi-api-fb49b47cb                              Error creating: pods "poi-api-fb49b47cb-" is forbidden: unable to validate against any pod security policy: []
```
... it will be because you're missing either a service account specified in the deployment or the service account is not bound to a pod security policy.
