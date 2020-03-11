# Migrating from legacy service brokers
This page describes strategies for migrating from legacy service brokers to the cloud service broker.  Services created by deprecated service brokers should continue to function as-is, so migration of existing services is not required.  The cloud service broker can be installed side-by-side with legacy service brokers.  Migrating to the cloud service broker is recommended when removing dependencies on deprecated service brokers is a priority or some breaking change causes a deprecated service broker to stop functioning.

> **_WARNING:_** Some migration strategies may be destructive to resources and data.  Please read and understand the processes before implementation.  If possible, test these strategies on test services and apps before applying them to important services and apps.

## Migration Strategies
The following strategies can be used to migrate off of a legacy service broker.
* If you want to [migrate to cloud service broker](#migrating-to-cloud-service-broker), unbind &amp; deprovision the legacy service and provision &amp; bind the equivalent services using cloud service broker with the appropriate brokerpak.
	* Note that unbinding and deprovisioning a service in a legacy broker may be destructive to the underlying resources.
	* For a service which has existing data that you want to preserve, use manual steps to backup and restore into new service.
* If the equivalent services in cloud service broker are not present or different than current services broker then you may create [user provided services that are mapped to the existing resources](#migating-to-user-provided-services).

After all services have been migrated, you can `cf delete-service-broker <SERVICE_BROKER_NAME>`, if you installed using tile, you can delete tile from OpsManager.

## Migating to user-provided services 
This strategy may be used when removing dependencies on legacy brokers is a priority or when a breaking change to the platform has caused a legacy broker to stop functioning.  The goal of this strategy is to preserve the existing cloud resources, but to remove the metadata that ties this service instance to a legacy service broker.

> **_WARNING:_**  Please don't use the `cf delete-service-instance` command because that will delete underlying resources.

To accomplish this, this procedure purges the service instance metadata and creates a new user provided service with the same binding credentials.  This will allow the bound apps to connect to the cloud resources created by the legacy service broker, but while being bound to user provided service that do not have a dependency on the legacy service broker.  Following these proceedures the service instance must be managed manually.

* First, copy the binding credentials and tags from the existing legacy service instance using `cf env $APP_NAME`
	*  Copy `credentials` json of the service to file named `credentials.json`.
	*  Copy the comma separated list of `tags` from the existing service.
* Create a [user provided service](https://docs.cloudfoundry.org/devguide/services/user-provided.html) using the copied metadata that will be used to connect to the existing cloud resources.
	```bash
	cf cups $NEW_SERVICE_NAME -p credentials.json  -t "tag1","tag2"
	```
* Bind the new user-provided service to the app
	```bash
	cf bind-service $APP_NAME $NEW_SERVICE_NAME
	```  
* To remove broker metadata related to the legacy service without deleting the underlying service's resources use [purge service instance](https://cli.cloudfoundry.org/en-US/cf/purge-service-instance.html).  Note: this action is destructive to the metadata and you will not be able to reconnect the service instance with the legacy service broker.
	```bash
	cf purge-service-instance $OLD_SERVICE_NAME
	``` 
* Restage the application to apply the changes to the application
	```bash 
	cf restage $APP_NAME
	```

## Migrating to cloud service broker
This migration strategy should be used when there is an equivalent service provided by the cloud service broker brokerpak or when the application can be updated to use a new service provided by the cloud service broker. 
> **_WARNING:_** This migration strategy is destructive to the legacy service instance's resources and data.  Manual steps must be taken to backup/restore data if the new service instance needs to preserve the existing data.

```bash
cf create-service $NEW_SERVICE_TYPE $PLAN_NAME $NEW_SERVICE_NAME
cf bind-service $APP_NAME $NEW_SERVICE_NAME

# Run backup/restore scripts if needed to new service.

cf unbind-service $APP_NAME $OLD_SERVICE_NAME
cf delete-service $OLD_SERVICE_NAME
cf restage $APP_NAME
```
