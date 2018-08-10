# Deploying the Reference Implementation



## Prerequisites

- Azure suscription
- [Azure CLI 2.0](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
- [Docker](https://docs.docker.com/)
- [Docker Compose](https://docs.docker.com/compose/install/)

Clone or download this repo locally.

```bash
git clone https://github.com/mspnp/microservices-reference-implementation.git
```

The deployment steps shown here use Bash shell commands. On Windows, you can use the [Windows Subsystem for Linux](https://docs.microsoft.com/windows/wsl/about) to run Bash.

## Create the Kubernetes cluster

Set environment variables.

```bash
export LOCATION=[YOUR_LOCATION_HERE]

export UNIQUE_APP_NAME_PREFIX=[YOUR_UNIQUE_APPLICATION_NAME_HERE]

export RESOURCE_GROUP="${UNIQUE_APP_NAME_PREFIX}-rg" && \
export CLUSTER_NAME="${UNIQUE_APP_NAME_PREFIX}-cluster"
```

Provision a Kubernetes cluster in ACS

```bash
# Log in to Azure
az login

# Create a resource group for ACS
az group create --name $RESOURCE_GROUP --location $LOCATION

# Create the ACS cluster
az aks create --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --node-count 8 --generate-ssh-keys --node-vm-size Standard_DS3_v2

# Install kubectl
sudo az aks install-cli

# Get the Kubernetes cluster credentials
az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME

# Create the BC namespaces
kubectl create namespace shipping && \
kubectl create namespace accounts && \
kubectl create namespace dronemgmt && \
kubectl create namespace 3rdparty 
```

Create an Azure Container Registry instance. 

> Note: Azure Container Registory is not required. If you prefer, you can store the Docker images for this solution in another container registry.

```bash
export ACR_NAME=[YOUR_CONTAINER_REGISTRY_NAME_HERE]

# Create the ACR instance
az acr create --name $ACR_NAME --resource-group $RESOURCE_GROUP --sku Basic

# Log in to ACR
az acr login --name $ACR_NAME

# Get the ACR login server name
export ACR_SERVER=$(az acr show -g $RESOURCE_GROUP -n $ACR_NAME --query "loginServer" -o tsv)
```

## Deploy the Delivery service

Provision Azure resources

```bash
export REDIS_NAME="${UNIQUE_APP_NAME_PREFIX}-delivery-service-redis" && \
export COSMOSDB_NAME="${UNIQUE_APP_NAME_PREFIX}-delivery-service-cosmosdb" && \
export DATABASE_NAME="${COSMOSDB_NAME}-db" && \
export COLLECTION_NAME="${DATABASE_NAME}-col"

# Create Azure Redis Cache
az redis create --location $LOCATION \
            --name $REDIS_NAME \
            --resource-group $RESOURCE_GROUP \
            --sku Premium \
            --vm-size P4

# Create Cosmos DB account with DocumentDB API
az cosmosdb create \
    --name $COSMOSDB_NAME \
    --kind GlobalDocumentDB \
    --resource-group $RESOURCE_GROUP \
    --max-interval 10 \
    --max-staleness-prefix 200 
```

Build the Delivery service

```bash
export DELIVERY_PATH=./microservices-reference-implementation/src/shipping/delivery
docker-compose -f $DELIVERY_PATH/docker-compose.ci.build.yml up
```

Build and publish the container image 

```bash
# Build the Docker image
docker build -t $ACR_SERVER/fabrikam.dronedelivery.deliveryservice:0.1.0 $DELIVERY_PATH/Fabrikam.DroneDelivery.DeliveryService/.

# Push the image to ACR
az acr login --name $ACR_NAME
docker push $ACR_SERVER/fabrikam.dronedelivery.deliveryservice:0.1.0

```

Create Kubernetes secrets

```bash
export REDIS_CONNECTION_STRING=[YOUR_REDIS_CONNECTION_STRING]

export COSMOSDB_KEY=$(az cosmosdb list-keys --name $COSMOSDB_NAME --resource-group $RESOURCE_GROUP --query primaryMasterKey) && \
export COSMOSDB_ENDPOINT=$(az cosmosdb show --name $COSMOSDB_NAME --resource-group $RESOURCE_GROUP --query documentEndpoint)

kubectl --namespace shipping create --save-config=true secret generic delivery-storageconf \
    --from-literal=CosmosDB_Key=${COSMOSDB_KEY[@]//\"/} \
    --from-literal=CosmosDB_Endpoint=${COSMOSDB_ENDPOINT[@]//\"/} \
    --from-literal=Redis_ConnectionString=${REDIS_CONNECTION_STRING} \
    --from-literal=EH_ConnectionString=
```

Deploy the Delivery service:

```bash
# Update the image tag in the deployment YAML
sed -i "s#image:#image: $ACR_SERVER/fabrikam.dronedelivery.deliveryservice:0.1.0#g" ./microservices-reference-implementation/k8s/delivery.yaml

## Update config values in the deployment YAML
sed -i "s/value: \"CosmosDB_DatabaseId\"/value: $DATABASE_NAME/g"      "./microservices-reference-implementation/k8s/delivery.yaml" && \
sed -i "s/value: \"CosmosDB_CollectionId\"/value: $COLLECTION_NAME/g"  "./microservices-reference-implementation/k8s/delivery.yaml" && \
sed -i "s/value: \"EH_EntityPath\"/value:/g"                           "./microservices-reference-implementation/k8s/delivery.yaml"

# Deploy the service
kubectl --namespace shipping apply -f ./microservices-reference-implementation/k8s/delivery.yaml
```

## Deploy the Package service

Provision Azure resources

```bash
export COSMOSDB_NAME="${UNIQUE_APP_NAME_PREFIX}-package-service-cosmosdb"
az cosmosdb create --name $COSMOSDB_NAME --kind MongoDB --resource-group $RESOURCE_GROUP
```

Build the Package service

```bash
export PACKAGE_PATH=microservices-reference-implementation/src/shipping/package

# Build the app
docker-compose -f $PACKAGE_PATH/build/docker-compose.ci.build.yml up

# Build the docker image
sudo docker build -f $PACKAGE_PATH/build/prod.dockerfile -t $ACR_SERVER/package-service:0.1.0 $PACKAGE_PATH

# Push the docker image to ACR
az acr login --name $ACR_NAME
docker push $ACR_SERVER/package-service:0.1.0
```

Deploy the Package service

```bash
# Update deployment YAML with image tage
sed -i "s#image:#image: $ACR_SERVER/package-service:0.1.0#g" ./microservices-reference-implementation/k8s/package.yml

# Create secret
export COSMOSDB_CONNECTION=$(az cosmosdb list-connection-strings --name $COSMOSDB_NAME --resource-group $RESOURCE_GROUP --query "connectionStrings[0].connectionString")
kubectl -n shipping create secret generic package-secrets --from-literal=mongodb-pwd=${COSMOSDB_CONNECTION[@]//\"/}

# Deploy service
kubectl --namespace shipping apply -f ./microservices-reference-implementation/k8s/package.yml
```

## Deploy the Ingestion service 
Provision Azure resources

```bash
export INGESTION_EH_NS=[INGESTION_EVENT_HUB_NAMESPACE_HERE]
export INGESTION_EH_NAME=[INGESTION_EVENT_HUB_NAME_HERE]
export INGESTION_EH_CONSUMERGROUP_NAME=[INGESTION_EVENT_HUB_CONSUMERGROUP_NAME_HERE]

# Create an Event Hubs namespace
az eventhubs namespace create --name $INGESTION_EH_NS \
                              --resource-group $RESOURCE_GROUP \ 
                              -l $LOCATION

# Create an event hub
az eventhubs eventhub create --name $INGESTION_EH_NAME \
                             --resource-group $RESOURCE_GROUP \
                             --namespace-name $INGESTION_EH_NS \
                             --partition-count 8

# Create consumer group
az eventhubs eventhub consumer-group create --eventhub-name $INGESTION_EH_NAME \
                                            --name $INGESTION_EH_CONSUMERGROUP_NAME \
                                            --namespace-name $INGESTION_EH_NS \
                                            --resource-group $RESOURCE_GROUP

# Create authorization rule
az eventhubs eventhub authorization-rule create --eventhub-name $INGESTION_EH_NAME \
                                                --name IngestionServiceAccessKey \
                                                --namespace-name $INGESTION_EH_NS \
                                                --resource-group $RESOURCE_GROUP \
                                                --rights Listen

# Get access key
export EH_ACCESS_KEY_VALUE=$(az eventhubs eventhub authorization-rule keys list --resource-group $RESOURCE_GROUP --namespace-name $INGESTION_EH_NS --name IngestionServiceAccessKey --eventhub-name $INGESTION_EH_NAME --query primaryKey)

# Strip quotes
export EH_ACCESS_KEY_VALUE=("${EH_ACCESS_KEY_VALUE[@]//\"/}")
```

Build the Ingestion service

```bash
export INGESTION_PATH=./microservices-reference-implementation/src/shipping/ingestion

# Build the app 
docker build -t openjdk_and_mvn-build:8-jdk -f $INGESTION_PATH/Dockerfilemaven $INGESTION_PATH && \
docker run -it --rm -v $( cd "${INGESTION_PATH}" && pwd )/:/sln openjdk_and_mvn-build:8-jdk

# Build the docker image
docker build -f $INGESTION_PATH/Dockerfile -t $ACR_SERVER/ingestion:0.1.0 $INGESTION_PATH

# Push the docker image to ACR
az acr login --name $ACR_NAME
docker push $ACR_SERVER/ingestion:0.1.0
```

Deploy the Ingestion service

```bash
# Update deployment YAML with image tage
sed -i "s#image:#image: $ACR_SERVER/ingestion:0.1.0#g" ./microservices-reference-implementation/k8s/ingestion.yaml

# Create secret
kubectl -n shipping create secret generic ingestion-secrets --from-literal=eventhub_namespace=${INGESTION_EH_NS} \
--from-literal=eventhub_name=${INGESTION_EH_NAME} \
--from-literal=eventhub_keyname=IngestionServiceAccessKey \
--from-literal=eventhub_keyvalue=${EH_ACCESS_KEY_VALUE}

# Deploy service
kubectl --namespace shipping apply -f ./microservices-reference-implementation/k8s/ingestion.yaml
```

## Deploy the Scheduler service 

Provision Azure resources
```bash
export SCHEDULER_STORAGE_ACCOUNT_NAME=[SCHEDULER_STORAGE_ACCOUNT_NAME_HERE]

az storage account create --resource-group $RESOURCE_GROUP --name $SCHEDULER_STORAGE_ACCOUNT_NAME --sku Standard_LRS
```

Build the Scheduler service

```bash
export SCHEDULER_PATH=./microservices-reference-implementation/src/shipping/scheduler

# Build the app 
docker build -t openjdk_and_mvn-build:8-jdk -f $SCHEDULER_PATH/Dockerfilemaven $SCHEDULER_PATH && \
docker run -it --rm -v $( cd "${SCHEDULER_PATH}" && pwd )/:/sln openjdk_and_mvn-build:8-jdk

# Build the docker image
docker build -f $SCHEDULER_PATH/Dockerfile -t $ACR_SERVER/scheduler:0.1.0 $SCHEDULER_PATH

# Push the docker image to ACR
az acr login --name $ACR_NAME
docker push $ACR_SERVER/scheduler:0.1.0
```

Deploy the Scheduler service

```bash
# Update deployment YAML with image tage
sed -i "s#image:#image: $ACR_SERVER/scheduler:0.1.0#g" ./microservices-reference-implementation/k8s/scheduler.yaml

# Get the following values from the Azure Portal
export EH_CONNECTION_STRING="[YOUR_EVENT_HUB_CONNECTION_STRING_HERE]"
export STORAGE_ACCOUNT_ACCESS_KEY=[YOUR_STORAGE_ACCOUNT_ACCESS_KEY_HERE]
export STORAGE_ACCOUNT_CONNECTION_STRING="[YOUR_STORAGE_ACCOUNT_CONNECTION_STRING_HERE]"

# Create secrets
kubectl -n shipping create secret generic scheduler-secrets --from-literal=eventhub_name=${INGESTION_EH_NAME} \
--from-literal=eventhub_sas_connection_string=${EH_CONNECTION_STRING} \
--from-literal=storageaccount_name=${SCHEDULER_STORAGE_ACCOUNT_NAME} \
--from-literal=storageaccount_key=${STORAGE_ACCOUNT_ACCESS_KEY} \
--from-literal=queueconstring=${STORAGE_ACCOUNT_CONNECTION_STRING}

# Deploy service
kubectl --namespace shipping apply -f ./microservices-reference-implementation/k8s/scheduler.yaml
```

## Deploy mock services

Build the mock services

```bash
export MOCKS_PATH=microservices-reference-implementation/src/shipping/delivery
docker-compose -f $MOCKS_PATH/docker-compose.ci.build.yml up
```

Build and publish the container image 

```bash
# Build the Docker image
docker build -t $ACR_SERVER/account:0.1.0 $MOCKS_PATH/MockAccountService/. && \
docker build -t $ACR_SERVER/dronescheduler:0.1.0 $MOCKS_PATH/MockDroneScheduler/. && \
docker build -t $ACR_SERVER/thirdparty:0.1.0 $MOCKS_PATH/MockThirdPartyService/. 

# Push the image to ACR
az acr login --name $ACR_NAME
docker push $ACR_SERVER/account:0.1.0 && \
docker push $ACR_SERVER/dronescheduler:0.1.0 && \
docker push $ACR_SERVER/thirdparty:0.1.0
```

Deploy the mock services:

```bash
# Update the image tag in the deployment YAML
sed -i "s#image:#image: $ACR_SERVER/account:0.1.0#g" ./microservices-reference-implementation/k8s/account.yaml && \
sed -i "s#image:#image: $ACR_SERVER/dronescheduler:0.1.0#g" ./microservices-reference-implementation/k8s/dronescheduler.yaml && \
sed -i "s#image:#image: $ACR_SERVER/thirdparty:0.1.0#g" ./microservices-reference-implementation/k8s/thirdparty.yaml 

# Deploy the service
kubectl --namespace accounts apply -f ./microservices-reference-implementation/k8s/account.yaml && \
kubectl --namespace dronemgmt apply -f ./microservices-reference-implementation/k8s/dronescheduler.yaml && \
kubectl --namespace 3rdparty apply -f ./microservices-reference-implementation/k8s/thirdparty.yaml
```

## Verify all services are running:

```bash
kubectl get all --all-namespaces -l co=fabrikam
```

Deploy Elasticsearch. For more information, see https://github.com/kubernetes/examples/tree/master/staging/elasticsearch

```bash
kubectl --namespace kube-system apply -f https://raw.githubusercontent.com/kubernetes/examples/master/staging/elasticsearch/service-account.yaml && \
kubectl --namespace kube-system apply -f https://raw.githubusercontent.com/kubernetes/examples/master/staging/elasticsearch/es-svc.yaml && \
kubectl --namespace kube-system apply -f https://raw.githubusercontent.com/kubernetes/examples/master/staging/elasticsearch/es-rc.yaml
```

Deploy Fluend. For more information, see https://docs.fluentd.org/v0.12/articles/kubernetes-fluentd

```bash
# The example elasticsearch yaml files deploy a service named "elasticsearch"
wget https://raw.githubusercontent.com/fluent/fluentd-kubernetes-daemonset/master/fluentd-daemonset-elasticsearch.yaml && \
sed -i "s/elasticsearch-logging/elasticsearch/" fluentd-daemonset-elasticsearch.yaml

# Commenting out X-Pack credentials for demo purposes. 
# Make sure to configure X-Pack in elasticsearch and provide credentials here for production workloads
sed -i "s/- name: FLUENT_ELASTICSEARCH_USER/#- name: FLUENT_ELASTICSEARCH_USER/" fluentd-daemonset-elasticsearch.yaml && \
sed -i 's/  value: "elastic"/#  value: "elastic"/' fluentd-daemonset-elasticsearch.yaml && \
sed -i "s/- name: FLUENT_ELASTICSEARCH_PASSWORD/#- name: FLUENT_ELASTICSEARCH_PASSWORD/" fluentd-daemonset-elasticsearch.yaml && \
sed -i 's/  value: "changeme"/#  value: "changeme"/' fluentd-daemonset-elasticsearch.yaml && \
kubectl --namespace kube-system apply -f fluentd-daemonset-elasticsearch.yaml
``` 

#### Deploy linkerd 

For more information, see [https://linkerd.io/getting-started/k8s/](https://linkerd.io/getting-started/k8s/)

> Note: 
> the service mesh configuration linked above is defaulting the namespace to "default" for service discovery.  
> Since Drone Delivery microservices are getting deployed into several custom namespaces, this config needs to be modified. This will consist of a small change in the dtab rules.

Deploy linkerd defaulting the namespace to shipping instead: 

```bash
wget https://raw.githubusercontent.com/linkerd/linkerd-examples/master/k8s-daemonset/k8s/servicemesh.yml && \
sed -i "s#/default#/shipping#g" servicemesh.yml && \
sed -i "149i \ \ \ \ \ \ \ \ /svc/account => /svc/account.accounts ;" servicemesh.yml && \ 
sed -i "149i \ \ \ \ \ \ \ \ /svc/dronescheduler => /svc/dronescheduler.dronemgmt ;" servicemesh.yml && \
sed -i "149i \ \ \ \ \ \ \ \ /svc/thirdparty => /svc/thirdparty.3rdparty ;" servicemesh.yml && \
sed -i "176i \ \ \ \ \ \ \ \ /svc/account => /svc/account.accounts ;" servicemesh.yml && \
sed -i "176i \ \ \ \ \ \ \ \ /svc/dronescheduler => /svc/dronescheduler.dronemgmt ;" servicemesh.yml && \
sed -i "176i \ \ \ \ \ \ \ \ /svc/thirdparty => /svc/thirdparty.3rdparty ;" servicemesh.yml && \
kubectl apply -f servicemesh.yml
``` 

Deploy Prometheus and Grafana. For more information, see https://github.com/linkerd/linkerd-viz#kubernetes-deploy

It is recommended to put an API Gateway in front of all APIs you want exposed to the public, 
however for convenience, we exposed the Ingestion service with a public IP address.

You can send delivery requests to the ingestion service using the swagger ui.

```bash
export INGESTION_SERVICE_EXTERNAL_IP_ADDRESS=$(kubectl get --namespace shipping svc ingestion -o jsonpath="{.status.loadBalancer.ingress[0].*}")
curl "http://${INGESTION_SERVICE_EXTERNAL_IP_ADDRESS}"/swagger-ui.html#/ingestion45controller/scheduleDeliveryAsyncUsingPOST
```
