# Application gateway tests

> Based on [this lesson](https://docs.microsoft.com/en-us/learn/modules/load-balance-web-traffic-with-application-gateway)

> Did this to test Azure Application Gateway Possibilities, I routed my personal domain andrevst.dev to the gateway IP and tested how to use path to route to internal and external servers.

## Creating base resources:

1. Resource Group

```shell
RG=appgw-test
az group create --name $RG --location brazilsouth
```

2. Network
```
VNET=andrevstAppVnet
WSSUBNET=webServerSubnet
az network vnet create --resource-group $RG --name $VNET --address-prefix 10.0.0.0/16 --subnet-name webServerSubnet --subnet-prefix 10.0.1.0/24
```
3. Create App VM

```shell
# Clone vm script
git clone https://github.com/MicrosoftDocs/mslearn-load-balance-web-traffic-with-application-gateway module-files
# Create VM 1
az vm create --resource-group $RG --name webServer1 --image UbuntuLTS --admin-username azureuser --generate-ssh-keys --vnet-name $VNET --subnet $WSSUBNET --public-ip-address "" --nsg "" --custom-data module-files/scripts/vmconfig.sh --no-wait
#Create VM 2
az vm create --resource-group $RG --name webServer2 --image UbuntuLTS --admin-username azureuser --generate-ssh-keys --vnet-name $VNET --subnet $WSSUBNET --public-ip-address "" --nsg "" --custom-data module-files/scripts/vmconfig.sh
# Validate
az vm list --resource-group $RG --show-details --output table
```

4. Create Web App (PaaS)

```shell
# Create App Plan
ASPLAN=testPlanAndrevstdev
az appservice plan create --resource-group $RG --name ASPLAN --sku S1
# Create App Service
APPSERVICE="licenserenewal$RANDOM"
az webapp create --resource-group $RG --name $APPSERVICE --plan $ASPLAN --deployment-source-url https://github.com/MicrosoftDocs/mslearn-load-balance-web-traffic-with-application-gateway --deployment-source-branch appService --runtime "DOTNETCORE|2.1"

