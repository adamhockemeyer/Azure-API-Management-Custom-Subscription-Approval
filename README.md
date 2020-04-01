# Azure API Management Custom Subscription Approval
An example of how to create a custom workflow for Azure API Management product subscriptions approvals.

## Introduction

[Azure API Management](https://azure.microsoft.com/en-us/services/api-management/ "Azure API Management") offers a lot of really great features for organizations to organize and access various systems by surfacing API's through a common platform both internally and externally.  Azure API Management (APIM for short) allows API publishers the ability to expose just an API, or a group of API's known as a product.  For example, a food truck service may want to expose an 'Order' product, but that 'product' may be made up of API's responsible for creating user accounts as well as actually placing an order.

As your company continues to adopt APIM, you may have considered if you should have multiple APIM instances or if you can use a shared APIM instance that all of the organizations API's and products go into.  A feature of APIM products' is that you can require users of your API to register for a subscription to use your API, and you can also set whether that subscription should be automatically approved or if an admin or some other user should be the approver.  The tricky issue here is that if your products are managed by different teams all using a shared APIM instance, then each team may require a unique group or user/users that should be the approvers for a specific product.

This guide will walk through how to create a Logic App that will allow for a custom subscription approval flow per product in Azure API Management.

## 1 - API Management

This guide isn't meant to go in depth about Azure API Management, but more around the specific use case mentioned above.  To follow along you will need an Azure API Management instance setup as well as an Azure Logic App.

Either create a new product or use an existing product that you have 'Requires Subscription' and 'Requires Approval' enabled.

![API Management Settings][APIM_1]

When 'Requires Approval' is checked, APIM can send an email to notify whoever you have setup in the 'Notifications' --> 'New Subscriptions' section of the portal.  This however doesn't differentiate by product as we mentioned in the introduction. If 'Adam' is set to receive the 'New Subscription' emails, but he doesn't know anything about the the product someone is requesting a subscription to, then he would have to track down someone and see if he should approve it or not.  You could add a bunch of different folks as approvers, but again people would be getting asked to approve for something that they are maybe not the owner of.

![API Management Notifications][APIM_2]

Now that we have a product that requires a subscription and approval, lets setup a logic app to do a custom approval workflow.

## 2 - Logic App

Create a new blank Logic App in the Azure portal if you haven't already.  We are going to use the [Azure Management REST API](https://docs.microsoft.com/en-us/rest/api/apimanagement/) to do the following:

1. Poll for API Management subscriptions that are in a 'submitted' state.
2. Filter the results by the product(scope) that the subscription is for.
3. Loop through each pending subscription and email the appropriate approver for the specific product.
4. If approver approved, call the REST API and update the state and notify the end user.

Here is a high level view of the Logic App, and then we will dig into each part.

![Logic App Overview][LOGIC_APP_1]

### 2.1 - Sliding Window Trigger

Create a Sliding Window trigger.  If you select for example a 5 minute window, the sliding trigger would start at 10:00AM and finish at 10:05AM and the Sliding Window trigger will output 2 variables with the start and end times.  We will use this 'window' later in the approval flow to make sure we are only sending new notifications and not ones that have already been sent.

![Logic App Trigger][LOGIC_APP_2]

### 2.2 HTTP Action (Get API Management Subscriptions - Submitted State)

Now that the Logic App as started from the Sliding Window trigger, we will call to the Azure Management REST API for API Management and list all of the subscriptions that are in the 'submitted' state.  To form the API call you will need to pass in a few variables.

![Logic App HTTP][LOGIC_APP_3]

The API endpoint is:
`https://management.azure.com/subscriptions/@{parameters('SubscriptionId')}/resourceGroups/@{parameters('ResourceGroupName')}/providers/Microsoft.ApiManagement/service/@{parameters('APIMServiceName')}/subscriptions?$filter=state eq 'submitted'&api-version=2019-01-01`

Since the Azure Management REST API is very powerful, you need to be authenticated to make a call to it.  I chose to use Managed Identity, but you could also use a OAuth Bearer token among others.  Managed Identity is nice because you don't need to deal with tokens.  To use Managed Identity, you would need to create a new Managed Identity in the Azure portal, and then in your API Management portal, under 'Access Control (IAM)' assign the Managed Identity you created to have access.

### 2.3 Parse JSON

The next step is to parse the JSON, which will make it easy to work with the results of the HTTP call later.

![Logic App Parse JSON][LOGIC_APP_4]

The schema I used was:

```
{
    "properties": {
        "count": {
            "type": "integer"
        },
        "value": {
            "items": {
                "properties": {
                    "id": {
                        "type": "string"
                    },
                    "name": {
                        "type": "string"
                    },
                    "properties": {
                        "properties": {
                            "allowTracing": {
                                "type": "boolean"
                            },
                            "createdDate": {
                                "type": "string"
                            },
                            "displayName": {},
                            "endDate": {},
                            "expirationDate": {},
                            "notificationDate": {},
                            "ownerId": {
                                "type": "string"
                            },
                            "primaryKey": {
                                "type": "string"
                            },
                            "scope": {
                                "type": "string"
                            },
                            "secondaryKey": {
                                "type": "string"
                            },
                            "startDate": {},
                            "state": {
                                "type": "string"
                            },
                            "stateComment": {}
                        },
                        "type": "object"
                    },
                    "type": {
                        "type": "string"
                    }
                },
                "required": [
                    "id",
                    "type",
                    "name",
                    "properties"
                ],
                "type": "object"
            },
            "type": "array"
        }
    },
    "type": "object"
}
```

### 2.4 Filter by Product

Now that we have the 'submitted' subscriptions, we can filter by the API Management 'product' based on the scope property.

![Logic App Filter][LOGIC_APP_5]

![Logic App Filter][LOGIC_APP_6]

### 2.5 Loop Each 'Submitted' Subscription

The next step is to loop through each 'submitted' product subscription, and send an email to an approver for the given product.  Below is a zoomed out view of what the loop flow looks like in its entirety before walking through each step.  In this example I am only taking an action if the request was approved.  This would likely need to be extended to handle if the request was rejected or also an additional or seperate peice of logic to clean up 'old' submissions (i.e. expire them, remind the approver, etc.)

![Logic App Subscription Loop][LOGIC_APP_7]

### 2.5.1 Only check for new submissions

A benefit of using the 'Sliding Window' trigger to kick this entire logic app off, is that it has two output variables that result from the trigger running.  We can use 'Start Time' and 'End Time' from the 'Sliding Window' trigger to ensure we are only looking at a product subscriptions for a given time frame.  In this example, the Logic App runs every 5 minutes, and then we will use this 5 minute period of time to see if any new product subscriptions were created in that time which need to be sent to an approver.

![Logic App Subscription Dates][LOGIC_APP_8]

If the result is 'true', then we have recieved a new product subscription within the 5 minute sliding window.

### 2.5.2 Send product subscription approval email

Next we will send an email to an appover of our choice (woohoo!) for the particular product flow we are working on.  I used the built in Office 365 'Send Approval Email', but you could also format up your own email and have buttons in the email that trigger an appropriate action.

![Logic App Subscription Dates][LOGIC_APP_9]

### 2.5.3 Check if the subscription was approved

Once the approver makes an action (Approve/Reject in this case), the next step of the Logic App is to handle the decision.  This example only handles the approval, but you would likely want to handle the rejections as well.  Add a condition action to check for the appropriate status.

![Logic App Subscription Approval][LOGIC_APP_10]

### 2.5.4 Update the product subscription via REST API

Now that the approver took 'Approve' action, call the API Management REST API and update the status for the subscription to 'active'.

![Logic App Approval Update][LOGIC_APP_11]

The API endpoint is:
`https://management.azure.com/subscriptions/@{parameters('SubscriptionId')}/resourceGroups/@{parameters('ResourceGroupName')}/providers/Microsoft.ApiManagement/service/@{parameters('APIMServiceName')}/subscriptions/@{items('For_each')?['name']}?api-version=2019-01-01&notify=true`

Headers:
`Content-Type: application/json`
`If-Match: *`

Body:
```
{
  "properties": {
    "state": "active"
  }
}
```

And thats it!  You have successfully created a custom approval flow for product subscriptions.



[APIM_1]: https://github.com/adamhockemeyer/Azure-API-Management-Custom-Subscription-Approval/blob/master/images/APIM_1.jpg "API Management Settings"
[APIM_2]: https://github.com/adamhockemeyer/Azure-API-Management-Custom-Subscription-Approval/blob/master/images/APIM_2.jpg "API Management Notifications"
[LOGIC_APP_1]: https://github.com/adamhockemeyer/Azure-API-Management-Custom-Subscription-Approval/blob/master/images/LOGIC_APP_1.jpg "Logic App Overview"
[LOGIC_APP_2]: https://github.com/adamhockemeyer/Azure-API-Management-Custom-Subscription-Approval/blob/master/images/LOGIC_APP_2.jpg "Logic App Trigger"
[LOGIC_APP_3]: https://github.com/adamhockemeyer/Azure-API-Management-Custom-Subscription-Approval/blob/master/images/LOGIC_APP_3.jpg "Logic App HTTP"
[LOGIC_APP_4]: https://github.com/adamhockemeyer/Azure-API-Management-Custom-Subscription-Approval/blob/master/images/LOGIC_APP_4.jpg "Logic App Parse JSON"
[LOGIC_APP_5]: https://github.com/adamhockemeyer/Azure-API-Management-Custom-Subscription-Approval/blob/master/images/LOGIC_APP_5.jpg "Logic App Filter Products"
[LOGIC_APP_6]: https://github.com/adamhockemeyer/Azure-API-Management-Custom-Subscription-Approval/blob/master/images/LOGIC_APP_6.jpg "Logic App Filter Products"
[LOGIC_APP_7]: https://github.com/adamhockemeyer/Azure-API-Management-Custom-Subscription-Approval/blob/master/images/LOGIC_APP_7.jpg "Logic App Filtered Product Subscriptions Loop"
[LOGIC_APP_8]: https://github.com/adamhockemeyer/Azure-API-Management-Custom-Subscription-Approval/blob/master/images/LOGIC_APP_8.jpg "Logic App Filtered Product Subscriptions Loop Date Condition"
[LOGIC_APP_9]: https://github.com/adamhockemeyer/Azure-API-Management-Custom-Subscription-Approval/blob/master/images/LOGIC_APP_9.jpg "Logic App Send Approval Email"
[LOGIC_APP_10]: https://github.com/adamhockemeyer/Azure-API-Management-Custom-Subscription-Approval/blob/master/images/LOGIC_APP_10.jpg "Logic App Approval"
[LOGIC_APP_11]: https://github.com/adamhockemeyer/Azure-API-Management-Custom-Subscription-Approval/blob/master/images/LOGIC_APP_11.jpg "Logic App Approval Update"
