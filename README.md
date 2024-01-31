# Unified Catalog Migration

A tool for migrating your Unified Catalog items and assets to the Amplify Enterprise Marketplace.

The script will:

1. Log into the platform using the `axway auth login` either using the browser or using a service account (if set in the configuration file)
2. Read all existing catalog items from a specific environment
3. For each catalog item
    * Look for the linked API Service
    * Create a new asset (or use an existing one) that will contain the API Service
    * Create a new product (if not existing yet) and link it to the asset created
    * (Optional) Publish the product to the selected Marketplace
    * (Optional if product is published) Create a corresponding Marketplace subscription and application in the Marketplace for each active catalog item subscription

## Prerequisites

* [Axway CLI](https://docs.axway.com/bundle/amplify-central/page/docs/integrate_with_central/cli_central/index.html)
* [jq](https://jqlang.github.io/jq/)
* [curl](https://curl.se/)

The script can be run on Microsoft Windows bash shell or Linux.

## Configuration

An environment file is available in the config directory to set some properties:

* _CLIENT_ID_ to use a service account instead of a real user
* _CLIENT_SECRET_ to set the service account private key.
* _CENTRAL_ENVIRONMENT_ to set the environment where to source the Catalog Items
* _PLAN_TITLE_ to set the default plan title
* _PLAN_QUOTA_ to set the default plan quota
* _PLAN_APPROVAL_MODE_ - automatic (default) or manual
* _PUBLISH_TO_MARKETPLACES_ to know if products need to be published to Marketplace
* _MARKETPLACE_TITLE_ to give the Marketplace title where the product needs to be published
* _ASSET_NAME_FOLLOW_SERVICE_VERSION_ to help name the Asset based on the APIService name (see Note below)

Note regarding _ASSET_NAME_FOLLOW_SERVICE_VERSION_:
 When the same service (i.e. the same name) exists in multiple environments, the script will create 1 Asset per major service release and only one Product.
 
 For that, you need to set _ASSET_NAME_FOLLOW_SERVICE_VERSION=Y_

Sample:

* Env1: APIService1 - v1.0.0
* Env2: APIService1 - v1.0.1
* Env3: APIService1 - v2.0.0

After Migration: using _ASSET_NAME_FOLLOW_SERVICE_VERSION=Y_

* Asset APIService1 V1 linked to APIService1 - v1.0.0 and APIService1 - v1.0.1
* Asset APIService1 V2 linked to APIService1 - v2.0.0
* Product APIService1 linked to Asset APIService1 V1 and Asset APIService1 V1

After Migration: using _ASSET_NAME_FOLLOW_SERVICE_VERSION=N_
* Asset APIService1 linked to APIService1 - v1.0.0 and APIService1 - v1.0.1 and APIService1 - v2.0.0
* Product APIService1 linked to Asset APIService1

## Mapping Unified Catalog => Marketplace

The following table shows how the properties from the Unified Catalog object are used to create the Marketplace objects:

| Initial Objects                      | Asset                | Product       | Marketplace subscription | Marketplace application |
|------------------------------------|------------------------|---------------|--------------------------|-------------------------|
| **Consumer instance**                |                      |               |                          |                         |
|  Id                                  |                      |               |                          |                         |
|  Name                                |                      |               |                          |                         |
|  Title                               | Title                | Title         |                          |                         |
|  Description                         | Description          | Description   |                          |                         |
|  API Service                         | APIService           |               |                          |                         |
|                                      |                      |               |                          |                         |
| **APIService**                       |                      |               |                          |                         |
|  Icon (not used)                     |                      |               |                          |                         |
|                                      |                      |               |                          |                         |
| **APIServiceInstance**               |                      |               |                          |                         |
|  CredentialRequestDefinition         |                      |               |                          |                         |
|  AccessRequestDefinition             |                      |               |                          |                         |
|                                      |                      |               |                          |                         |
| **CatalogItem (= consumerInstance)** |                      |               |                          |                         |
|  Image/base64                        | Icon                 | Icon          |                          |                         |
|  Category(ies)                       |                      | Category(ies) |                          |                         |
|                                      |                      |               |                          |                         |
| **CatalogItemDocumentation**         |                      |               |                          |                         |
|  Value                               |                      | Article       |                          |                         |
|                                      |                      |               |                          |                         |
| **Subscription**                     |                      |               |                          |                         |
|  Name                                |                      |               | Name                     |                         |
|  Application name (if available)     |                      |               |                          | Name                    |
|  OwningTeam                          |                      |               | OwningTeam               | OwningTeam              |

### Subscription handling

There are some core differences between a Unified Catalog subscription and Marketplace subscription.

For the Unified Catalog, there is only one subscription allowed per application for a catalog item. In the Marketplace, this will translate to one subscription per product plan, allowing only one application to be registred for a single API resource inside the product, to guarantee that the plan quota can be enforced correctly on the dataplane.

![Alt text](subscription.png)

The migration script will create only 1 subscription that can be used with each resource in the product. This subscription is then used to add access to the various applications.

**Limitation**:
On the Marketplace side, it is not possible to access the same product resource using a subscription plan having a quota restriction and multiple applications. This is to help providers correctly enforce the subscription plan quotas.

The migration script displays this message `/!\ Cannot add access to {UNIFED_CATAOLOGSUBSCRIPTION_APPLICATION_NAME} using subscription {MARKETPLACE_SUBSCRIPTION_NAME}: access already exist for another application` if multiple applications try to access the same resource using the same subscription.

You can overcome this by using the unlimited quota variable: `PLAN_QUOTA="unlimited"`

## Usage

Migrating all catalog items that belong to a specific environment:

```bash
./migrateUnifiedCatalog.sh
```

Migrating a single catalog item that belong to a specific environment:

```bash
./migrateUnifiedCatalog.sh "catalogItemTitle"
```

## Known limitations

* Tags from the Unified Catalog are not added to the product
* No product visibility is set based on the catalog item sharing
* A product can be published in only one Marketplace
