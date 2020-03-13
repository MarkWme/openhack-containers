* Create an AKS cluster
    ```
    az group create -n challenge2 -l northeurope
    az aks create -g challenge2 -n akscluster --kubernetes-version 1.17.0
    ```

* Attach Azure Container Registry to Azure Kubernetes Service
    ```
    az aks update -n akscluster -g challenge2 --attach-acr registryamd4608
    ```
* Deploy the applications using the Kubernetes manifests in this repo.
* Check the web application is working by accessing via a port forwarder
    ```
    kubectl port-forward service/tripviewer-service 8080:80
    ```
* Access the web site from a browser and confirm that the Profie and Trips links work and don't return errors

* Optional: Test the API components using port forwarding and curl
    ```
    kubectl port-forward service/poi-service 8080:8080
    kubectl port-forward service/trips-service 8080:80
    kubectl port-forward service/user-profile-java-service 8080:80
    kubectl port-forward service/user-profile-js-service 8080:80

    curl -i -X GET 'http://localhost:8080/api/poi/healthcheck' 
    HTTP/1.1 200 OK
    Date: Fri, 13 Mar 2020 11:02:20 GMT
    Content-Type: application/json; charset=utf-8
    Server: Kestrel
    Transfer-Encoding: chunked

    {
    "message": "POI Service Healthcheck",
    "status": "Healthy"
    }

    curl -i -X GET 'http://localhost:8080/api/trips/healthcheck' 
    HTTP/1.1 200 OK
    Content-Type: application/json; charset=UTF-8
    Date: Fri, 13 Mar 2020 11:02:48 GMT
    Content-Length: 58

    {"message":"Trip Service Healthcheck","status":"Healthy"}

    curl -i -X GET 'http://localhost:8080/api/user-java/healthcheck'  
    HTTP/1.1 200 
    Content-Type: application/json;charset=UTF-8
    Transfer-Encoding: chunked
    Date: Fri, 13 Mar 2020 11:04:35 GMT

    {"message":"healthcheck","status":"healthy"}

    curl -i -X GET 'http://localhost:8080/api/user/healthcheck'  
    HTTP/1.1 200 OK
    Content-Type: application/json; charset=utf-8
    Content-Length: 44
    ETag: W/"2c-Z/X/1/sKDZv7XBEmQHl7gI0+zjE"
    Date: Fri, 13 Mar 2020 11:05:06 GMT
    Connection: keep-alive

    {"message":"healthcheck","status":"healthy"}
    ```