# Migrate Spring Config Server applications from TAS to Tanzu Application Platform

Tanzu Application Service (TAS) and Tanzu Application Platform (commonly known as TAP) both enable
you to load Java properties into an application from a configuration repository.

You can migrate an application running on TAS with Spring Configuration Service (SCS) to Tanzu
Application Platform with Application Configuration Service (ACS).

This migration does not require changes to the application code or the configuration repository
where properties are stored. However, the mechanism for loading properties into a Tanzu Application
Platform workload is quite different and requires a substantially different setup.

At a high level SCS and ACS operate as follows:

- SCS instances run as web applications serving configuration by using REST requests. TAS applications
  bound to SCS instances fetch configuration using an autowired `ConfigServerConfigDataLoader` bean
  and add it to the application properties.

- ACS `ConfigurationServer` instances are Kubernetes custom resources, reconciled by a controller
  manager. These resources are wired into Tanzu Application Platform workloads using a
  `ConfigurationSlice` object. Each `ConfigurationSlice` creates a bindable secret object containing
  properties, which is mounted to the application file system and loaded into the Spring application
  from there.

This topic uses the [cook](https://github.com/spring-cloud-services-samples/cook)
sample application to demonstrate how to migrate.

## <a id="deploy-to-tas"></a> Deploy the `cook` application to TAS

At a high level, the steps to deploy on TAS are:

1. Create an SCS instance that points to the
   [cook-config GitHub repository](https://github.com/spring-cloud-services-samples/cook-config) by
   running:

   ```console
   cf create-service -c '{ "git": { "uri": "https://github.com/spring-cloud-services-samples/cook-config", \
   "label": "main"  } }' ${service_name} standard cook-config-server
   ```

1. Run `cf push` to push the application to TAS with an application manifest that binds to the
   service instance:

    ```yaml
    ---
    applications:
      - name: cook
        host: cookie
        instances: 1
        memory: 1G
        services:
          - cook-config-server
        env:
          SPRING_PROFILES_ACTIVE: development
          JBP_CONFIG_OPEN_JDK_JRE: '{ jre: { version: 17.+ }}'
    ```

For more detailed instructions, see the
[README](https://github.com/spring-cloud-services-samples/cook/blob/main/README.adoc) file.

## <a id="migrate-tas-to-tap"></a> Migrate from TAS to Tanzu Application Platform

This section describes the areas to translate when creating a Tanzu Application Platform workload
that will deploy the same application.

### <a id="create-cnfg-srvr-instance"></a> Create a Config Server instance

TAS/SCS
: Create a service instance by running `cf create-service`, specifying the URI and label (branch) of
  the `cook-config` repository in the configuration JSON:

   ```console
   cf create-service -c '{ "git": { "uri": "https://github.com/spring-cloud-services-samples/cook-config", \
   "label": "main"  } }' p.config-server standard cook-config-server
   ```

Tanzu Application Platform/ACS
: Create a `ConfigurationSource` resource, specifying the location of the `cook-config` repository in
  the `spec.backends` property. For detailed options, see the
  [ACS documentation](https://{{ vars.staging_toggle }}.vmware.com/en/Application-Configuration-Service-for-VMware-Tanzu/2.3/acs/GUID-gettingstarted-configuringworkloads.html#create-a-configuration-source).

### <a id="spec-app-name-profiles"></a> Specify the application name and Spring profiles

TAS/SCS
: To specify the application name and Spring profiles:

   1. Specify the application name is specified in the `spring.application.name` property. For the
      `cook` application this is found in [application.yml](https://github.com/spring-cloud-services-samples/cook/blob/main/src/main/resources/application.yml).

   1. Set the `SPRING_PROFILES_ACTIVE` environment variable in the `env` key in the application
      manifest to designate profiles. Multiple entries must be comma-separated.

Tanzu Application Platform/ACS
: To specify the application name and Spring profiles:

   1. Create a `ConfigurationSlice` resource, specifying the name of the `ConfigurationSource`
      resource in the `spec.configurationSource` property.

   1. Create one or more entries in the `spec.content` array property of the `ConfigurationSlice`
      resource with the format `APP-NAME/PROFILE-NAME/OPTIONAL-LABEL` where:

      - `APP-NAME` is the application name.
      - `PROFILE-NAME` is the profile name.
      - `OPTIONAL-LABEL` is an optional comma-separated list of labels from which to retrieve properties.

  For more information, see the
  [ACS documentation](https://{{ vars.staging_toggle }}.vmware.com/en/Application-Configuration-Service-for-VMware-Tanzu/2.3/acs/GUID-crds-configurationslice.html#configurationslice-content).

### <a id="bind-cnfg-srvr-to-app"></a> Bind the Config Server to the application

TAS/SCS
: To bind the Config Server to the application, either add the service-instance name in the services
  key in the application manifest, or run `cf bind-service` to bind the service to the application.

Tanzu Application Platform/ACS
: To bind the Config Server to the application:

   1. Create a `ResourceClaim` resource, specifying the details of the `ConfigurationSlice` resource
      in the `spec.ref` section of the configuration.
   1. Specify the `ResourceClaim` resource in the `spec.serviceClaims` section of the workload. This
      mounts the properties to the running pod’s file system.
   1. Specify a `SPRING_CONFIG_IMPORT` property in the `spec.env` section of the workload with the
      value `optional:configtree:${SERVICE_BINDING_ROOT}/CLAIM-NAME/`, where `CLAIM-NAME` is the name
      you specified in the `spec.serviceClaims` list.

      This environment variable tells the Spring runtime to pull properties from the file system at
      the path where the `serviceClaims` mechanism mounted them. For more information, see
      [ACS documentation](https://{{ vars.staging_toggle }}.vmware.com/en/Application-Configuration-Service-for-VMware-Tanzu/2.3/acs/GUID-gettingstarted-configuringworkloads.html#using-configuration-in-a-workload).

## <a id="deploy-cook-to-tap"></a> Deploy the `cook` application to Tanzu Application Platform

Apply the following resources to Tanzu Application Platform in order to run the `cook` sample:

```yaml
apiVersion: "config.apps.tanzu.vmware.com/v1alpha4"
kind: ConfigurationSource
metadata:
  name: cook-config-source
spec:
  backends:
    - type: git
      uri: https://github.com/spring-cloud-services-samples/cook-config
---
apiVersion: "config.apps.tanzu.vmware.com/v1alpha4"
kind: ConfigurationSlice
metadata:
  name: cook-config-slice
spec:
  configurationSource: cook-config-source
  content:
  - cook/production/tap
  - cook/default
---
apiVersion: services.apps.tanzu.vmware.com/v1alpha1
kind: ResourceClaim
metadata:
  name: cook-config-claim
spec:
  ref:
    apiVersion: config.apps.tanzu.vmware.com/v1alpha4
    kind: ConfigurationSlice
    name: cook-config-slice
---
apiVersion: carto.run/v1alpha1
kind: Workload
metadata:
  name: cook
  labels:
    app.kubernetes.io/part-of: cook-app
    apps.tanzu.vmware.com/has-tests: "true"
    apps.tanzu.vmware.com/workload-type: web
spec:
  serviceClaims:
  - name: spring-properties
    ref:
      apiVersion: services.apps.tanzu.vmware.com/v1alpha1
      kind: ResourceClaim
      name: cook-config
  build:
    env:
      - name: BP_JVM_VERSION
        value: "17"
  env:
    - name: SPRING_CONFIG_IMPORT
      value: "optional:configtree:${SERVICE_BINDING_ROOT}/spring-properties/"
    - name: SPRING_CLOUD_CONFIG_ENABLED
      value: "false"
    - name: SPRING_PROFILES_ACTIVE
      value: "development"
  source:
    git:
      url: https://github.com/spring-cloud-services-samples/cook.git
      ref:
        branch: tap
```

Configuration samples for deploying the `cook` application on Tanzu Application Platform are in
[GitHub](https://github.com/spring-cloud-services-samples/cook/tree/tap/tap).