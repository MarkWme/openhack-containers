* Setup Azure AD
    ```
    aksname="akscluster"

    # Create the Azure AD application
    serverApplicationId=$(az ad app create \
        --display-name "${aksname}Server" \
        --identifier-uris "https://${aksname}Server" \
        --query appId -o tsv)

    # Update the application group memebership claims
    az ad app update --id $serverApplicationId --set groupMembershipClaims=All

    # Create a service principal for the Azure AD application
    az ad sp create --id $serverApplicationId

    # Get the service principal secret
    serverApplicationSecret=$(az ad sp credential reset \
        --name $serverApplicationId \
        --credential-description "AKSPassword" \
        --query password -o tsv)

    az ad app permission add \
        --id $serverApplicationId \
        --api 00000003-0000-0000-c000-000000000000 \
        --api-permissions e1fe6dd8-ba31-4d61-89e7-88639da4683d=Scope 06da0dbc-49e2-44d2-8312-53f166ab848a=Scope 7ab1d382-f21e-4acd-a863-ba3e13f7da61=Role

    az ad app permission grant --id $serverApplicationId --api 00000003-0000-0000-c000-000000000000
    az ad app permission admin-consent --id  $serverApplicationId

    clientApplicationId=$(az ad app create \
        --display-name "${aksname}Client" \
        --native-app \
        --reply-urls "https://${aksname}Client" \
        --query appId -o tsv)

    az ad sp create --id $clientApplicationId

    oAuthPermissionId=$(az ad app show --id $serverApplicationId --query "oauth2Permissions[0].id" -o tsv)

    az ad app permission add --id $clientApplicationId --api $serverApplicationId --api-permissions ${oAuthPermissionId}=Scope
    az ad app permission grant --id $clientApplicationId --api $serverApplicationId
    ```

* Get the ID of the existing network called "vnet"
    ```
    vnetId=$(az network vnet subnet list \
        --resource-group teamResources \
        --vnet-name vnet \
        --query "[0].id" --output tsv)
    ```
* Deploy an AKS cluster with Azure AD integration and Azure CNI enabled
    ```
    az group create -l northeurope -n challenge3 

    tenantId=$(az account show --query tenantId -o tsv)

    az aks create \
        --resource-group challenge3 \
        --name $aksname \
        --kubernetes-version 1.17.0 \
        --enable-cluster-autoscaler \
        --min-count 1 \
        --max-count 10 \
        --node-count 3 \
        --aad-server-app-id $serverApplicationId \
        --aad-server-app-secret $serverApplicationSecret \
        --aad-client-app-id $clientApplicationId \
        --aad-tenant-id $tenantId \
        --network-plugin azure \
        --vnet-subnet-id $vnetId \
        --docker-bridge-address 172.17.0.1/16 \
        --dns-service-ip 10.200.0.10 \
        --service-cidr 10.200.0.0/24
    ```
* Get kubeconfig setup as cluster administrator
    ```
    az aks get-credentials -g challenge3 -n akscluster --admin
    ```
* Create a ClusterRoleBinding to make each "hacker" account a cluster admin

* Get kubeconfig setup as a regular user

* Run a kubectl command and confirm that you are prompted to authenticate with Azure AD. Use one of the hacker accounts to login.

* Test that you are connected to the correct VNet. Start a container
    ```
    kubectl run -i -t busybox --image=busybox:latest -- /bin/sh
    ```
    Then when the prompt appears
    ```
    / # ping internal-vm

    PING internal-vm (10.2.0.4): 56 data bytes
    64 bytes from 10.2.0.4: seq=0 ttl=64 time=1.515 ms
    64 bytes from 10.2.0.4: seq=1 ttl=64 time=0.950 ms
    64 bytes from 10.2.0.4: seq=2 ttl=64 time=0.794 ms
    --- internal-vm ping statistics ---
    3 packets transmitted, 3 packets received, 0% packet loss
    round-trip min/avg/max = 0.794/1.086/1.515 ms
    ```

