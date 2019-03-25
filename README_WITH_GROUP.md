## ARM template for use with preexisting Resource Group

Example command with preexisting server farm:
```bash
az group deployment create --name <name-of-deployment> --template-file template-with-preexisting-group.json --resource-group <name-or-id-of-group> --subscription <app-guid> --parameters appId=<msa-app-guid> appSecret="<msa-app-password>" botId=<id-or-name-of-bot> existingServerFarm=<name-of-apps-service-plan-in-group> newWebAppName=<name-of-web-app>
```

Expected order of commands when starting from sample:
```bash
az group deployment
az bot prepare-deploy
# User zips up code manually
az webapp deployment source config-zip
```

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "appId": {
            "type": "string"
        },
        "appSecret": {
            "type": "string",
            "defaultValue": ""
        },
        "botId": {
            "type": "string"
        },
        "botSku": {
            "defaultValue": "F0",
            "type": "string"
        },
        "newServerFarmName": {
            "type": "string",
            "defaultValue": ""
        },
        "newServerFarmSku": {
            "type": "object",
            "defaultValue": {
                "name": "S1",
                "tier": "Standard",
                "size": "S1",
                "family": "S",
                "capacity": 1
            }
        },
        "newServerFarmLocation": {
            "type": "string",
            "defaultValue": "westus"
        },
        "existingServerFarm": {
            "type": "string",
            "defaultValue": ""
        },
        "newWebAppName": {
            "type": "string",
            "defaultValue": ""
        }
    },
    "variables": {
        "defaultServerName": "[if(empty(parameters('existingServerFarm')), 'createNewServer', parameters('existingServerFarm'))]",
        "useExistingServerFarm": "[not(equals(variables('defaultServerName'), 'createNewServer'))]",
        "serverFarmName": "[if(variables('useExistingServerFarm'), parameters('existingServerFarm'), parameters('newServerFarmName'))]",
        "resourcesLocation": "[parameters('newServerFarmLocation')]",
        "webAppName": "[if(empty(parameters('newWebAppName')), parameters('botId'), parameters('newWebAppName'))]",
        "siteHost": "[concat(parameters('newWebAppName'), '.azurewebsites.net')]",
        "botEndpoint": "[concat('https://', variables('siteHost'), '/api/messages')]"
    },
    "resources": [
        {
            "comments": "Create a new Server Farm if no existing Server Farm name was passed in.",
            "type": "Microsoft.Web/serverfarms",
            "condition": "[not(variables('useExistingServerFarm'))]",
            "name": "[variables('serverFarmName')]",
            "apiVersion": "2018-02-01",
            "location": "[variables('resourcesLocation')]",
            "sku": "[parameters('newServerFarmSku')]",
            "properties": {
                "name": "[variables('serverFarmName')]"
            }
        },
        {
            "comments": "Create a Web App using a Server Farm",
            "type": "Microsoft.Web/sites",
            "apiVersion": "2015-08-01",
            "location": "[variables('resourcesLocation')]",
            "kind": "app",
            "dependsOn": [
                "[variables('serverFarmName')]"
            ],
            "name": "[variables('webAppName')]",
            "properties": {
                "name": "[variables('webAppName')]",
                "serverFarmId": "[variables('serverFarmName')]",
                "siteConfig": {
                    "appSettings": [
                        {
                            "name": "WEBSITE_NODE_DEFAULT_VERSION",
                            "value": "10.14.1"
                        },
                        {
                            "name": "SCM_DO_BUILD_DURING_DEPLOYMENT",
                            "value": "false"
                        },
                        {
                            "name": "BotEnv",
                            "value": "[parameters('botEnv')]"
                        },
                        {
                            "name": "MicrosoftAppId",
                            "value": "[parameters('appId')]"
                        },
                        {
                            "name": "MicrosoftAppPassword",
                            "value": "[parameters('appSecret')]"
                        }
                    ],
                    "cors": {
                        "allowedOrigins": [
                            "https://botservice.hosting.portal.azure.net",
                            "https://hosting.onecloud.azure-test.net/"
                        ]
                    }
                }
            }
        },
        {
            "apiVersion": "2017-12-01",
            "type": "Microsoft.BotService/botServices",
            "name": "[parameters('botId')]",
            "location": "global",
            "kind": "bot",
            "sku": {
                "name": "[parameters('botSku')]"
            },
            "properties": {
                "name": "[parameters('botId')]",
                "displayName": "[parameters('botId')]",
                "endpoint": "[variables('botEndpoint')]",
                "msaAppId": "[parameters('appId')]",
                "developerAppInsightsApplicationId": null,
                "developerAppInsightKey": null,
                "publishingCredentials": null,
                "storageResourceId": null
            },
            "dependsOn": [
                "[resourceId('Microsoft.Web/sites/', variables('webAppName'))]"
            ]
        }
    ]
}
```

### Parameters:
```json
"parameters": {
    "appId": {
        "type": "string"
    },
    "appSecret": {
        "type": "string",
        "defaultValue": ""
    },
    "botId": {
        "type": "string"
    },
    "botSku": {
        "defaultValue": "F0",
        "type": "string"
    },
    "newServerFarmName": {
        "type": "string",
        "defaultValue": ""
    },
    "newServerFarmSku": {
        "type": "object",
        "defaultValue": {
            "name": "S1",
            "tier": "Standard",
            "size": "S1",
            "family": "S",
            "capacity": 1
        }
    },
    "newServerFarmLocation": {
        "type": "string",
        "defaultValue": "westus"
    },
    "existingServerFarm": {
        "type": "string",
        "defaultValue": ""
    },
    "newWebAppName": {
        "type": "string",
        "defaultValue": ""
    }
}
```