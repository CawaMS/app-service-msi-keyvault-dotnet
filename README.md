---
services: app-service, key-vault
platforms: dotnet
author: varunsh-msft
---

# Use Key Vault from App Service with Managed Service Identity

## Background
For Service-to-Azure-Service authentication, the approach so far involved creating an Azure AD application and associated credential, and using that credential to get a token. The sample [here](https://docs.microsoft.com/en-us/azure/key-vault/key-vault-use-from-web-application) shows how this approach is used to authenticate to Azure Key Vault from a Web App. While this approach works well, there are two shortcomings:
1. The Azure AD application credentials are typically hard coded in source code. Developers tend to push the code to source repositories as-is, which leads to credentials in source.
2. The Azure AD application credentials expire, and so need to be renewed, else can lead to application downtime.

With [Managed Service Identity (MSI)](https://docs.microsoft.com/en-us/azure/app-service/app-service-managed-service-identity), both these problems are solved. This sample shows how a Web App can authenticate to Azure Key Vault without the need to explicitly create an Azure AD application or manage its credentials.

>Here's another sample that shows how to programatically deploy an ARM template from a .NET Console application running on an Azure VM with a Managed Service Identity (MSI) - [https://github.com/Azure-Samples/windowsvm-msi-arm-dotnet](https://github.com/Azure-Samples/windowsvm-msi-arm-dotnet)

## Prerequisites
To run and deploy this sample, you need the following:
1. An Azure subscription to create an App Service and a Key Vault.
    * [Create your Azure free account today](https://azure.microsoft.com/en-us/free/)
2. [Visual Studio 2017](https://www.visualstudio.com/)
    * Install Web development and Azure development workloads.    
3. [Azure Services Authentication Extension](https://go.microsoft.com/fwlink/?linkid=862354)
4. [.NET Framework 4.7.1](https://www.microsoft.com/en-us/download/details.aspx?id=56115)

## Step 1: Create an App Service with a Managed Service Identity (MSI)
<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure-Samples%2Fapp-service-msi-keyvault-dotnet%2Fmaster%2Fazuredeploy.json" target="_blank">
    <img src="http://azuredeploy.net/deploybutton.png"/>
</a>

Use the "Deploy to Azure" button to deploy an ARM template to create the following resources:
1. App Service with MSI.
2. Key Vault with a secret, and an access policy that grants the App Service access to **Get Secrets**.
>Note: When filling out the template you will see a textbox labelled 'Key Vault Secret'. Enter a secret value there. A secret with the name 'secret' and value from what you entered will be created in the Key Vault.

Review the resources created using the Azure portal. You should see an App Service and a Key Vault. View the access policies of the Key Vault to see that the App Service has access to it.

## Step 2: Grant yourself data plane access to the Key Vault
Using the Azure Portal, go to the Key Vault's access policies, and grant yourself **Secret Management** access to the Key Vault. This will allow you to run the application on your local development machine.

1.	Search for your Key Vault in “Search Resources dialog box” in Azure Portal.
2.	Select "Overview", and click on Access policies
3.	Click on "Add New", select "Secret Management" from the dropdown for "Configure from template"
4.	Click on "Select Principal", add your account
5.	Save the Access Policies

## Step 3: Clone the repo
Clone the repo to your development machine.

The project has two relevant Nuget packages:
1. Microsoft.Configuration.ConfigurationBuilders.Azure - Enables an application to use Key Vault as a configuration store without changing source code
2. Microsoft.Configuration.ConfigurationBuilders - Enables an application to use additional configuration stores other than web.config file

The HomeController Index class can use the exact same old way to access a configuration setting through ConfigurationManager class, without any code change

```csharp    
public async System.Threading.Tasks.Task<ActionResult> Index()
       {
           AzureServiceTokenProvider azureServiceTokenProvider = new AzureServiceTokenProvider();

           try
           {

               var secret = ConfigurationManager.AppSettings["secret"];

               ViewBag.Secret = $"Secret: {secret}";

           }
```

In web.config file, specify the name of the Key Vault and use the Key Vault configuration builder in appSettings section of web.config file

```xml
<configBuilders>
   <builders>
     <add name="Environment" type="Microsoft.Configuration.ConfigurationBuilders.EnvironmentConfigBuilder, Microsoft.Configuration.ConfigurationBuilders, Version=1.0.0.0, Culture=neutral" />
     <add name="Secrets" secretsFile="C:\Users\cawa\Documents\secret.xml" type="Microsoft.Configuration.ConfigurationBuilders.UserSecretsConfigBuilder, Microsoft.Configuration.ConfigurationBuilders, Version=1.0.0.0, Culture=neutral" />
     <add name="AzureKeyVault" vaultName="replace_with_your_KeyVault_Name" type="Microsoft.Configuration.ConfigurationBuilders.AzureKeyVaultConfigBuilder, Microsoft.Configuration.ConfigurationBuilders.Azure, Version=1.0.0.0, Culture=neutral" />
   </builders>
 </configBuilders>
 <appSettings configBuilders="AzureKeyVault">
   <add key="webpages:Version" value="3.0.0.0" />
   <add key="webpages:Enabled" value="false" />
   <add key="ClientValidationEnabled" value="true" />
   <add key="UnobtrusiveJavaScriptEnabled" value="true" />
   <add key="secret" value="" />
 </appSettings>
```
The configBuilders section defines additional configuration stores. For example, it can be environment variables, a file that's saved outside of source control folder, or a Key Vault for secret settings. In appSettings section, specify which configuration builder to use by using the name attribute of the builder. The value in the additional configuration store will replace the value of an entry listed in the appSettings section. For example, the setting "secret" with value equals to an emptry string will be replaced by the actual secret value from the Key Vault.

Depending on your user scenario, if you are just doing a quick prototype and don't want to provision any Azure resources, use the Secrets configuration builder which can save secrets to any file specified in the secretFile attribute. This can help prevent secrets to be checked in to source control and leaked on Github.

## Step 4: Run the application on your local development machine
The Key Vault configuration builder will use the developer's security context to get a token to authenticate to Key Vault. This removes the need to create a service principal, and share it with the development team. It also prevents credentials from being checked in to source code.  

Since your developer account has access to the Key Vault, you should see the secret on the web page. Principal Used will show type "User" and your user account.


## Step 6: Deploy the Web App to Azure
Use any of the methods outlined on [Deploy your app to Azure App Service](https://docs.microsoft.com/en-us/azure/app-service-web/web-sites-deploy) to publish the Web App to Azure.
After you deploy it, browse to the web app. You should see the secret on the web page, and this time the Principal Used will show "App", since it ran under the context of the App Service.
The AppId of the MSI will be displayed.

## Summary
The web app was successfully able to get a secret at runtime from Azure Key Vault using your developer account during development, and using MSI when deployed to Azure, without any code change between local development environment and Azure.
As a result, you did not have to explicitly handle a service principal credential to authenticate to Azure AD to get a token to call Key Vault. You do not have to worry about renewing the service principal credential either, since MSI takes care of that.  


## Troubleshooting

### Common issues when deployed to Azure App Service:

1. MSI is not setup on the App Service.

Check the environment variables MSI_ENDPOINT and MSI_SECRET exist using [Kudu debug console](https://azure.microsoft.com/en-us/resources/videos/super-secret-kudu-debug-console-for-azure-web-sites/). If these environment variables do not exist, MSI is not enabled on the App Service.

### Common issues across environments:

1. Access denied

The principal used does not have access to the Key Vault. The principal used in show on the web page. Grant that user (in case of developer context) or application "Get secret" access to the Key Vault.
