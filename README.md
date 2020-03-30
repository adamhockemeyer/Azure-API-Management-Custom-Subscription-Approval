# Azure API Management Custom Subscription Approval
An example of how to create a custom workflow for Azure API Management product subscriptions approvals.

## Introduction

[Azure API Management](https://azure.microsoft.com/en-us/services/api-management/ "Azure API Management") offers a lot of really great features for organizations to organize and access various systems by surfacing API's through a common platform both internally and externally.  Azure API Management (APIM for short) allows API publishers the ability to expose just an API, or a group of API's known as a product.  For example, a food truck service may want to expose an 'Order' product, but that 'product' may be made up of API's responsible for creating user accounts as well as actually placing an order.

As your company continues to adopt APIM, you may have considered if you should have multiple APIM instances or if you can use a shared APIM instance that all of the organizations API's and products go into.  A feature of APIM products' is that you can require users of your API to register for a subscription to use your API, and you also also set whether that subscription should be automatically approved or if an admin or some other user should be the approver.  The tricky issue here is that if your products are managed by different teams, then each team may require a unique group or user/users that should be the approvers for a specific product.

This guide will walk through how to create a Logic App that will allow for a custom subscription approval flow per product in Azure API Management.
