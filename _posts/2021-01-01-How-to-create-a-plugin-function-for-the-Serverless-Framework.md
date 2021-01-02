---
title: "How to create a plugin function for the Serverless Framework"
layout: posts
categories:
tags: 
  - serverless
  - Serverless Framework
---
This wiki will use the exists plugging to hook the step and run your function.

Before you start to deploy a plugin function for the serverless framework, you can read the below 2 articles. It will be very helpful for you to know the complete flow for serverless:

https://www.serverless.com/blog/writing-serverless-plugins
https://www.serverless.com/blog/writing-serverless-plugins-2

### Our use case
In the serverless project, we want to use the existing API documentation to create the API gateway.

In this way, we can reduce the learning curve to use the serverless default function to define the API gateway, but we need to control "AWS::ApiGateway::Deployment" in the serverless deployment.



If we use the serverless default function to create the API gateway with lambda, the serverless will help us to create a new API gateway deployment resource.

But in our case, we need to update the resource name for each API deployment.



There is a serverless framework plugging, serverless-scriptable-plugin, can help us to custom our deploy process without to create a new plugin.

We need to define the plugging that we use and define the javascript function that should be run at which Lifecycle step.

In our case, we will run force-update-deployment.js after the step, aws:package:finalize:mergeCustomProviderResources.

serverless.yaml
custom:
scriptHooks:
after:aws:package:finalize:mergeCustomProviderResources: build/serverless/force-update-deployment.js

plugins:
- serverless-scriptable-plugin

In force-update-deployment.js, we will find AWS::ApiGateway::Deployment and update the resource name with timestamp

force-update-deployment.js
```html
const resources = serverless.service.provider.compiledCloudFormationTemplate.Resources;
Object.keys(resources)
    .filter(name => resources[name].Type === 'AWS::ApiGateway::Deployment')
    .forEach(
        name => {
            resources[name+Date.now()] = resources[name]
            delete resources[name]
        }
    )
console.log(resources)

```

We can find the resource ApiGatewayDeployment will be changed to ApiGatewayDeployment1608281710995.

So we can keep deploying the serverless without changing AWS::ApiGateway::Deployment manually for each deployment.

![image] (/assets/images/deploy_apigateway_with_plugin_sceenshot.png)

### Serverless Framework LifecycleEvents for AWS
If we want to deploy a serverless, there are some steps for a whole serverless lifecycle.

We have to know which step we need to hook and run the function.

Defining the commands and their lifecycle events is useful to describe what the plugin does. Hooks describe how the plugin does it.

| Lifecycle         | Lifecycle events                   | Lifecycle command                                                                                                                                                          |   |   |
|-------------------|------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---|---|
| package           |                                    |                                                                                                                                                                            |   |   |
|                   | package:cleanup                    |                                                                                                                                                                            |   |   |
|                   |                                    | aws:common:validate:validate aws:common:cleanupTempDir:cleanup                                                                                                             |   |   |
|                   | package:initialize                 |                                                                                                                                                                            |   |   |
|                   | package:setupProviderConfiguration |                                                                                                                                                                            |   |   |
|                   | package:createDeploymentArtifacts  |                                                                                                                                                                            |   |   |
|                   | package:compileFunctions           |                                                                                                                                                                            |   |   |
|                   | package:compileEvents              |                                                                                                                                                                            |   |   |
|                   | package:finalize                   |                                                                                                                                                                            |   |   |
|                   |                                    | aws:package:finalize:mergeCustomProviderResources aws:package:finalize:saveServiceState aws:common:moveArtifactsToPackage:move                                             |   |   |
| deploy            |                                    |                                                                                                                                                                            |   |   |
|                   | before:deploy:deploy               |                                                                                                                                                                            |   |   |
|                   |                                    | aws:common:validate:validate aws:common:moveArtifactsToTemp:move                                                                                                           |   |   |
|                   | deploy:deploy                      |                                                                                                                                                                            |   |   |
|                   |                                    | aws:deploy:deploy:createStack aws:deploy:deploy:checkForChanges aws:deploy:deploy:uploadArtifacts aws:deploy:deploy:validateTemplate aws:deploy:deploy:updateStack         |   |   |
|                   | deploy:finalize                    |                                                                                                                                                                            |   |   |
|                   |                                    | aws:deploy:finalize:cleanup                                                                                                                                                |   |   |
| deploy function   |                                    |                                                                                                                                                                            |   |   |
|                   | deploy:function:initialize         |                                                                                                                                                                            |   |   |
|                   | deploy:function:packageFunction    |                                                                                                                                                                            |   |   |
|                   |                                    | package:function:package                                                                                                                                                   |   |   |
|                   | deploy:function:deploy             |                                                                                                                                                                            |   |   |
|                   |                                    | aws:common:cleanupTempDir:cleanup                                                                                                                                          |   |   |
| deploy list       |                                    |                                                                                                                                                                            |   |   |
|                   | deploy:list:log                    |                                                                                                                                                                            |   |   |
| info              |                                    |                                                                                                                                                                            |   |   |
|                   | info:info                          |                                                                                                                                                                            |   |   |
|                   |                                    | aws:info:validate aws:info:gatherData aws:info:displayServiceInfo aws:info:displayApiKeys aws:info:displayEndpoints aws:info:displayFunctions aws:info:displayStackOutputs |   |   |
| remove            |                                    |                                                                                                                                                                            |   |   |
|                   | remove:remove                      |                                                                                                                                                                            |   |   |
| rollback          |                                    |                                                                                                                                                                            |   |   |
|                   | rollback:initialize                |                                                                                                                                                                            |   |   |
|                   | rollback:rollback                  |                                                                                                                                                                            |   |   |
| rollback function |                                    |                                                                                                                                                                            |   |   |
|                   | rollback:function:rollback         |                                                                                                                                                                            |   |   |
| logs              |                                    |                                                                                                                                                                            |   |   |
|                   | logs:logs                          |                                                                                                                                                                            |   |   |
| invoke            |                                    |                                                                                                                                                                            |   |   |
|                   | invoke:invoke                      |                                                                                                                                                                            |   |   |
| invoke local      |                                    |                                                                                                                                                                            |   |   |
|                   | invoke:local:loadEnvVars           |                                                                                                                                                                            |   |   |
|                   | invoke:local:invoke                |                                                                                                                                                                            |   |   |
| create            |                                    |                                                                                                                                                                            |   |   |
|                   | create:create                      |                                                                                                                                                                            |   |   |
| config            |                                    |                                                                                                                                                                            |   |   |
|                   | config:credentials:config          |                                                                                                                                                                            |   |   |
| install           |                                    |                                                                                                                                                                            |   |   |
|                   | install:install                    |                                                                                                                                                                            |   |   |
| login             |                                    |                                                                                                                                                                            |   |   |
|                   | login:login                        |                                                                                                                                                                            |   |   |
| logout            |                                    |                                                                                                                                                                            |   |   |
|                   | logout:logout                      |                                                                                                                                                                            |   |   |
| metrics           |                                    |                                                                                                                                                                            |   |   |
|                   | metrics:metrics                    |                                                                                                                                                                            |   |   |
| slstats           |                                    |                                                                                                                                                                            |   |   |
|                   | slstats:slstats                    |                                                                                                                                                                            |   |   |

### Reference
LifecycleEvents:

https://gist.github.com/HyperBrain/50d38027a8f57778d5b0f135d80ea406

writing-serverless-plugins:

https://www.serverless.com/blog/writing-serverless-plugins

https://www.serverless.com/blog/writing-serverless-plugins-2