# Windows Admin Center

Install and configure the Windows Admin Center for Azure Arc-enabled Server.

Cannot create the endpoint in an ARM template yet, except with Deployment Scripts.

## Prereqs

1. Azure Arc-enabled Server (Windows)
1. RBAC role assignment for Windows Admin Center Administrator Login group

## References

* <https://learn.microsoft.com/windows-server/manage/windows-admin-center/azure/manage-arc-hybrid-machines>
* <https://learn.microsoft.com/azure/azure-arc/servers/manage-vm-extensions>
* <https://learn.microsoft.com/azure/virtual-machines/extensions/features-linux>
* <https://learn.microsoft.com/azure/virtual-machines/extensions/features-windows>
* <https://learn.microsoft.com/rest/api/hybridcompute/machine-extensions/create-or-update>

## Manual extension installation

Example commands for an Azure Arc-enabled Server.

1. Variables

    ```bash
    subscription=$(az account show --query id --output tsv)
    resourceGroup="arc_pilot"
    machineName="win-01"
    location="westeurope"
    typeHandler="0.0"
    ```

1. RBAC role assignment

    We'll use a security group. Create the group.

    ```bash
    groupId=$(az ad group create --display-name "Windows Admin Center Administrators" --mail-nickname wacadmins --query id --output tsv)
    ```

    Get your objectId.

    ```bash
    myId=$(az ad signed-in-user show --query id --output tsv)
    ```

    Add yourself as a member.

    ```bash
    az ad group member add --group $groupId --member-id $myId
    ```

    Create the role assignment.

    ```bash
    az role assignment create \
      --assignee $groupId \
      --role "Windows Admin Center Administrator Login" \
      --scope $(az group show --name $resourceGroup --query id --output tsv)
    ```

1. Install the extension

    ```bash
    az connectedmachine extension create --name AdminCenter \
      --resource-group $resourceGroup --location $location \
      --machine-name $machineName \
      --type AdminCenter --publisher Microsoft.AdminCenter \
      --settings '{"port": "6516"}'
    ```

1. Add the Endpoint

    ```bash
    url="https://management.azure.com/subscriptions/${subscription}/resourceGroups/${resourceGroup}/providers/Microsoft.HybridCompute/machines/${machineName}/providers/Microsoft.HybridConnectivity/endpoints/default?api-version=2021-10-06-preview"
    ```

    ```bash
    az rest --method put \
      --url $url \
      --body "{'properties': {'type': 'default'}}"
    ```

1. Display the Endpoint

    ```bash
    az rest --url $url
    ```

## ARM Template

```bash
az deployment group create --resource-group $resourceGroup --name "${machineName}-AdminCenter" --template-file azuredeploy-extension_and_endpoint.json --parameters vmName=$machineName
```

> Note that this will fail with teh default ALZ policy assignment as the storage account created by deploymentScripts will be public.

## Troubleshooting

Restarting the himds service directly on the server usually brings it to life.

```powershell
Restart-Service -Name himds
```

Also check connectivity for egress and reverse proxy.

```powershell
Invoke-RestMethod -Method GET -Uri https://wac.azure.com
```

```powershell
azcmagent config list
```

> You can specify multiple ports for azcmagent, e.g.
>
> `azcmagent config set incomingconnections.ports "22,6516"`

## Policy installation

Management group assumed here, using id `alz`. Check `az policy definition create` for alternative scopes. The policy is extension only at the moment.

1. Move to this directory
1. Create the audit definition

    ```bash
    az policy definition create \
      --management-group "alz" \
      --name "audit_windows_admin_center_on_windows_arc_vms" \
      --display-name "[Preview]: Windows Admin Center extension should be installed on your Windows Arc machine" \
      --description "Windows Admin Center extension should be installed on your Windows Arc machine." \
      --metadata version="0.1.0-preview" category="Azure Arc" preview="true" \
      --mode "Indexed" \
      --rules policy-audit_windows_admin_center_on_windows_arc_vms.rules.json \
      --params policy-audit_windows_admin_center_on_windows_arc_vms.parameters.json
    ```

1. Create the configure definition

    ```bash
    az policy definition create \
      --management-group "alz" \
      --name "configure_windows_admin_center_on_windows_arc_vms" \
      --display-name "[Preview]: Configure Windows Arc machines to automatically install the Windows Admin Center extension" \
      --description "Configure Windows Admin Center extension for Windows Arc machines." \
      --metadata version="0.1.0-preview" category="Azure Arc" preview="true" \
      --mode "Indexed" \
      --rules policy-configure_windows_admin_center_on_windows_arc_vms.rules.json \
      --params policy-configure_windows_admin_center_on_windows_arc_vms.parameters.json
    ```

1. Create the initiative

    ```bash
    az policy set-definition create \
      --management-group "alz" \
      --name "windows_admin_center_on_windows_arc_vms" \
      --display-name "[Preview]: Deploy Windows Admin Center to Windows Azure Arc machines" \
      --description "Deploy the Windows Admin Center extension to Windows Azure Arc-enabled virtual machines." \
      --metadata version="0.1.0-preview" category="Azure Arc" preview="true" \
      --definitions initiative-windows_admin_center_on_windows_arc_vms.definitions.json \
      --params initiative-windows_admin_center_on_windows_arc_vms.parameters.json
    ```

    > Note that the resourceIds in the initiative are hardcoded for the alz management group.

1. Assign the intiative at the landing zones management group

    Use the parameter defaults.

    ```bash
    az policy assignment create --name wac_on_arc \
      --display-name "Deploy Windows Admin Center to Windows Azure Arc machines" \
      --description "Deploy the Windows Admin Center extension to Windows Azure Arc-enabled virtual machines" \
      --policy-set-definition /providers/Microsoft.Management/managementGroups/alz/providers/Microsoft.Authorization/policySetDefinitions/windows_admin_center_on_windows_arc_vms \
      --scope /providers/Microsoft.Management/managementGroups/alz-landingzones \
      --mi-system-assigned --location westeurope \
      --identity-scope /providers/Microsoft.Management/managementGroups/alz-landingzones \
      --role "Azure Connected Machine Resource Administrator"
    ```
