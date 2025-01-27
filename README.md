# API Management solution for receiving Graph API notifications via Subscription APIs

## Installation

1. Create an App Registration in the Graph API tenant with application permissions on the resource subject of subscription e.g. for receiving notifications on messages, the App Registration must have `Mail.Read` permission on `Mail` resource (API Permissions -> Graph API -> Application -> Mail.Read and then "Grant admin consent").

2. Create a self-signed certificate and upload the public part of it to the App Registration. The certificate will be used to authenticate the App Registration when creating subscriptions. This can be achieved through the following PowerShell statements:

    ```powershell
    $c = New-SelfSignedCertificate -Subject "my-certificate-name" -KeyExportPolicy Exportable -CertStoreLocation Cert:\CurrentUser\My -NotAfter "2028-01-01"
    $b = $c.Export([System.Security.Cryptography.X509Certificates.X509ContentType]::Pfx)
    [IO.File]::WriteAllBytes("C:\temp\my-certificate-name.pfx", $b)
    [Convert]::ToBase64String($b)  # Copy the output of this command and store it as a secret in the Key Vault accessed by the API Management instance
    $c.ExportCertificatePem() | Out-File -Encoding ASCII "c:\temp\my-certificate-name.cer" # Upload this file to the App Registration - Certificate & secrets -> Certificates -> Upload certificate
    ```

3. Create an Application Insights instance in the same region as the API Management instance.

4. Create an Event Hub namespace (if not already done) **AND** an Event Hub in it. The namespace should be in the same region as the API Management instance.

5. Create an API Management instance and associate it to the Azure Application Insights created above.

6. Create a new API by importing the EventHun-API.openapi.yaml as Swagger/OpenAPI definition.

7. Create a new product and add the API to the product. Disable the "requires subscription" option.

8. In the API view, select `EventHub API`, go to the "Settings" tab and make sure the "Subscription required" option is disabled. Enable application insights and possibly log payload too.

9. Go to "Named values" section and create the following named values:
    - **ClientState**: a arbitrary string that must match the one originally sent in the Graph API subscription request.
    - **GraphAPI-TenantId**: the tenant id of the Graph API.
    - **GraphAPI-ClientId**: the client id of the App Registration, in Graph API tenant, that has the permissions to create/update subscriptions.
    - **GraphAPI-Certificate**: BASE64-encoded PFX certificate (with no password) that is used to authenticate the App Registration. This will be typically stored in a Key Vault. If the certificate is stored in a Key Vault, make sure the Managed Identity of the API Management instance has the "Key Vault Secret User" role on the Key Vault.
    - **EventHub-Namespace**: the namespace of the Event Hub where the notifications will be sent to. _Make sure the Managed Identity of the API Management instance has the necessary permissions on the Event Hub_.
    - **EventHub-Name**: the name of the Event Hub where the notifications will be sent to.

10. Go to API view, select `EventHub API`, select "All operations". In the "Inbound processing" section, click on the policy XML editor and replace all the text with the content of the `alloperations-policy.xml`. Click on "Save" button to save the policy.

11. Click on "POST lifecycle". In the "Inbound processing" section, click on the policy XML editor and replace all the text with the content of the `lifecycle-policy.xml`. Click on "Save" button to save the policy.

12. Click on "POST subscription". In the "Inbound processing" section, click on the policy XML editor and replace all the text with the content of the `subscription-policy.xml`. Click on "Save" button to save the policy.

## Creating a Graph API subscription for receiving notifications on messages received by a specific user

`POST https://graph.microsoft.com/v1.0/subscriptions`

```json
{
    "changeType": "created,updated",
    "notificationUrl": "https://<APIM-INSTANCE-NAME>.azure-api.net/subscription",
    "lifecycleNotificationUrl": "https://<APIM-INSTANCE-NAME>.azure-api.net/lifecycle",
    "resource": "users/<YOUR-USERNAME>@<YOUR-DOMAIN>/messages",
    "expirationDateTime":"2024-12-24T18:00:00Z",
    "clientState": "MyCustomClientState",
    "latestSupportedTlsVersion": "v1_2"
}
```

## Permissions recap

- **User issuing the initial Graph API subscription request**:  
  - Graph API _Delegated_ permission `Mail.Read` AND must have access to the specificed mailbox
- **Graph API App Registration**:  
  - Graph API _Application_ permission `Mail.Read`
- **API Management Managed Identity**:  
  - `Key Vault Secret User` role on the Key Vault where the certificate is stored and necessary permissions on the Event Hub namespace
  - `Azure Event Hubs Data Owner` role on the Event Hub

## Monitoring

Many aspects of this solutions can be easily monitored via Azure Alert Rule. I will provide here a solution to make sure that the subscription is kept alive and that the lifecycle notifications are received. This is crucial because a brake in the subscription will result in a loss of notifications with no explicit errors.

- Open the Application Insights instance associated with the API Management instance.
- Go to "Alerts" section.
- Click on "Create" -> "Alert rule".
  - Select "Custom log search" as the signal type
  - Set the following query as "Search query":

    ```sql
    requests | where name == "POST /lifecycle" and success == true
    ```

  - Measure: "Table rows"
  - Aggregation type: "Count"
  - Aggregation granularity: "1 hour"
  - Alert logic:
    - Operator: "Less than"
    - Threshold value: "1"
    - Frequency of evaluation: "1 hour"
- Select "Next" and choose the action group to be notified when the alert is triggered.
- In "Details" section, provide a name for the alert rule, a description and set the Severity to "1 - Error".
- Confirm the alert rule creation.

---
by [Gabriel Mercuri](gmercuri@microsoft.com) and [Luca Giuliani](giulianil@microsoft.com)
