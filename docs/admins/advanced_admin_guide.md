# Advanced Admin Guide

This guide outlines some advanced features for deploying and maintaining the
Open Front End:

- Multi-project configuration
- Creation and management of service accounts
- Least-privilege roles for users
- Least-privilege enabled APIs for projects

## Multi-project Configuration

By default, the Open Front End will deploy using the specified project then it
creates a service account which, when registered in the portal, allows the
project to create HPC clusters and the associated resources. Usually, a
single project for deploying then maintaining GCP resources is sufficient.

It is possible though to separate concerns, so that one project deploys the
Open Front End and another project provisions GCP resources via the Front End,
allowing finer user access management. Separation of projects is a fairly
straightforward matter of registering different service account credentials, 
for the different projects, to the Open Front End. The process to create
additional service accounts is outlined in the following section.

## Service Account Management

[Service accounts](https://cloud.google.com/iam/docs/service-accounts) are
used by the Front End to provision GCP resources on behalf of projects.
These accounts are registered to the Front End using a generated json
credential file. A default service account and credential is (optionally)
created by the deployment script, however a more complex setup may be required
for a multi-project configuration, or when a service account with custom roles
is required.

Service accounts can be created in a number of ways, as outlined below. In each
case, the generated json credential is registered within the Open Front End
in the same way, which is outlined in the [Admin Guide](../admin_guide/).

### Creating a Service Account via the Helper Script

The helper script included in the HPC Toolkit repository can be used to create
a service account with the required basic roles/permissions when used by a user
that has privileges within the project (e.g. *Owner*, or *Editor*). The
roles/permissions could then be modified via `gcloud` or GCP Console (both
covered below).

To create a service account and credential file in json format:

```bash
script/service_account.sh create <PROJECT_ID> <SERVICE_ACCOUNT_NAME>
script/service_account.sh credential <PROJECT_ID> <ACCOUNT_NAME> <PATH_TO_KEY_FILE>
```

The script also has options to *list*, *check* and *delete* - see the
built-in help for instructions:

```bash
script/service_account.sh help
```

**Note to administrators/developers:** if the roles required for a
service account changes, the script must be modified (and docs including the
list of roles below, updated).

### Creating a Service Account via the GCP Console

A user with project privileges can also create service accounts via the GCP
Console:

1. Log in to the [GCP console](https://console.cloud.google.com/) and select
   the GCP project that hosts the Open Front End.
1. From the Navigation menu, select *IAM & Admin*, then *Service Accounts*.
   - Click the *CREATE SERVICE ACCOUNT* button.
   - Name the service account, optionally provide a description, and then
     click the *CREATE* button.
1. Grant the service account the following roles (these are the same basic
   roles the helper script would apply - for finer control see later section):
   - Cloud Filestore Editor
   - Compute Admin
   - Create Service Accounts
   - Delete Service Accounts
   - Project IAM Admin
   - Notebooks Admin
   - Vertex AI administrator
1. Click *Done* button.
1. Locate the new service account from the list, click *Manage Keys* from the
   *Actions* menu.
   - Click *ADD KEY*, then *Create new key*.
     - Select JSON as key type, and click the *CREATE* button.
     - A JSON key file will then be downloaded.
     - Copy the generated JSON content which should then be pasted into the
       credential creation form within the Open Front End.
1. Click *Validate and Save* to register the new credential to the Front End.

### Creating a Service Account using `gcloud`

Alternatively the `gcloud` command line tool can be used to create a suitable
service account (this is what the helper script does with additional checks).
To create a service account with the basic required roles:

```bash
gcloud iam service-accounts create <SERVICE_ACCOUNT_NAME>
for roleid in file.editor \
              compute.admin \
              iam.serviceAccountCreator \
              iam.serviceAccountDelete \
              resourcemanager.projectIamAdmin \
              notebooks.admin aiplatform.admin; do \
     gcloud projects add-iam-policy-binding <PROJECT_ID> \
      --member="serviceAccount:<SERVICE_ACCOUNT_NAME>@<PROJECT_ID>.iam.gserviceaccount.com" \
      --role="roles/$roleid"; \
done

gcloud iam service-accounts keys create <PATH_TO_KEY_FILE> \
    --iam-account=<SERVICE_ACCOUNT_NAME>@<PROJECT_ID>.iam.gserviceaccount.com
```

Once complete, the service account key json text should be copied from
`PATH_TO_KEY_FILE` into the credentials form on the Open Front End.

The credential can now be used to create network, storage and compute resources
from the Front End.

The roles can be changed to give finer control as outlined in the next section.

### Custom Roles/Permissions and APIs

The projects and user account used for deploying the Open Front End can be more
tightly controlled with respect to the enabled APIs and roles/permissions.

#### User Account

Rather than using *Owner* role, or the high-level roles stated in the Admin
Guide, the user account deploying the Front End can use a custom set of
least-privilege roles. The complete list of required permissions is as follows:
<!-- TODO: this list needs regularly checking and maintaining -->

```Text
 compute.acceleratorTypes.list
 compute.addresses.use
 compute.disks.create
 compute.disks.get
 compute.firewalls.create
 compute.firewalls.delete
 compute.firewalls.get
 compute.globalOperations.get
 compute.instances.create
 compute.instances.delete
 compute.instances.get
 compute.instances.getSerialPortOutput
 compute.instances.setLabels
 compute.instances.setMetadata
 compute.instances.setServiceAccount
 compute.instances.setTags
 compute.machineTypes.list
 compute.networks.create
 compute.networks.delete
 compute.networks.get
 compute.networks.updatePolicy
 compute.projects.get
 compute.regionOperations.get
 compute.routers.create
 compute.routers.delete
 compute.routers.get
 compute.routers.update
 compute.subnetworks.create
 compute.subnetworks.delete
 compute.subnetworks.get
 compute.subnetworks.use
 compute.subnetworks.useExternalIp
 compute.zoneOperations.get
 compute.zones.get
 compute.zones.list
 file.instances.create
 file.instances.delete
 file.instances.get
 file.operations.get
 iam.serviceAccounts.actAs
 iam.serviceAccounts.create
 iam.serviceAccounts.delete
 iam.serviceAccounts.get
 iam.serviceAccounts.getIamPolicy
 iam.serviceAccounts.setIamPolicy
 pubsub.subscriptions.create
 pubsub.subscriptions.delete
 pubsub.subscriptions.get
 pubsub.subscriptions.getIamPolicy
 pubsub.subscriptions.setIamPolicy
 pubsub.topics.attachSubscription
 pubsub.topics.create
 pubsub.topics.delete
 pubsub.topics.get
 pubsub.topics.getIamPolicy
 pubsub.topics.setIamPolicy
 resourcemanager.projects.get
 resourcemanager.projects.getIamPolicy
 resourcemanager.projects.setIamPolicy
 storage.buckets.create
 storage.buckets.delete
 storage.buckets.get
 storage.buckets.getIamPolicy
 storage.buckets.setIamPolicy
 storage.objects.create
 storage.objects.delete
 storage.objects.get
 storage.objects.list
```

<!--TODO: For a TKFE Service Account (the one registered with a credential to administer resources via the portal) is... -->

#### Project APIs

In a multi-project configuration, the enabled project APIs can also be
reduced to a subset of those APIs only needed for the functions required.

If a project is only deploying the Open Front End, and not provisioning GCP
resources via the Open Front End, only the following APIs need to be enabled:

```Text
 Compute Engine API
 Cloud Monitoring API
 Cloud Logging API
 Cloud Pub/Sub API
 Cloud Resource Manager
 Identity and Access Management (IAM) API
```

A project that administers GCP resources via the Open Front End (that has a
service account created within it, as covered above), needs the following APIs:

```Text
 Compute Engine API
 Cloud Monitoring API
 Cloud Resource Manager
 Cloud Logging API
 Cloud OS Login API
 Cloud Filestore API
 Cloud Billing API
 Vertex AI API
```
