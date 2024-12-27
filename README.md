# API Management solution for receiving Graph API notifications via Subscription APIs

## Installation

1. Create an API Management instance and associate an instance of Azure Application Insights to it.
2. Create a new API by importing the EventHun-API.openapi.yaml as Swagger/OpenAPI definition.
3. Create a new product and add the API to the product. Disable the "requires subscription" option.
4. In the API view, select `EventHub API`, go to the "Settings" tab and make sure the "Subscription required" option is disabled. Enable application insights and possibly log payload too.
5. Go to "Named values" section and create the following named values:
    - **ClientState**: a arbitrary string that must match the one originally sent in the Graph API subscription request.
    - **GraphAPI-TenantId**: the tenant id of the Graph API.
    - **GraphAPI-ClientId**: the client id of the App Registration, in Graph API tenant, that has the permissions to create/update subscriptions.
    - **GraphAPI-Certificate**: BASE64-encoded PFX certificate (with no password) that is used to authenticate the App Registration. This will be typically stored in a Key Vault.
    - **EventHub-Namespace**: the namespace of the Event Hub where the notifications will be sent to. _Make sure the Managed Identity of the API Management instance has the necessary permissions on the Event Hub_.
    - **EventHub-Name**: the name of the Event Hub where the notifications will be sent to.
6. Go to API view, select `EventHub API`, select "All operations". In the "Inbound processing" section, click on the policy XML editor and replace all the text with the content of the `alloperations-policy.xml`. Click on "Save" button to save the policy.
7. Click on "POST lifecycle". In the "Inbound processing" section, click on the policy XML editor and replace all the text with the content of the `lifecycle-policy.xml`. Click on "Save" button to save the policy.
8. Click on "POST subscription". In the "Inbound processing" section, click on the policy XML editor and replace all the text with the content of the `subscription-policy.xml`. Click on "Save" button to save the policy.

by [Gabriel Mercuri](gmercuri@microsoft.com) and [Luca Giuliani](giulianil@microsoft.com)
