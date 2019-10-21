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
-   Select the VNET created before
-   Create API (go to "APIS" in the left navbar then "Blank API". )
-   Create operation (select the API, then "Add operation") 
-   Add policy for the operation (see further down)


## VM for DNS-updater
Please do this more securely! :-)
````
az vm create \
  --resource-group "p2spoc" \
  --name "dnsupdater" \
  --image "UbuntuLTS" \
  --admin-username "p2suser" \
  --admin-password "p2suser@123" \
  --location local
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

On the client, I put this "API code". For it to run, you must for do

````
pip install flask-restful
````

````
from flask import Flask                                                                         
from flask_restful import Api, Resource, reqparse, request                                      
                                                                                                
app = Flask(__name__)                                                                           
api = Api(app)                                                                                  
                                                                                                
users = [                                                                                       
    {                                                                                           
        "name": "Alexander",                                                                    
        "age": 100,                                                                             
        "occupation": "Beard Man"                                                               
    },                                                                                          
    {                                                                                           
        "name": "Elvin",                                                                        
        "age": 32,                                                                              
        "occupation": "Doctor"                                                                  
    },                                                                                          
    {                                                                                           
        "name": "Peter",                                                                        
        "age": 45,                                                                              
        "occupation": "Azure dude"                                                              
    }                                                                                           
]                                                                                               
                                                                                                
tools = [                                                                                       
    {                                                                                           
        "name": "Screwdriver",                                                                  
        "size": 1                                                                               
    },                                                                                          
    {                                                                                           
        "name": "Jack",                                                                         
        "capacity": "5000 kg"                                                                   
    },                                                                                          
    {                                                                                           
        "name": "Wrench",                                                                       
        "weight": "10 kg"                                                                       
    }                                                                                           
]                                                                                               
                                                                                                
                                                                                                
class Tool(Resource):                                                                           
    def get(self, name):                                                                        
        url = request.url                                                                       
        print ("Request to: " +url)                                                             
        print ("Name: " +name)                                                                  
                                                                                                
        for tool in tools:                                                                      
          if(name == tool["name"]):                                                             
            return tool, 200                                                                    
        return "Tool not found", 404                                                            
                                                                                                
class User(Resource):                                                                           
    def get(self, name):                                                                        
        url = request.url                                                                       
        print ("Request to: " +url)                                                             
        print ("Name: " +name)                                                                  
                                                                                                
        for user in users:                                                                      
          if(name == user["name"]):                                                             
            return user, 200                                                                    
        return "User not found", 404                                                            
                                                                                                
    def post(self, name):                                                                       
        parser = reqparse.RequestParser()                                                       
        parser.add_argument("age")                                                              
        parser.add_argument("occupation")                                                       
        args = parser.parse_args()                                                              
                                                                                                
        for user in users:                                                                      
            if(name == user["name"]):                                                           
                return "User with name {} already exists".format(name), 400                     
                                                                                                
        user = {                                                                                
            "name": name,                                                                       
            "age": args["age"],                                                                 
            "occupation": args["occupation"]                                                    
        }                                                                                       
        users.append(user)                                                                      
        return user, 201                                                                        
                                                                                                
    def put(self, name):                                                                        
        parser = reqparse.RequestParser()                                                       
        parser.add_argument("age")                                                              
        parser.add_argument("occupation")                                                       
        args = parser.parse_args()                                                              
                                                                                                
        for user in users:                                                                      
            if(name == user["name"]):                                                           
                user["age"] = args["age"]                                                       
                user["occupation"] = args["occupation"]                                         
                return user, 200                                                                
                                                                                                
        user = {                                                                                
            "name": name,                                                                       
            "age": args["age"],                                                                 
            "occupation": args["occupation"]                                                    
        }                                                                                       
        users.append(user)                                                                      
        return user, 201                                                                        
                                                                                                
    def delete(self, name):                                                                     
        global users                                                                            
        users = [user for user in users if user["name"] != name]                                
        return "{} is deleted.".format(name), 200                                               
                                                                                                
api.add_resource(User, "/user/<string:name>")                                                   
api.add_resource(Tool, "/tool/<string:name>")                                                   
                                                                                                
app.run(port= 8080, host= '0.0.0.0')                                                            
````
