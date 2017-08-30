---
services: azure-resource-manager, azure-storeage
platforms: dotnet
author: guanghu
---

# Manage Azure Stack resource and storage with .Net

This sample explains how to manage your resources and storage services in Azure Stack using the Azure .NET SDK. 

**On this page**
- [Run this sample](#run)
- [What is program.cs doing?](#example)
    - [login for Azure Stack](#login)
    - [set up the ResourceManagementClient object](#resourcemngtclient)
    - [set up the StorageManagementClient object](#storagemngtclient)
    - [create a storage account](#create-sa)
    - [create a blob container](#blob)

<a id="run"></a>
## Run this sample
### Prerequisits
1. This sample requires to be run either from the [Azure Stack Development Kit(ASDK)](https://docs.microsoft.com/en-us/azure/azure-stack/azure-stack-connect-azure-stack#connect-with-remote-desktop) or from an external client if you are [connected through VPN](https://docs.microsoft.com/en-us/azure/azure-stack/azure-stack-connect-azure-stack#connect-with-vpn).
1. If you don't have it, install the [.NET Core SDK](https://www.microsoft.com/net/core).
1. Recommand to [Install and configure CLI for use with Azure Stack](https://docs.microsoft.com/en-us/azure/azure-stack/azure-stack-connect-cli)
### Steps
1. Clone the repository.
    ```
    git clone https://github.com/guanghuthegreat/azurestack-storage-resources-sample-dotnet.git
    ```
1. Install the dependencies.
    ```
    dotnet restore
    ```
1. Create an Azure service principal either through
    [Azure CLI](https://azure.microsoft.com/documentation/articles/resource-group-authenticate-service-principal-cli/),
    [PowerShell](https://azure.microsoft.com/documentation/articles/resource-group-authenticate-service-principal/)
    or [the portal](https://azure.microsoft.com/documentation/articles/resource-group-create-service-principal-portal/).
   In Azure Stack give Contributor permissions to the subscription where the resources are stored. 
   An quick way is to run the following Azure CLI cmdlet on Azure Stack and record the **appId** and **password** as client id and client secret key. 
   ```
   az ad sp create-for-rbac -n {choose a name for sp}
   ```
1. Obtain the value for your account subscription ID and tenant ID. You can find these info by running CLI cmdlet `az account show` on the target Azure Stack and find out these value in the fields **id** and **tenantId** from the cmdlet outputs 
1. Obtain the URI for AAD login, AAD resource ID and endpoints for management, storage. You can easily find out these info by running CLI cmdlet `az cloud show` on the target Azure Stack and find out these value in the fields of **activeDirectory**, **activeDirectoryResourceId**, **management** and **storageEndpoint** from this cmdlet outputs. 
1. Export these environment variables and fill in the values you created in previous steps.  
    ```
    Set AZS_ACTTIVEDIRECTORY={the AAD login URI}
    Set AZS_ACTTIVEDIRECTORYRESOURCEID={the AAD resource ID}
    Set AZS_MANAGEMENTENDPOINT={the management endpoint URI}
    Set AZS_STORAGENDPOINT={the storage endpoint URI}
    Set AZS_SUBID={your subscription id}
    Set AZS_TENANTID={your tenant id}
    Set AZS_CLIENTID={your client id}
    Set AZS_SECRETKEY={your client secret key}
    Set AZS_LOCATION={the location (region) of your Azure Stack deployment, like 'local' in a ASDK deployments}
    ```
1. Run the sample.
    ```
    dotnet run
    ```
<a id="example"></a>
## What is program.cs doing? 
The sample walks you through several resrouce group and storage services operations. It starts by setting up a ResourceManagementClient and StorageManagementClient objects using your subscription and credentials. 
<a id="login"></a>
#### Login for Azure Stack:
In order to access to the specific Azure Stack you want to operate, you need to customize a **ActiveDirectoryServiceSettings** with the value of **AzS_ActiveDirectory** and **AzS_ActiveDirectoryResourceID** we prepared in previous steps pass it for login. Otherwise, your operation will just go to the public Azure as a default behavior. 
```
ActiveDirectoryServiceSettings s = new ActiveDirectoryServiceSettings();
s.AuthenticationEndpoint = new Uri(AzS_ActiveDirectory);
s.TokenAudience = new Uri(AzS_ActiveDirectoryResourceID);
s.ValidateAuthority = true;
var serviceCreds = await ApplicationTokenProvider.LoginSilentAsync(AzS_TenantID, AzS_ClientID, AzS_SecretKey, s);
```
<a id="resourcemngtclient"></a>
#### set up the ResourceManagementClient object
The mangement endpoint for resource management client object in Azure Stack has to be replaced by the value of **AzS_ManagementEndPoint** we prepared in previous steps. This is for the specific Azure Stack we need to manage. 
```
var resourceClient = new ResourceManagementClient(creds)
{
    BaseUri = new Uri(AzS_ManagementEndPoint),
    SubscriptionId = AzS_SubscriptionID
};
```
<a id="storagemngtclient"></a>
#### set up the StorageManagementClient object 
The mangement endpoint for storage management client object in Azure Stack has to be replaced by the value of **AzS_ManagementEndPoint** we prepared in previous steps. This is for the specific Azure Stack we need to manage. 
```            
var storageClient = new StorageManagementClient(creds)
{
    BaseUri = new Uri(AzS_ManagementEndPoint),
    SubscriptionId = AzS_SubscriptionID
};
```
<a id="create-sa"></a>
### create a storage account 
In Azure Stack, there are some differences compared to public Azure, for example, Azure Stack storage supports LRS and gerneral-purpose storage account type, so we need to identify these value correctly when creating a storage account. For more information in [Azure Stack Storage: Differences and considerations](https://docs.microsoft.com/en-us/azure/azure-stack/azure-stack-acs-differences)
```
// set default parameters for storage account in Azure Stack 
public static Microsoft.Azure.Management.Storage.Models.Sku DefaultSku = new Microsoft.Azure.Management.Storage.Models.Sku(SkuName.StandardLRS); // Azure Stack only supports LRS
public static Kind DefaultStorageKind = Kind.Storage; // Azure Stack only supports general-purpose stroage account type 
public static Dictionary<string, string> DefaultTags = new Dictionary<string, string>
{
    {"key1","value1"},
    {"key2","value2"}
};
```
```
// create storage accounts 
StorageAccountCreateParameters parameters = new StorageAccountCreateParameters
{
    Location = AzS_Location, 
    Kind = DefaultStorageKind, 
    Tags = DefaultTags, 
    Sku = DefaultSku 
};
var storageAccount = storageMgmtClient.StorageAccounts.Create(rgname, acctName, parameters);
```
<a id="blob"></a>
### create a blob container 
When creating a blob container in Azure Stack Storage service, only one thing needs to be specified, customized the storage endpoint uri during the CloudStorageAccount initializaiton. How to prepare the **Az_StorageEndPoint** you can find details in previous steps. 

```
StorageCredentials cre = new StorageCredentials(accountName, key);
CloudStorageAccount storageAccount = new CloudStorageAccount(cre, AzS_StorageEndPoint, true); // specify the value of storage endpoint for Azure Stack
CloudBlobClient blob = storageAccount.CreateCloudBlobClient();
CloudBlobContainer blobContainer = blob.GetContainerReference(blobcontainerName);
blobContainer.CreateIfNotExists();
```
