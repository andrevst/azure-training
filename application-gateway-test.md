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

5. Create Application Gateway

5.1 Setup Gateway Network and IP

```shell
# Gateway Subnet
AGSUBNET=vehicleSubnet
az network vnet subnet create --resource-group $RG --vnet-name $VNET --name $AGSUBNET --address-prefixes 10.0.2.0/24
 az network vnet subnet create --resource-group $RG --vnet-name $VNET --name $AGSUBNET --address-prefixes 10.0.2.0/24
# Gateway Public IP
az network public-ip create --resource-group $RG --name appgwPublicIp --sku Standard --dns-name andrevstdev${RANDOM}
```

Now on Google, I setup my DNS to point to this IP, a A record for @ and WWW

5.2 Create the Application Gateway with WAF

```shell
az network application-gateway create --resource-group $RG --name andrevstdevAppGateway --sku WAF_v2 --capacity 2 --vnet-name $VNET --subnet $AGSUBNET --public-ip-address appGatewayPublicIp --http-settings-protocol Http --http-settings-port 8080 --private-ip-address 10.0.2.4 --frontend-port 8080
```

6. Add servers to backend pools

```shell
# get vm1 ip
az vm list-ip-addresses --resource-group $RG --name webserver1 --query [0].virtualMachine.network.privateIpAddresses[0] --output tsv
# get vm2 ip
az vm list-ip-addresses --resource-group $RG --name webserver1 --query [0].virtualMachine.network.privateIpAddresses[0] --output tsv
# Create VM Pool
az network application-gateway address-pool create --gateway-name andrevstdevAppGateway --resource-group $RG --name vmPool --servers 10.0.1.4 10.0.1.5
#Create Webapp Pool
az network application-gateway address-pool create --gateway-name andrevstdevAppGateway --resource-group $RG --name appServicePool --servers $APPSERVICE.azurewebsites.net
```
7. Create ports, Listeners and Probes

```shell
#create port 80
az network application-gateway frontend-port create --resource-group $RG --gateway-name andrevstdevAppGateway --name port80 --port 80
# Listener to handle requests on port 80
az network application-gateway http-listener create --resource-group $RG --name andrevstdevListener --frontend-port port80 --frontend-ip appGatewayFrontendIP --gateway-name andrevstdevAppGateway
```
> Create a Health Probe to monitor avaliability

```shell
az network application-gateway probe create --resource-group $RG --gateway-name andrevstdevAppGateway --name customProbe --path / --interval 15 --threshold 3 --timeout 10 --protocol Http --host-name-from-http-settings true
```
