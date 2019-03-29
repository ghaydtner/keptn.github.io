---
title: Runbook Automation and Self-healing
description: Gives an overview of how to leverage the power of runbook automation to build self-healing applications. Therefore, you will use automation tools for executing and managing the runbooks.
weight: 20
keywords: [self-healing]
aliases:
---

This use case gives an overview of how to leverage the power of runbook automation to build self-healing applications. Therefore, you will use ServiceNow workflows that are triggered to remediate incidents.

## About this use case

Configuration changes during runtime are sometimes necessary to increase flexibility. A prominent example are feature flags that can be toggled also in a production environment. In this use case we will change the promotion rate of a shopping cart service, which means that a defined percentage of interactions with the shopping cart will add promotional items (e.g., small gifts) to the shopping carts of our customers. However, we will experience troubles with this configuration change. Therefore, we will set means in place that are capable of auto-remediating issues at runtime. In fact, we will leverage workflows in ServiceNow. 

## Prerequisites

- ServiceNow instance or [free ServiceNow developer instance](https://developer.servicenow.com)
- Dynatrace Tenant [free trial](https://www.dynatrace.com/trial)
- Clone the GitHub repository with the necessary files for the use case:
    
    ```
    git clone --branch keptn-v0.2.x https://github.com/keptn/servicenow-service.git
    cd servicenow-service
    ```

## Configure keptn

In order for keptn to use both ServiceNow and Dynatrace, the corresponding credentials have to be stored as Kubernetes secrets in the cluster.
Please set the environment variables and adapt the following commands with your personal credentials:

```
export DT_TENANT_ID=xxx
export DT_API_TOKEN=xxx
```

Create Dynatrace secret to leverage the Dynatrace API.
```
kubectl -n keptn create secret generic dynatrace --from-literal="DT_TENANT_ID=$DT_TENANT_ID" --from-literal="DT_API_TOKEN=$DT_API_TOKEN"
```

Create ServiceNow secret to create/update incidents in ServiceNow and run workflows.
For the sake of simplicity, you can use your ServiceNow user, e.g., _admin_ as user and your ServiceNow password as the token.
```
kubectl -n keptn create secret generic servicenow --from-literal="tenant=xxx" --from-literal="user=xxx" --from-literal="token=xxx"
```

## Setup the Workflow in ServiceNow

A ServiceNow Update Set is provided to run this use case. To install the Update Set follow these steps:

1. Login to your ServiceNow instance.
1. Search for _update set_ in the left search box and navigate to **Update Sets to Commit** 
    {{< popup_image
    link="./assets/service-now-update-set-overview.png"
    caption="ServiceNow Update Set">}}

1. Click on **Import Update Set from XML** 

1. Import the file from your file system that you find in your `servicenow-service/usecase` folder: `keptn_demo_remediation_updateset.xml`

1. Open the Update Set
    {{< popup_image
    link="./assets/service-now-update-set-list.png"
    caption="ServiceNow Update Sets List">}}

1. In the right upper corner, click on **Preview Update Set** and once previewed, click on **Commit Update Set** to apply it to your instance
    {{< popup_image
    link="./assets/service-now-update-set-commit.png"
    caption="ServiceNow Update Set Commit">}}

1. After importing, enter **keptn** as the search term into the upper left search box.
    {{< popup_image 
    link="./assets/service-now-keptn-creds.png"
    caption="ServiceNow keptn credentials">}}
1. Click on **New** and enter your Dynatrace API token as well as your Dynatrace tenant ID.

1. _(Optional)_ You can also take a look at the predefined workflow that is able to handle Dynatrace problem notifications and remediate issues.
    - Navigate to the workflow editor by typing **Workflow Editor** and clicking on the item **Workflow -> Workflow Editor**
    - The workflow editor is opened in a new window/tab
    - Search for the workflow **keptn_demo_remediation** (it might as well be on the second or third page)
    {{< popup_image 
    link="./assets/service-now-workflow-list.png"
    caption="ServiceNow keptn workflow">}}
    - Open the workflow by clicking on it. It will look similar to the following image. By clicking on the workflow notes you can further investigate each step of the workflow.
    {{< popup_image 
    link="./assets/service-now-keptn-workflow.png"
    caption="ServiceNow keptn workflow">}}

## Setup a Dynatrace Problem Notification

In order to create incidents in ServiceNow and to trigger workflows, an integration with Dynatrace has to be set up.

1. Login to your Dynatrace tenant.
1. Navigate to **Settings -> Integration -> Problem Notifications**
1. Click on **Set up notifications** and select **Custom Notification**
1. Choose a name for your integration, e.g., _keptn integration_
1. In the webhook URL, paste the value of your keptn external eventbroker endpoint appended by `/dynatrace`, e.g., `https://event-broker-ext.keptn.XX.XXX.XXX.XX.xip.io/dynatrace`
    - Note: retrieve the base URL by running:

    ```
    kubectl get ksvc event-broker-ext -n keptn
    ```
    Please click the checkbox **Accept any SSL certificate**
1. Additionally, an Authorization Header is needed to authorize against the keptn server. 
    - Click on **Add header**
    - The name for the header is: `Authorization`
    - The value has to be set to the following: `Bearer KEPTN_API_TOKEN` where KEPTN_API_TOKEN has to be replaced with your actual Api Token that was received during installation. You can always retrieve the token again by executing:

    ```
    kubectl get secret keptn-api-token -n keptn -o=yaml | yq - r data.keptn-api-token | base64 --decode
    ```

1. As the custom payload, a valid Cloud Event has to be defined:

    ```json
    {
        "specversion":"0.2",
        "type":"sh.keptn.events.problem",
        "shkeptncontext":"{PID}",
        "source":"dynatrace",
        "id":"{PID}",
        "time":"",
        "contenttype":"application/json",
        "data": {
            "State":"{State}",
            "ProblemID":"{ProblemID}",
            "PID":"{PID}",
            "ProblemTitle":"{ProblemTitle}",
            "ProblemDetails":{ProblemDetailsJSON},
            "ImpactedEntities":{ImpactedEntities},
            "ImpactedEntity":"{ImpactedEntity}"
        }
    }
    ```

    {{< popup_image
    link="./assets/dynatrace-problem-notification-integration.png"
    caption="Dynatrace Problem Notification Integration">}}

## Adjust Anomaly Detection in Dynatrace

The Dynatrace platform is built on top of AI which is great for production use cases but for this demo we have to override some default settings in order for Dynatrace to trigger the problem.

1. Navigate to **Transaction & Services** and find the service **carts-ItemsController** in the _production_ namespace. 
2. Open the service and click on the three dots button to **Edit** the service.

    {{< popup_image
        link="./assets/dynatrace-service-edit.png"
        caption="Edit Service">}}

1. In the section **Anomaly detection** override the global anomaly detection and set the value for the **failure rate** to use **fixed thresholds** and to alert if **10%** custom failure rate are exceeded. Finally, set the **Sensitiviy** to **High**.
    {{< popup_image
        link="./assets/dynatrace-service-anomaly-detection.png"
        caption="Edit Anomaly Detection">}}

## Run the Use Case

Now that all pieces are in place we can run the use case. Therefore, we will start by generating some load on the `carts` service in our production environment. Afterwards we will change configuration of this service at runtime. This will cause some troubles in our production environment, Dynatrace will detect the issue and will create a problem ticket. Thanks to the problem notification we just set up, keptn will be informed about the problem and will forward it to the ServiceNow service that in turn creates an incident in ServiceNow. This incident will trigger a workflow that is able to remediate the issue at runtime. Along the remediation, comments and details on configuration changes are posted to Dynatrace.

### Load generation

1. Navigate to the _servicenow-service/usecase_ folder: 

    ```
    cd usecase
    ```
1. Run the script:

    ```
    ./add-to-cart.sh "carts.production.$(kubectl get svc istio-ingressgateway -n istio-system -o yaml | yq - r status.loadBalancer.ingress[0].ip).xip.io"
    ```
1. You should see some logging output each time an item is added to your shopping cart:

    ```
    ...
    adding item to cart...
    {"id":"3395a43e-2d88-40de-b95f-e00e1502085b","itemId":"03fef6ac-1896-4ce8-bd69-b798f85c6e0b","quantity":73,"unitPrice":0.0}
    adding item to cart...
    {"id":"3395a43e-2d88-40de-b95f-e00e1502085b","itemId":"03fef6ac-1896-4ce8-bd69-b798f85c6e0b","quantity":74,"unitPrice":0.0}
    adding item to cart...
    {"id":"3395a43e-2d88-40de-b95f-e00e1502085b","itemId":"03fef6ac-1896-4ce8-bd69-b798f85c6e0b","quantity":75,"unitPrice":0.0}
    ...
    ```
1. You will notice that your load generation script output will include some error messages after applying the script:
    ```
    ...
    adding item to cart...
    {"id":"3395a43e-2d88-40de-b95f-e00e1502085b","itemId":"03fef6ac-1896-4ce8-bd69-b798f85c6e0b","quantity":80,"unitPrice":0.0}
    adding item to cart...
    {"timestamp":1553686899190,"status":500,"error":"Internal Server Error","exception":"java.lang.Exception","message":"promotion campaign not yet implemented","path":"/carts/1/items"}
    adding item to cart...
    {"id":"3395a43e-2d88-40de-b95f-e00e1502085b","itemId":"03fef6ac-1896-4ce8-bd69-b798f85c6e0b","quantity":81,"unitPrice":0.0}
    ...
    ```


### Configuration change at runtime

1. Open another terminal to make sure the load generation is still running and again, navigate to the _servicenow-service/usecase_ folder.
1. _(Optional:)_ Verify that the environment variables you set earlier are still available:
    ```
    echo $DT_TENANT_ID
    echo $DT_API_TOKEN
    ```

1. Run the script:
    ```
    ./enable-promotion.sh "carts.production.$(kubectl get svc istio-ingressgateway -n istio-system -o yaml | yq - r status.loadBalancer.ingress[0].ip).xip.io" 30
    ```
    Please note the parameter `30` at the end, which is the value for the configuration change and can be interpreted as for 30 % of the shopping cart interactions a special item is added to the shopping cart. This value can be set from `0` to `100`. For this use case the value `30` is just fine.


### Problem Detections by Dynatrace

Validate in Dynatrace, that the configuration change has been applied.
    {{< popup_image
        link="./assets/dynatrace-config-event.png"
        caption="Dynatrace Custom Configuration Event">}}

After a couple of minutes, Dynatrace will open a problem ticket based on the increase of the failure rate.
    {{< popup_image
        link="./assets/dynatrace-problem-open.png"
        caption="Dynatrace Open Problem">}}



### Incident Creation & Workflow Execution by ServiceNow

The Dynatrace problem ticket notification is sent out to keptn which puts it into the problem channel where the ServiceNow service in subscribed. Thus, the ServiceNow service takes the event and creates a new incident in ServiceNow. 
In your ServiceNow instance, you can take a look at all incidents by typing in **incidents** in the top-left search box and click on **Service Desk -> Incidents**. You should be able to see the newly created incident, click on it to view some details.
    {{< popup_image
        link="./assets/service-now-incident.png"
        caption="ServiceNow incident">}}

After creation of the incident, a workflow is triggered in ServiceNow that has been setup during the import of the update set earlier. The workflow takes a look at the incident, resolves the URL that is stored in the _Remediation_ tab in the incident detail screen. Along with that, a new custom configuration change is sent to Dynatrace. Besides, the ServiceNow service running in keptn sends comments to the Dynatrace problem to be able to keep track of executed steps.

Once the problem is resolved, Dynatrace sends out another notification which again is handled by the ServiceNow service. Now the incidents gets resolved and another comment is sent to Dynatrace.
    {{< popup_image
        link="./assets/service-now-incident-resolved.png"
        caption="Resolved ServiceNow incident">}}


# Troubleshooting

- Please note that Dynatrace has its feature called **Frequent Issue Detection** enabled by default. This means, that if Dynatrace detects the same problem multiple times, it will be classified as a frequent issue and problem notifications won't be sent out to third party tools. Therefore, the use case might not be able to be run a couple of times in a row.

- In ServiceNow you can take a look at the **System Log -> All** to verify which actions have been executed. 
    {{< popup_image
        link="./assets/service-now-systemlog.png"
        caption="ServiceNow System Log">}}