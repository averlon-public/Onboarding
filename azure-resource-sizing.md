
Following script provides high level information about the types of resources and their count in Azure subscription (it assumes you have [az cli](https://learn.microsoft.com/en-us/cli/azure/) and [jq](https://jqlang.github.io/jq/) installed):

```
#!/bin/bash

# Check if Azure CLI is installed
if ! command -v az &> /dev/null
then
    echo "Azure CLI not found. Please install and configure Azure CLI."
    exit 1
fi

# Authenticate to Azure
echo "Authenticating to Azure..."
az account show &> /dev/null

if [ $? -ne 0 ]; then
    echo "Azure authentication failed. Please login using 'az login'."
    exit 1
fi

# Prompt the user to enter locations
read -p "Enter the locations (comma-separated, e.g., eastus,westus,centralus,eastus2,westus2): " regions_input
IFS=',' read -r -a locations <<< "$regions_input"

# Function to count resources for a given subscription and region
count_resources() {
    local subscription_id=$1
    local location=$2

    echo "Gathering resource counts for Subscription: $subscription_id, Region: $location"

    # Set the subscription and location for Azure CLI commands
    az account set --subscription $subscription_id

    # Count resources
    rbac_roles_count=$(az role definition list --query '[].name' --output json | jq length)
    users_count=$(az ad user list --query 'length(@)' --output tsv)
    managed_identities_count=$(az identity list --query "[?location=='$location']" --output json | jq length)
    role_assignments_count=$(az role assignment list --query 'length(@)' --output tsv)
    vm_count=$(az vm list --query "[?location=='$location']" --output json | jq length)
    aks_clusters_count=$(az aks list --query "[?location=='$location']" --output json | jq length)
    container_instances_count=$(az container list --query "[?location=='$location']" --output json | jq length)
    function_apps_count=$(az functionapp list --query "[?location=='$location']" --output json | jq length)
    sql_servers_count=$(az sql server list --query "[?location=='$location']" --output json | jq length)
    storage_accounts_count=$(az storage account list --query "[?location=='$location']" --output json | jq length)
    security_groups_count=$(az network nsg list --query "[?location=='$location']" --output json | jq length)

    # Get total number of images across all ACR repositories
    acr_images_count=0
    for registry in $(az acr list --query "[?location=='$location'].name" --output tsv); do
        count=$(az acr repository list --name "$registry" --output json | jq length)
        acr_images_count=$((acr_images_count + count))
    done

    # Fetch all VM Scale Sets in the location
    vmss_list=$(az vmss list --query "[?location=='$location']" --output json)

    # Count VMs in VM Scale Sets
    vmss_count=0
    for vmss in $(echo "$vmss_list" | jq -r '.[].name'); do
        resource_group=$(echo "$vmss_list" | jq -r ".[] | select(.name==\"$vmss\").resourceGroup")
        count=$(az vmss list-instances --name "$vmss" --resource-group "$resource_group" --query 'length(@)' --output tsv)
        vmss_count=$((vmss_count + count))
    done

    total_vm_count=$((vm_count + vmss_count))

    echo "Results for Subscription: $subscription_id, Region: $location"
    echo "-------------------------------------------"
    echo "Azure RBAC Roles: $rbac_roles_count"
    echo "Users: $users_count"
    echo "Managed Identities: $managed_identities_count"
    echo "Role Assignments: $role_assignments_count"
    echo "Virtual Machines (VMs): $total_vm_count (including VMs in Scale Sets)"
    echo "AKS Clusters: $aks_clusters_count"
    echo "Container Instances: $container_instances_count"
    echo "Function Apps: $function_apps_count"
    echo "SQL Servers: $sql_servers_count"
    echo "Storage Accounts: $storage_accounts_count"
    echo "Network Security Groups: $security_groups_count"
    echo "Total ACR Images: $acr_images_count"
    echo "-------------------------------------------"
    echo ""
}

# Get the current subscription ID
current_subscription_id=$(az account show --query 'id' --output tsv)

# Prompt the user to choose between current subscription or all subscriptions
read -p "Do you want to use only the current subscription (yes/no)? " use_current_subscription

if [[ "$use_current_subscription" == "yes" ]]; then
    echo "Using only the current subscription: $current_subscription_id"
    for location in "${locations[@]}"; do
        count_resources $current_subscription_id $location
    done
else
    echo "Enumerating resources for all subscriptions."
    subscriptions=$(az account list --query '[].id' --output tsv)
    for subscription_id in $subscriptions; do
        for location in "${locations[@]}"; do
            count_resources $subscription_id $location
        done
    done
fi
```

Steps to execute the script: 
1. Save the script.(Ex: azure_resource_count.sh)
2. Make the script executable
```
chmod +x azure_resource_count.sh
```
3. Run the script
```
./azure_resource_count.sh 
```


