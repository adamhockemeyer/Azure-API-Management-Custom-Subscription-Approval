# Azure API Management Custom Subscription Approval
An example of how to create a custom workflow for Azure API Management product subscriptions approvals.

## Introduction

[Azure API Management](https://azure.microsoft.com/en-us/services/api-management/ "Azure API Management") offers a lot of really great features for organizations to organize and access various systems by surfacing API's through a common platform both internally and externally.  Azure API Management (APIM for short) allows API publishers the ability to expose just an API, or a group of API's known as a product.  For example, a food truck service may want to expose an 'Order' product, but that 'product' may be made up of API's responsible for creating user accounts as well as actually placing an order.

As your company continues to adopt APIM, you may have considered if you should have multiple APIM instances or if you can use a shared APIM instance that all of the organizations API's and products go into.  A feature of APIM products' is that you can require users of your API to register for a subscription to use your API, and you also also set whether that subscription should be automatically approved or if an admin or some other user should be the approver.  The tricky issue here is that if your products are managed by different teams, then each team may require a unique group or user/users that should be the approvers for a specific product.

This guide will walk through how to create a Logic App that will allow for a custom subscription approval flow per product in Azure API Management.

## 1 - API Management

This guide isn't meant to go in depth about Azure API Management, but more around the specific use case mentioned above.  To follow along you will need an Azure API Management instance setup as well as an Azure Logic App.

Either create a new product or use an existing product that you have 'Requires Subscription' and 'Requires Approval' enabled.

![API Management Settings][APIM_1]

When 'Requires Approval' is checked, APIM can send an email to notify whoever you have setup in the 'Notifications' --> 'New Subscriptions' section of the portal.  This however doesn't differentiate by product as we mentioned in the introduction. If 'Adam' is set to receive the 'New Subscription' emails, but he doesn't know anything about the the product someone is requesting a subscription to, then he would have to track down someone and see if he should approve it or not.  You could add a bunch of different folks as approvers, but again people would be getting asked to approve for something that they are maybe not the owner of.

![API Management Notifications][APIM_2]

Now that we have a product that requires a subscription and approval, lets setup a logic app to do a custom approval workflow.

## 2 - Logic App


[APIM_1]: https://github.com/adamhockemeyer/Azure-API-Management-Custom-Subscription-Approval/blob/master/images/APIM_1.jpg "API Management Settings"
[APIM_2]: https://github.com/adamhockemeyer/Azure-API-Management-Custom-Subscription-Approval/blob/master/images/APIM_2.jpg "API Management Notifications"
