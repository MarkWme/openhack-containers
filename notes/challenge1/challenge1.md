# Challenge 1

* Fork and clone the open-hack containers repository. You can fork the repository into your own GitHub repository by going to https://github.com/Azure-Samples/openhack-containers and then click the "Fork" icon at the top right of the page. Once that's done, download a local copy of the repository to work from
    ```
    git clone https://github.com/<your repo name>/openhack-containers.git
    ```
* Build the docker containers for each of the applications
* Create the Docker network
    ```
    docker network create --driver bridge sql_local
    ```
* Pull the SQL Server image
    ```
    docker pull mcr.microsoft.com/mssql/server:2017-latest
    ```
* Start the SQL Server container
    ```
    docker run -e "ACCEPT_EULA=Y" -e "SA_PASSWORD=TsaosSQL1a6" -p 1433:1433 --name sql1 --network=sql_local -d mcr.microsoft.com/mssql/server:2017-latest
    ```
* Connect to the SQL Server container
    ```
    docker exec -it sql1 "bash"
    ```
* Start the sqlcmd tool
    ```
    /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P "<Password>"
    ```
* Create the database:
    ```
    1> CREATE DATABASE mydrivingDB
    2> GO
    ```
* Load the data
    ```
    docker run --network sql_local -e SQLFQDN=<Container ID> -e SQLUSER=sa -e SQLPASS=<Password> -e SQLDB=mydrivingDB openhack/data-load:v1
    ```
    Note: The SQLFQDN will be the ID of the container

* Start the POI application

    ```
    docker run -d -p 8080:80 --network=sql_local --name poi -e "SQL_USER=sa" -e "SQL_PASSWORD=<Password>" -e "SQL_SERVER=<Container ID>" -e "ASPNETCORE_ENVIRONMENT=Local" poi:1.0
    ```
    Note that the ```Local``` in ```ASPNETCORE_ENVIRONMENT``` is case sensitive!

* Test the POI endpoint

    ```
    curl -i -X GET 'http://localhost:8080/api/poi'
    ```

    You should see the following result

    ```
    HTTP/1.1 200 OK
    Date: Thu, 12 Mar 2020 12:48:58 GMT
    Content-Type: application/json; charset=utf-8
    Server: Kestrel
    Transfer-Encoding: chunked

    [
    {
        "tripId": "ea2f7ae0-3cef-49cb-b7d1-ce972113120c",
        "latitude": 47.690263235265249,
        "longitude": -122.09165568111251,
        "poiType": 2,
        "timestamp": "1900-01-01T00:00:00",
        "deleted": false,
        "id": "264ffaa3-1fe8-4fb0-a4fb-63bdbc9999ae"
    },
    {
        "tripId": "ea2f7ae0-3cef-49cb-b7d1-ce972113120c",
        "latitude": 47.667148026473811,
        "longitude": -122.11271963003664,
        "poiType": 1,
        "timestamp": "1900-01-01T00:00:00",
        "deleted": false,
        "id": "a7c2d2c6-e803-43a8-bf3d-340b7987e73a"
    }
    ]
    ```
* Tag and push the images to Azure Container Registry under ```tripinsights```

