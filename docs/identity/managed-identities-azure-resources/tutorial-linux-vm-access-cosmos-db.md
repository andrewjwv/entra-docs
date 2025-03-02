---
title: Tutorial`:`Use a managed identity to access Azure Cosmos DB - Linux
description: A tutorial that walks you through the process of using a Linux VM system-assigned managed identity to access Azure Cosmos DB.

author: barclayn
manager: amycolannino

ms.service: entra-id
ms.subservice: managed-identities
ms.topic: tutorial
ms.tgt_pltfrm: na
ms.date: 12/10/2020
ms.author: barclayn
ROBOTS: NOINDEX
---
# Tutorial: Use a Linux VM system-assigned managed identity to access Azure Cosmos DB 

[!INCLUDE [preview-notice](~/includes/entra-msi-preview-notice.md)]

This tutorial shows you how to use a system-assigned managed identity for a Linux virtual machine (VM) to access Azure Cosmos DB. You learn how to:

> [!div class="checklist"]
> * Create an Azure Cosmos DB account
> * Create a collection in the Azure Cosmos DB account
> * Grant the system-assigned managed identity access to an Azure Cosmos DB instance
> * Retrieve the `principalID` of the of the Linux VM's system-assigned managed identity
> * Get an access token and use it to call Azure Resource Manager
> * Get access keys from Azure Resource Manager to make Azure Cosmos DB calls

## Prerequisites

- If you're not familiar with the managed identities for Azure resources feature, see this [overview](overview.md). 
- If you don't have an Azure account, [sign up for a free account](https://azure.microsoft.com/free/) before you continue.
- To perform the required resource creation and role management, your account needs "Owner" permissions at the appropriate scope (your subscription or resource group). If you need assistance with role assignment, see [Assign Azure roles to manage access to your Azure subscription resources](/azure/role-based-access-control/role-assignments-portal).
- To run the example scripts, you have two options:
    - Use the [Azure Cloud Shell](/azure/cloud-shell/overview), which you can open using the **Try It** button on the top right corner of code blocks.
    - Run scripts locally by installing the latest version of the [Azure CLI](/cli/azure/install-azure-cli), then sign in to Azure using [az login](/cli/azure/reference-index#az-login). Use an account associated with the Azure subscription in which you'd like to create resources.

## Create an Azure Cosmos DB account

If you don't already have one, create an Azure Cosmos DB account. You can skip this step and use an existing Azure Cosmos DB account. 

1. Click the **+ Create a resource** button found on the upper left-hand corner of the Azure portal.
2. Click **Databases**, then **Azure Cosmos DB**, and a new "New account" panel  displays.
3. Enter an **ID** for the Azure Cosmos DB account, which you use later.
4. **API** should be set to "SQL." The approach described in this tutorial can be used with the other available API types, but the steps in this tutorial are for the SQL API.
5. Ensure the **Subscription** and **Resource Group** match the ones you specified when you created your VM in the previous step.  Select a **Location** where Azure Cosmos DB is available.
6. Click **Create**.

### Create a collection in the Azure Cosmos DB account

Next, add a data collection in the Azure Cosmos DB account that you can query in later steps.

1. Navigate to your newly created Azure Cosmos DB account.
2. On the **Overview** tab click the **+/Add Collection** button, and an "Add Collection" panel slides out.
3. Give the collection a database ID, collection ID, select a storage capacity, enter a partition key, enter a throughput value, then click **OK**.  For this tutorial, it is sufficient to use "Test" as the database ID and collection ID, select a fixed storage capacity and lowest throughput (400 RU/s).  

## Grant access

To gain access to the Azure Cosmos DB account access keys from the Resource Manager in the following section, you need to retrieve the `principalID` of the Linux VM's system-assigned managed identity.  Be sure to replace the `<SUBSCRIPTION ID>`, `<RESOURCE GROUP>` (resource group in which your VM resides), and `<VM NAME>` parameter values with your own values.

```azurecli-interactive
az resource show --id /subscriptions/<SUBSCRIPTION ID>/resourceGroups/<RESOURCE GROUP>/providers/Microsoft.Compute/virtualMachines/<VM NAMe> --api-version 2017-12-01
```

The response includes the details of the system-assigned managed identity (note the principalID as it is used in the next section):

```output  
{
    "id": "/subscriptions/<SUBSCRIPTION ID>/<RESOURCE GROUP>/providers/Microsoft.Compute/virtualMachines/<VM NAMe>",
  "identity": {
    "principalId": "6891c322-314a-4e85-b129-52cf2daf47bd",
    "tenantId": "733a8f0e-ec41-4e69-8ad8-971fc4b533f8",
    "type": "SystemAssigned"
 }
```

### Grant your Linux VM's system-assigned identity access to the Azure Cosmos DB account access keys

Azure Cosmos DB does not natively support Microsoft Entra authentication. However, you can use a managed identity to retrieve an Azure Cosmos DB access key from the Resource Manager, then use the key to access Azure Cosmos DB. In this step, you grant your system-assigned managed identity access to the keys to the Azure Cosmos DB account.

To grant the system-assigned managed identity access to the Azure Cosmos DB account in Azure Resource Manager using the Azure CLI, update the values for `<SUBSCRIPTION ID>`, `<RESOURCE GROUP>`, and `<COSMOS DB ACCOUNT NAME>` for your environment. Replace `<MI PRINCIPALID>` with the `principalId` property returned by the `az resource show` command in Retrieve the principalID of the Linux VM's MI.  Azure Cosmos DB supports two levels of granularity when using access keys:  read/write access to the account, and read-only access to the account.  Assign the `DocumentDB Account Contributor` role if you want to get read/write keys for the account, or assign the `Cosmos DB Account Reader Role` role if you want to get read-only keys for the account:

```azurecli-interactive
az role assignment create --assignee <MI PRINCIPALID> --role '<ROLE NAME>' --scope "/subscriptions/<SUBSCRIPTION ID>/resourceGroups/<RESOURCE GROUP>/providers/Microsoft.DocumentDB/databaseAccounts/<COSMODS DB ACCOUNT NAME>"
```

The response includes the details for the role assignment created:

```output
{
  "id": "/subscriptions/<SUBSCRIPTION ID>/resourceGroups/<RESOURCE GROUP>/providers/Microsoft.DocumentDB/databaseAccounts/<COSMOS DB ACCOUNT>/providers/Microsoft.Authorization/roleAssignments/5b44e628-394e-4e7b-bbc3-d6cd4f28f15b",
  "name": "5b44e628-394e-4e7b-bbc3-d6cd4f28f15b",
  "properties": {
    "principalId": "c0833082-6cc3-4a26-a1b1-c4b5f90a981f",
    "roleDefinitionId": "/subscriptions/<SUBSCRIPTION ID>/providers/Microsoft.Authorization/roleDefinitions/fbdf93bf-df7d-467e-a4d2-9458aa1360c8",
    "scope": "/subscriptions/<SUBSCRIPTION ID>/resourceGroups/<RESOURCE GROUP>/providers/Microsoft.DocumentDB/databaseAccounts/<COSMOS DB ACCOUNT>"
  },
  "resourceGroup": "<RESOURCE GROUP>",
  "type": "Microsoft.Authorization/roleAssignments"
}
```

## Access data

For the remainder of the tutorial, work from the virtual machine.

To complete these steps, you need an SSH client. If you are using Windows, you can use the SSH client in the [Windows Subsystem for Linux](/windows/wsl/install). If you need assistance configuring your SSH client's keys, see [How to Use SSH keys with Windows on Azure](/azure/virtual-machines/linux/ssh-from-windows), or [How to create and use an SSH public and private key pair for Linux VMs in Azure](/azure/virtual-machines/linux/mac-create-ssh-keys).

1. In the Azure portal, navigate to **Virtual Machines**, go to your Linux virtual machine, then from the **Overview** page click **Connect** at the top. Copy the string to connect to your VM. 
2. Connect to your VM using your SSH client.  
3. Next, you are prompted to enter in your **Password** you added when creating the **Linux VM**. You should then be successfully signed in.  
4. Use CURL to get an access token for Azure Resource Manager: 
     
    ```bash
    curl 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https%3A%2F%2Fmanagement.azure.com%2F' -H Metadata:true   
    ```
 
    > [!NOTE]
    > In the previous request, the value of the "resource" parameter must be an exact match for what is expected by Microsoft Entra ID. When using the Azure Resource Manager resource ID, you must include the trailing slash on the URI.
    > In the following response, the access_token element as been shortened for brevity.
    
    ```json
    {
      "access_token":"eyJ0eXAiOi...",
      "expires_in":"3599",
      "expires_on":"1518503375",
      "not_before":"1518499475",
      "resource":"https://management.azure.com/",
      "token_type":"Bearer",
      "client_id":"1ef89848-e14b-465f-8780-bf541d325cd5"
    }
    ```

### Get access keys from Azure Resource Manager to make Azure Cosmos DB calls

Now use CURL to call Resource Manager using the access token retrieved in the previous section to retrieve the Azure Cosmos DB account access key. Once we have the access key, we can query Azure Cosmos DB. Be sure to replace the `<SUBSCRIPTION ID>`, `<RESOURCE GROUP>`, and `<COSMOS DB ACCOUNT NAME>` parameter values with your own values. Replace the `<ACCESS TOKEN>` value with the access token you retrieved earlier.  If you want to retrieve read/write keys, use key operation type `listKeys`.  If you want to retrieve read-only keys, use the key operation type `readonlykeys`:

```bash 
curl 'https://management.azure.com/subscriptions/<SUBSCRIPTION ID>/resourceGroups/<RESOURCE GROUP>/providers/Microsoft.DocumentDB/databaseAccounts/<COSMOS DB ACCOUNT NAME>/<KEY OPERATION TYPE>?api-version=2016-03-31' -X POST -d "" -H "Authorization: Bearer <ACCESS TOKEN>" 
```

> [!NOTE]
> The text in the prior URL is case-sensitive, so use the case that matches the case used in the name of your resource group. Additionally, it’s important to know that this is a POST request not a GET request and ensure you pass a value to capture a length limit with -d that can be NULL.  

The CURL response gives you the list of Keys.  For example, if you get the read-only keys:  

```bash 
{"primaryReadonlyMasterKey":"bWpDxS...dzQ==",
"secondaryReadonlyMasterKey":"38v5ns...7bA=="}
```

Now that you have the access key for the Azure Cosmos DB account, you can pass it to an Azure Cosmos DB SDK and make calls to access the account.

## Next steps

In this tutorial, you learned how to use a system-assigned managed identity on a Linux virtual machine to access Azure Cosmos DB.  To learn more about Azure Cosmos DB, see:

> [!div class="nextstepaction"]
>[Azure Cosmos DB overview](/azure/cosmos-db/introduction)
