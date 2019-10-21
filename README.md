# APIM-poc

## Create Resource Group
e.g. 
````
az group create -n p2spoc -l westeurope
````

## Create VNET/VNET Gateway/Certificates
https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-howto-point-to-site-resource-manager-portal



## Create private DNS zone

````
az network private-dns zone create -g p2spoc -n private.p2spoc

az network private-dns link vnet create -g p2spoc -n MyDNSLink -z private.p2spoc -v <VNET Name> -e true
````

## Create APIM 
-	Developer sku. 
-	Internal (under network settings)
- Select the VNET created before
- Create API (e.g. "test-api")
- Create operation (e.g. "user") 
- Add policy for the operation (see further down)


## VM for DNS-updater
````
az vm create -n dnsupdater -g p2spoc --image UbuntuLTS
````
Then ssh to the VM and do 
````
git clone https://github.com/pelithne/dnsupdater.git
cd dnsupdater
az login
````
az login is needed because the dns updater script relies on being logged in, since it is using az-commands.

Before starting the dns updater, you might have to edit the script a little. The two variables you might have to change are:
````
dnszone = "private.p2spoc"
resourcegroup = "p2spoc"
````

Now start the updater
````
python dnsupdater.py
````


When the updater is up and running, you can e.g. curl it, with a requiest of this format:
````
curl -X POST 172.17.0.5:8080/host/m1
````

where "m1" is the hostname that you want to use for this connection. The resulting A Record will look something like this:

````
m1.p2spoc.private 10.10.10.10
````

Now, this dns name can be added to the API manager, with a policy similar to this:
<!--
    IMPORTANT:
    - Policy elements can appear only within the <inbound>, <outbound>, <backend> section elements.
    - To apply a policy to the incoming request (before it is forwarded to the backend service), place a corresponding policy element within the <inbound> section element.
    - To apply a policy to the outgoing response (before it is sent back to the caller), place a corresponding policy element within the <outbound> section element.
    - To add a policy, place the cursor at the desired insertion point and select a policy from the sidebar.
    - To remove a policy, delete the corresponding policy statement from the policy document.
    - Position the <base> element within a section element to inherit all policies from the corresponding section element in the enclosing scope.
    - Remove the <base> element to prevent inheriting policies from the corresponding section element in the enclosing scope.
    - Policies are applied in the order of their appearance, from the top down.
    - Comments within policy elements are not supported and may disappear. Place your comments between policy elements or at a higher level scope.
-->
<policies>
    <inbound>
        <choose>
            <when condition="@(context.Request.Url.Query.GetValueOrDefault("machine") == "m1")">
                <set-backend-service base-url="http://m1.p2spoc.private:8080" />
            </when>
            <when condition="@(context.Request.Url.Query.GetValueOrDefault("machine") == "m2")">
                <set-backend-service base-url="http://m2.p2spoc.private:8080" />
            </when>
        </choose>
        <base />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>





## Create VPN "client"
https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-howto-point-to-site-resource-manager-portal
