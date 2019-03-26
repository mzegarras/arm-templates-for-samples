# arm-templates-for-samples
For soliciting feedback on ARM templates

Example command:
```bash
az deployment create --name <name-of-deployment> --template-file all-up-template.json --resource-group <name-or-id-of-group> --subscription <app-guid> --parameters appId=<msa-app-guid> appSecret="<msa-app-password>" botId=<id-or-name-of-bot> newServerFarmName=<name-of-server-farm> newWebAppName=<name-of-web-app>
```

Expected order of commands when starting from sample:
```bash
# 1. User plays around with sample, makes code changes

# 2. User decides to provision resources via ARM
az deployment create ...

# 3. User retrieves necessary IIS/Kudu files
az bot prepare-deploy ...

# 4. User zips up code manually

# 5. User deploys code to Azure using az webapp
az webapp deployment source config-zip ...
```

### Inline ARM template ([link](/all-up-template.json))

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "groupLocation": {
            "defaultValue": "westus",
            "type": "string"
        },
        "groupName": {
            "type": "string"
        },
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
            "type": "string"
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
        "newWebAppName": {
            "type": "string",
            "defaultValue": ""
        },
        "alwaysBuildOnDeploy": {
            "type": "bool",
            "defaultValue": "false",
            "metadata": {
                "comments": "Configures environment variable SCM_DO_BUILD_DURING_DEPLOYMENT on Web App. When set to true, the Web App will automatically build or install NPM packages when a deployment occurs."
            }
        }
    },
    "variables": {
        "serverFarmName": "[parameters('newServerFarmName')]",
        "resourcesLocation": "[parameters('newServerFarmLocation')]",
        "webAppName": "[if(empty(parameters('newWebAppName')), parameters('botId'), parameters('newWebAppName'))]",
        "siteHost": "[concat(variables('webAppName'), '.azurewebsites.net')]",
        "botEndpoint": "[concat('https://', variables('siteHost'), '/api/messages')]"
    },
    "resources": [
        {
            "name": "[parameters('groupName')]",
            "type": "Microsoft.Resources/resourceGroups",
            "apiVersion": "2018-05-01",
            "location": "[parameters('groupLocation')]",
            "properties": {
            }
        },
        {
            "type": "Microsoft.Resources/deployments",
            "apiVersion": "2018-05-01",
            "name": "storageDeployment",
            "resourceGroup": "[parameters('groupName')]",
            "dependsOn": [
                "[resourceId('Microsoft.Resources/resourceGroups/', parameters('groupName'))]"
            ],
            "properties": {
                "mode": "Incremental",
                "template": {
                    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
                    "contentVersion": "1.0.0.0",
                    "parameters": {},
                    "variables": {},
                    "resources": [
                        {
                            "comments": "Create a new Server Farm",
                            "type": "Microsoft.Web/serverfarms",
                            "condition": "[variables('serverFarmName')]",
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
                                            "value": "[parameters('alwaysBuildOnDeploy')]"
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
                    ],
                    "outputs": {}
                }
            }
        }
    ]
}
```

### Parameters:
```json
"parameters": {
    "groupLocation": {
        "defaultValue": "westus",
        "type": "string"
    },
    "groupName": {
        "type": "string"
    },
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
        "type": "string"
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
    "newWebAppName": {
        "type": "string",
        "defaultValue": ""
    },
    "alwaysBuildOnDeploy": {
        "type": "bool",
        "defaultValue": "false",
        "metadata": {
            "comments": "Configures environment variable SCM_DO_BUILD_DURING_DEPLOYMENT on Web App. When set to true, the Web App will automatically build or install NPM packages when a deployment occurs."
        }
    }
}
```