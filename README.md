# Auto-Search and Delete All VNET Integration from the App Services that are Associated with a Specific Subnet
By design, Azure App Services are not deployed into a Virtual network. If your requirement is to set up a Site-Site VPN to an On-Premise network using Azure Virtual Network gateway (VPN Gateway), VNet Integration (azure App Services) is the way to go to provide better continuity for your workloads in hybrid cloud setup with Azure.

VNET Integration gives your App Services access to resources in your virtual network but does not grant private access to your App Services from the virtual network. A common scenario where you would use VNET Integration is enabling access from your App Services to a database or azure resources running in your Azure virtual network.

If you have configured VNET Integration with multiple sites with a single Subnet, and if you want to delete that Subnet then you can't delete unless you find and remove all the VNET associations with that Subnet. Currently, there is no way for you to figure out how many associations your Subnet has with the app services. In this scenario, you would need to go through all the app services manually within the serverfarm and remove the association.

The following PowerShell script will automate this task for you. It will list down all the App Services name that is associated with a given Subnet, and you can configure to delete all the association too at once:

```
$VNETName = "{VNET Name}"
$SubnetName = "{Subnet Name}"
$ResourceGroupName = "{VNET Resource Group Name}"
$DeleteAllAssociation = 'false' #value should be either 'true' or 'false'

$result = Get-AzureRmResource -ResourceGroupName $ResourceGroupName -ResourceType Microsoft.Network/virtualNetworks/subnets -ResourceName "$VNETName/$SubnetName" -ApiVersion 2018-07-01

$resourceURI = $result.Properties.serviceAssociationLinks | select -ExpandProperty properties

$serverFarm = Get-AzureRmResource -ResourceId $resourceURI.link | select Name, ResourceGroupName

Write-Host "The Subnet" $SubnetName "is associated with the serverfarm" $serverFarm.Name

$getSites = Get-AzureRmResource -ResourceGroupName $serverFarm.ResourceGroupName -ResourceType Microsoft.Web/serverfarms/sites -ResourceName $serverFarm.Name -ApiVersion 2018-02-01

Write-Host "Sites associated with this Subnet:"

foreach($site in $getSites)
{
    $getSite = Get-AzureRmWebApp -ResourceGroupName $site.ResourceGroupName -Name $site.Name

    if($getSite.SiteConfig.VnetName.ToLower() -like "*"+$SubnetName.ToLower()+"*")
        {   
            Write-Host  $site.Name

            if($DeleteAllAssociation -eq 'true')
                {
                    Write-Host "Deleting the Subnet association with the App Service"$getSite.Name
                    $id = $getSite.Id+"/networkconfig/virtualNetwork"
                    Remove-AzureRmResource -ResourceId $id -Force
                }
        }
} 

```
Hope you find this doc useful!!!
