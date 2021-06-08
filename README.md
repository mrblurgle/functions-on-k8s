# Azure Functions on K8s

## Introduction

This project provides:

1. a simple Azure Function with a queue listener
2. a set of instructions for deploying the function to a k8s cluster, optionally using AKS
3. a test program for adding messages to the queue

## Creating a function with a Docker file

This step explains how to create a Docker enabled function.

You will need to have the Azure Function CLI installed to use the `func` tools. You can find the [Azure Functions Core Tools installation instructions here](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=windows%2Ccsharp%2Cbash#install-the-azure-functions-core-tools).

Instructions for creating a [k8s enabled function can be found here](https://docs.microsoft.com/en-us/azure/azure-functions/functions-kubernetes-keda).

The commands in this documentation are similar to what you will find in the [Azure Functions command line quick-start](https://docs.microsoft.com/en-us/azure/azure-functions/create-first-function-cli-csharp?tabs=azure-cli%2Cbrowser&source=docs), but are specific for creating a Dockerised function app.

Once you have installed the tools, you can initialize a new Azure Functions project - with Docker support - by running the following command:

`func init ReviewsWorkerFunctionApp --dotnet --docker`

The `./ReviewsWorkerFunctionApp` sub-directory will be generated via the previous command, containing the scaffolding for a function app. Use:

`cd ./ReviewsWorkerFunctionApp`

...to change to this directory.

For the purposes of this demo, we will create a simple Azure Storage Queue listener. We can add a new storage queue listener function with the following command from within:

`func new`

A list of template options will be provided - choose:
 `QueueTrigger`

...and for the name we will use:
`ReviewQueueListener`

The purposes of this function will be to listen to messages from a reviews queue, which we will populate in a moment. Note that triggers like the queue-listener may require special scaling metrics - such as the [KEDA queue storage scaler](https://keda.sh/docs/1.4/scalers/azure-storage-queue/) - while HTTP triggered functions can leverage the default Horizontal Pod Autoscaling metrics for a web API running under Kubernetes.

Because we are building a queue storage trigger, we will need an Azure Storage Queue. Note that you can use other types of queues and event streams with KEDA including:

- (Azure Service Bus)[https://keda.sh/docs/1.4/scalers/azure-service-bus/]
- (RabbitMq)[https://keda.sh/docs/1.4/scalers/rabbitmq-queue/]
- (Apache Kafka)[https://keda.sh/docs/1.4/scalers/apache-kafka/]
- (AWS Kinesis)[https://keda.sh/docs/1.4/scalers/aws-kinesis/]

## Creating the Azure Storage Account

Next I created an [Azure Storage Queue using the Azure CLI](https://docs.microsoft.com/en-us/azure/storage/common/storage-account-create?tabs=azure-cli). I have borrowed some of the [material from the Azure team blog](https://medium.com/microsoftazure/lifting-function-to-kubernetes-with-keda-e24de86fca2e). You can also [create an Azure Queue via the portal](https://docs.microsoft.com/en-us/azure/storage/queues/storage-quickstart-queues-portal). Make sure you login:

`az.cmd login`

I'm assuming you [already have an Azure subscription](https://docs.microsoft.com/en-us/cli/azure/manage-azure-subscriptions-azure-cli). Find the target subscription with:

`az.cmd account list -o table`

You will be presented with a table of subscriptions. Make sure `IsDefault` is true for the target subscription, otherwise use the following command to set the subscription:

`az.cmd account set --subscription <subscription-id-here>`

Replace `<subscription-id-here>` with the relevant subscription Id from the output of the list command.

We'll create a new resource-group for our resources. Set a variable for the resource-group name, as we'll need to use this a few times:

`group=reviews-rg`

You'll also need to choose a location to host your resources. You can list the locations with:

`az.cmd account list-locations -o table`

I set a location variable with:

`location=australiaeast`

Create the resource-group with:

`az.cmd group create --name $group --location $location`

The final command will create the Azure Storage Account. We will first set a few variables

1) the name of the parent storage account for the queue
`storageAccountName=reviewsonk8storage`

2) the `kind` of the account - typically you will want `StorageV2` at the time of writing:
`storageKind=StorageV2`

3) the (storage-account SKU)[storageKind]. As I'm only using this for a demo, I've selected:
`storageSku=Standard_LRS`

4) access-tier is `hot` as we will be querying the data frequently:
`storageAccessTier=hot`

5) I've set `https-only` to true, so that only requests sent over HTTPS are supported:
`storageHttpsOnly=true`

The command to create the storage-account will then be:

`az.cmd storage account create --resource-group $group --name $storageAccountName --location $location --kind $storageKind --sku $storageSku --access-tier $storageAccessTier --https-only $storageHttpsOnly`

Finally we need to create the storage queue inside the new storage account. [The queue creation parameters are explained here](https://docs.microsoft.com/en-us/cli/azure/storage/queue?view=azure-cli-latest#az_storage_queue_create), but we will go through them.

We will call our storage queue `review-submitted`:
`queueName=review-submitted`

We also need a storage-account key to use with the queue. We will generate this using the following command and store the output in a variable for later use:

`storageAccountKey=$(az.cmd storage account keys list --resource-group $group --account-name $storageAccountName --query "[0].value" | tr -d '"')`

We will also grab a connection string which we will need to use with our function app in a later section:

`storageAccountConnectionString="DefaultEndpointsProtocol=https;AccountName=$storageAccountName;AccountKey=$storageAccountKey;EndpointSuffix=core.windows.net"`

Use `echo $storageAccountConnectionString` to output the connection string and make note of it for later. 

We can then create the new storage queue using the generated account key and connection string:

`az.cmd storage queue create --name $queueName --account-key $storageAccountKey --account-name $storageAccountName --connection-string $storageAccountConnectionString`

## Connect the function to the queue

We will now make some small changes to the function we generated earlier to connect it to the queue.

In the function code: `./ReviewsWorkerFunctionApp/ReviewQueueListener.cs`

...add the queue-name and a new alias for the connection string, i.e.

`public static void Run([QueueTrigger("myqueue-items", Connection = "")]string myQueueItem, ILogger log)`

becomes:

`public static void Run([QueueTrigger("review-submitted", Connection = "ReviewQueueConnectionString")]string myQueueItem, ILogger log)`

Note that this means we need the running environment of the function app to provide a connection-string called `ReviewQueueConnectionString`. We'll address this in a minute.

## Creating a private Docker repository with ACR

Unless you want to push your Docker image to a public image repository, you'll need to create a private Docker store. For this purpose, I will use ACR (Azure Container Registry). Below are the commands to provision the registry.

First, declare a name for the registry:

`containerRegistryName=reviewsContainerRegistryDemo`

The following command will create the container registry. As a side-note, you'll likely want to create a container registry for your entire project, possibly in its own resource group. I have created it in the same group for the convenience of being able to delete the whole group after the demo. I'll use the (`Basic` SKU)[https://docs.microsoft.com/en-us/azure/container-registry/container-registry-skus] as we're just using this for a demo:

`az.cmd acr create --resource-group $group --name $containerRegistryName --sku Basic`

The next command assigns a system managed identity to the container-registry:

`az.cmd acr identity assign --identities [system] --name $containerRegistryName`

## Generate the Docker image

## Push the Docker image to ACR

## (Optional) Create an AKS cluster

## Deploy the service to Kubernetes

## Add some messages to the queue