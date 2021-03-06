# Cloudify Manager of Managers

The blueprint in this repo allows managing Cloudify Manager instances
(Tier 1 managers) using a master Cloudify Manager (Tier 2 manager).

## Using the blueprint

In order to use the blueprint the following prerequisites are
necessary:

1. A working 4.3 (RC or GA) Cloudify Manager (this will be the Tier 2
manager). You need to be connected to this manager (`cfy profiles use`).
1. An SSH key linked to a cloud keypair (this will be used to SSH
into the Tier 1 VMs).
1. A clone of this repo.

Optional:
1. A `pip` virtualenv with `wagon` installed in it. This is only
necessary if you're planning on building the CMoM plugin yourself
(more on the plugin below).

### Uploading plugins

Two plugins are required to use the blueprint - an IaaS plugin
(currently only OpenStack is supported) and the Cloudify Manager of
Managers (or CMoM for short) plugin (which is a part of this repo).

First, upload the IaaS plugin to the manager. e.g. run:

```
cfy plugins upload <WAGON_URL> -y <YAML_URL>
```

[link_to_wagon](https://github.com/cloudify-cosmo/cloudify-openstack-plugin/releases/download/2.5.2/cloudify_openstack_plugin-2.5.2-py27-none-linux_x86_64-centos-Core.wgn)
[link to yaml](https://github.com/cloudify-cosmo/cloudify-openstack-plugin/releases/download/2.5.2/plugin.yaml)

Second, you'll need to create a Wagon from the CMoM plugin. Run (this
is assuming you're inside the `manager-of-managers` folder):
```
wagon create -f plugins/cmom -o <CMOM_WAGON_OUTPUT>
```

Now upload this plugin as well:

```
cfy plugins upload <CMOM_WAGON_OUTPUT> -y plugins/cmom/plugin.yaml
```


### Tier 2 manager files

a few files need to be present on the Tier 2 manager. All of those files
need to be accessible by Cloudify's user - `cfyuser`. Because of this,
it is advised to place them in `/etc/cloudify`, and make sure they are
`chown`ed by `cfyuser` (e.f. `chown cfyuser: /etc/cloudify/filename`).
The files are:

1. The private SSH key connected to the cloud keypair. Its input is
is `ssh_private_key_path`.

1. The install RPM (its world-accessible URL will be provided
separately). Its input is `install_rpm_path`.

1. The CA certificate and key. Those will be used by the Tier 2 manager
to connect to the Tier 1 managers, as well as for generating the Tier 1
managers' external certificates. The inputs are `ca_cert` and `ca_key`.

In summary, this inputs section should look like this:

```
inputs:
  ca_cert: /etc/cloudify/ca_certificate.pem
  ca_key: /etc/cloudify/ca_key.pem
  ssh_private_key_path: /etc/cloudify/ssh_key
  install_rpm_path: : /etc/cloudify/cloudify-manager-install.rpm
```


### Installing the blueprint

Now all that is left is to edit the inputs file (you can copy the
[`sample_inputs`](sample_inputs.yaml) file and edit it - see the
inputs section below for a full explanation), and run:

```
cfy install blueprint.yaml -b <BLUEPRINT_NAME> -d <DEPLOYMENT_ID> -i <INPUTS_FILE>
```

### Getting the outputs

To get the outputs of the installation (currently the IPs of the master
and slaves Cloudify Managers) run:

```
cfy deployments outputs <DEPLOYMENT_ID>
```


## Blueprint inputs

Below is a list with explanations for all the inputs necessary
for the blueprint to function. Much of this is mirrored in the
[`sample_inputs`](sample_inputs.yaml) file.

### OpenStack inputs

Currently only Openstack is supported as the platform for this
blueprint. Implementations for other IaaSes will follow.

#### Common OS inputs

* `os_image` - OpenStack image name or ID to use for the new server
* `os_flavor` - OpenStack flavor name or ID to use for the new server
* `os_network` - OpenStack network name or ID the new server will be connected to
* `os_keypair` - OpenStack key pair name or ID of the key to associate with the new server
* `os_security_group` - The name or ID of the OpenStack security group the new server will connect to
* `os_server_group_policy` - The policy to use for the server group
* `os_username` - Username to authenticate to OpenStack with
* `os_password` - OpenStack password
* `os_tenant` - Name of OpenStack tenant to operate on
* `os_auth_url` - Authentication URL for KeyStone

#### Defining the manager's public IP

There are currently 3 supported ways to assign the manager's IP.
To toggle between the different modes you'll need to leave only one of
the lines in [`infra`](include/openstack/infra.yaml) uncommented -
[`private_ip.yaml`](include/openstack/private_ip.yaml),
[`floating_ip.yaml`](include/openstack/floating_ip.yaml) or
[`private_fixed_ip.yaml`](include/openstack/private_fixed_ip.yaml)
needs to be imported.

> **Important**: only *one* of the files mentioned above needs to be
> imported, otherwise the blueprint will not work.

The 3 modes are:
1. Using the FloatingIP mechanism. This requires providing a special
input:
* `os_floating_network` - The name or ID of the OpenStack network to use
for allocating floating IPs

2. Using only an internal network, without a floating IP. This requires
creating a new port, which is assumed to be connected to an existing
subnet; thus a special input is needed:
* `os_subnet` - OpenStack name or ID of the subnet that's
connected to the network that is to be used by the manager

3. Using a known in advance resource pool of IPs and hostnames. Like
in the previous section, this requires creating a new port. This
method also creates a "resource pool" object, that holds a list of
resources and allocates them as the need arises. The inputs for this
mode are:
* `os_subnet` - Like in the above mode
* `resource_pool` - A list of resources from which the IP addresses and
the hostnames should be chosen. The format should be as follows:
```
resource_pool:
    - ip_address: <IP_ADDRESS_1>
      hostname: <HOSTNAME_1>
    - ip_address: <IP_ADDRESS_2>
      hostname: <HOSTNAME_2>
```

#### KeyStone v3 inputs

The following inputs are only relevant in KeyStone v3 environments:
* `os_region` - OpenStack region to use
* `os_project` - Name of OpenStack project (tenant) to operate on
* `os_project_domain` - The name of the OpenStack project domain to use
* `os_user_domain` - The name of the OpenStack user domain to use

#### Block storage devices

When working with block storage devices (e.g. Cinder volumes) there
is a special input that needs to be provided:
* `os_device_mapping` - this is a list of volumes as defined by the API
[here](https://docs.openstack.org/nova/pike/user/block-device-mapping.html).
An example input would look like this:
```
os_device_mapping:
  - boot_index: "0"
    uuid: "41a1f177-1fb0-4708-a5f1-64f4c88dfec5"
    volume_size: 30
    source_type: image
    destination_type: volume
    delete_on_termination: true
```
Where `uuid` is the UUID of the OS image that should be used when
creating the volume.

> Note: When using the `os_device_mapping` input, the `os_image` input
> should be left empty.

Other potential inputs (for example, with subnet names, CIDRs etc.)
might be added later.

### General inputs

These are general inputs necessary for the blueprint:

* `install_rpm_path` - as specified above
* `ca_cert` - as specified above
* `ca_key` - as specified above
* `manager_admin_password` - as specified above
* `manager_admin_username` - the admin username for the Tier 1 managers
(default: admin)
* `num_of_instances` - the number of Tier 1 instances to be created
(default: 2). This affects the size of the HA cluster.
* `ssh_user` - User name used when SSH-ing into the Tier 1 manager VMs
* `ssh_private_key_path` - as described above.
* `additional_config` - An arbitrary dictionary which should mirror the
structure of [config.yaml](https://github.com/cloudify-cosmo/cloudify-manager-install/blob/master/config.yaml)
It will be merged (while taking precedence) with the config as described
in the cloudify.nodes.CloudifyTier1Manager type in the plugin.yaml file.
Whenever possible the inputs in the blueprint.yaml file should be used.
For example:

```
inputs:
  additional_config:
    sanity:
      skip_sanity: true
    restservice:
      log:
        level: DEBUG
    mgmtworker:
      log_level: DEBUG
```

### LDAP inputs

Inside [`ldap_inputs.yaml`](include/ldap_inputs.yaml) is a defined
datatype for LDAP inputs. It is useful to utilize it for convenience.

All the following inputs need to reside under `ldap_config` in
the inputs file:

* `server` - The LDAP server address to authenticate against
* `username` - LDAP admin username. This user needs to be able to make requests
          against the LDAP server
* `password` - LDAP admin password
* `domain` - The LDAP domain to be used by the server
* `dn_extra` - Extra LDAP DN options. (separated by the `;` sign. e.g. a=1;b=2).
Useful, for example, when it is necessary to provide an organization ID.
* `is_active_directory` - Specify whether the LDAP server used for authentication is an
          Active Directory server.

 The actual input should look like this:

 ```
 inputs:
   ldap_config:
     server: SERVER
     username: USERNAME
     password: PASSWORD
     ...
 ```


### Additional inputs


> **NOTE**: When mentioning local paths in the context of the inputs
below, the paths in question are paths on the *Tier 2* manager.
So if it is desirable to upload files from a local location, these
files need to be present on the Tier 2 manager in advance.
Otherwise, URLs may be used freely.

It is possible to create/upload certain types of resources on the
Tier 1 cluster after installation. Those are:

1. `tenants` - a list of tenants to create after the cluster is
installed. The format is:
```
inputs:
  tenants:
    - <TENANT_NAME_1>
    - <TENANT_NAME_2>
```

2. `plugins` - a list of plugins to upload after the cluster is
installed. The format is:
```
inputs:
  plugins:
    - wagon: <WAGON_1>
      yaml: <YAML_1>
      tenant: <TENANT_1>
    - wagon: <WAGON_2>
      yaml: <YAML_2>
      visibility: <VIS_2>
```

Where:
* `WAGON` is either a URL of a Cloudify Plugin (e.g.
[openstack.wgn](http://repository.cloudifysource.org/cloudify/wagons/cloudify-openstack-plugin/2.0.1/cloudify_openstack_plugin-2.0.1-py27-none-linux_x86_64-centos-Core.wgn)),
  or a local (i.e. on the Tier 2 manager) path to such wagon (required)
* `YAML` is the plugin's plugin.yaml file - again, either URL or local
  path (required)
* `TENANT` is the tenant to which the plugin will be uploaded (the tenant
  needs to already exist on the manager - use the above `tenants` input
  to create any tenants in advance). (Optional - default is
  default_tenant)
* `VISIBILITY` defines who can see the plugin - must be one of
  \[private, tenant, global\] (Optional - default is tenant).
Both WAGON and YAML are *required* fields

3. `secrets` - a list of secrets to create after the cluster is
installed. The format is:

```
inputs:
  secrets:
    - key: <KEY_1>
      string: <STRING_1>
      file: <FILE_1>
      visibility: <VISIBILITY_1>
```

Where:
* `KEY` is the name of the secret which will then be used by other
  blueprints by the intrinsic `get_secret` function (required)
* `STRING` is the string value of the secret \[mutually exclusive with FILE\]
* `FILE` is a local path to a file which contents should be used as the
  secrets value \[mutually exclusive with VALUE\]
* `VISIBILITY` defines who can see the secret - must be one of
  \[private, tenant, global\] (Optional - default is tenant).
`KEY` is a *required* field, as well as one (and only one) of STRING or FILE.

4. `blueprints` - a list of blueprints to upload after the cluster is
installed. The format is:

```
inputs:
  blueprints:
    - path: <PATH_1>
      id: <ID_1>
      filename: <FILENAME_1>
      tenant: <TENANT_1>
      visibility: <VISIBILITY_1>
```

Where:
* `PATH` can be either a local blueprint yaml file, a blueprint
  archive or a url to a blueprint archive (required)
* `ID` is the unique identifier for the blueprint (if not specified, the
  name of the blueprint folder/archive will be used
* `FILENAME` is the name of an archive's main blueprint file. Only
  relevant when uploading an archive
* `TENANT` is the tenant to which the blueprint will be uploaded (the tenant
  needs to already exist on the manager - use the above `tenants` input
  to create any tenants in advance). (Optional - default is
  default_tenant)
* `VISIBILITY` defines who can see the secret - must be one of
  \[private, tenant, global\] (Optional - default is tenant).

5. `scripts` - a list of scripts to run after the manager's installation.
All these scripts need to be available on the Tier 2 manager and
accessible by `cfyuser`. These scripts will be executed *after* the
manager is installed but *before* the cluster is created. The format is:

```
scripts:
  - <PATH_TO_SCRIPT_1>
  - <PATH_TO_SCRIPT_2>
```

6. `files` - a list of files to copy to the Tier 1 managers from the
Tier 2 amnager after the Tier 1 managers' installation.
All these files need to be available on the Tier 2 manager and
accessible by `cfyuser`. These files will be copied *after* the
manager is installed but *before* the cluster is created. The format is:

```
files:
  - src: <TIER_2_PATH_1>
    dst: <TIER_1_PATH_1>
  - src: <TIER_2_PATH_2>
    dst: <TIER_1_PATH_2>
```

### Upgrade inputs

The following inputs are only relevant when upgrading a previous
deployment. Use them only when installing a new deployment to which
you wish to transfer data/agents from an old deployment.

* `restore` - Should the newly installed Cloudify Manager be restored
from a previous installation. Must be used in conjunction with some of
the other inputs below. See [`plugin.yaml`](plugins/cmom/plugin.yaml)
for more details (default: false)
* `snapshot_path` - A local (relative to the Tier 2 manager) path to a snapshot that should be
used. Mutually exclusive with `old_deployment_id` and `snapshot_id`
(default: '')
* `old_deployment_id` - The ID of the previous deployment which was used to control the Tier 1
managers. If the `backup` workflow was used with default values there will
be a special folder with all the snapshots from the Tier 1 managers. If the
`backup` input is set to `false` `snapshot_id` must be provided as well
(default: '')
* `snapshot_id` - The ID of the snapshot to use. This is only relevant if `old_deployment_id`
is provided as well (default: '')
* `transfer_agents` - If set to `true`, an `install_new_agents` command will be executed after
the restore is complete (default: true)
* `restore_params` - An optional list of parameters to pass to the underlying
`cfy snapshots restore` command. Accepted values are: 
[`--without-deployment-envs`, `--force`, `--restore-certificates`, 
`--no-reboot`]. These need to be passed as-is with both dashes. (default: [])

### Backup workflow

There is a `backup` workflow that can be used at any time. It creates a 
snapshot on the Tier 1 cluster, and downloads it to the Tier 2 manager.
The snapshots are all saved in `/etc/cloudify/DEPLOYMENT_ID/snapshots`.

Run the workflow like this:
```
cfy executions start backup -p inputs.yaml
```

Where the optional inputs file may contain the following: 

* `snapshot_id` - The ID of the snapshot to create (will default to the 
current time/date) 
* `backup_params` - An optional list of parameters to pass to the underlying
`cfy snapshots create` command. Accepted values are: [`--include-metrics`,
`--exclude-credentials`, `--exclude-logs`, `--exclude-events`]. These need to be 
passed as-is with both dashes. (default: [])


## Healing

The blueprint implements an auto-healing mechanism for the Tier 1
managers. Due to the fact that Tier 1 managers are configured in a
cluster, and software issues are to be handled by
[HA failovers](http://docs.getcloudify.org/4.3.0/manager/high-availability-clusters/),
healing is only performed on nodes that loose connection to the Tier 2
manager (e.g. shut-off or terminated VMs, network issues, etc).

A simple monitoring policy is defined that sends metrics (using the
[Diamond plugin](http://docs.getcloudify.org/4.3.0/plugins/diamond/))
to the Tier 2 manager.

> Metric heartbeats are sent every 10 seconds by default. See
[here](http://docs.getcloudify.org/4.3.0/plugins/diamond/#global-config)
to learn how to alter the `interval` if necessary.

The above metrics are parsed by a simple [host-failure
policy](http://docs.getcloudify.org/4.3.0/manager_policies/built-in-policies/#host-failure),
which is triggered if the metrics stop being delivered. This policy
triggers a custom healing workflow, which build on the default
[heal workflow](http://docs.getcloudify.org/4.3.0/workflows/built-in-workflows/#the-heal-workflow)
and adds functionality to it. The additional functionality has to do
with the fact that we need to first remove the faulty node from the
Tier 1 HA cluster, and then, after it was reinstalled, rejoin the same
cluster.

The flow of the healing workflow is as follows:

1. Try to find at least one Tier 1 manager that is still alive. If none
is found, **the workflow will fail and retry**. This is intended -
the scenario we're trying to avoid is a case where connection to
the whole cluster is down. We don't want to perform heal in this case,
because the cluster is stateful, and healing is not intended for disaster
recovery.

> In case the whole cluster is down, the workflow will retry the
default amount of times (60) with the default interval (15 seconds),
unless configured otherwise in the blueprint (see
[here](http://docs.getcloudify.org/4.3.0/workflows/error-handling/#task-retries)
to learn how to alter `task_retries` and `retry_interval`).
The message displayed in this scenario will be: `Could not find a
profile with a cluster configured. This might mean that the whole
network is unreachable.`

2. Once found, remove the faulty node's IP from the cluster. This step
is necessary in order for it to rejoin the cluster later on.

3. Do a backup - this is just a precaution. If something will go wrong
with the healing, you can create a new cluster that will be restored
from a snapshot created during this step.

> The snapshots are saved in a folder created specifically for the
deployment being backed-up under `/etc/cloudify`. Unless specified,
the default is to use the current date and time as the snapshot name.
So, a snapshot path will look like this:
`/etc/cloudify/<DEPLOYMENT_NAME>/snapshots/2018-03-21-09:09:05.zip`

4. Uninstall the faulty node. This means removing the whole VM.

5. Reinstall the faulty node. This means re-creating the VM, and
reinstalling Cloudify Manager on it.

> Note that the Manager will be recreated with the same IP and other
configurations.

6. Re-join the Tier 1 HA cluster.


Because we're expecting HA failovers in cases of faulty nodes, the
interval between subsequent heal workflow runs was increased to 600
seconds (from the default 300), in order to accommodate the selection
of a new cluster leader. The value can be configured in the main
blueprint YAML file.

### Post-heal actions

After a successful heal any users working with the Tier 1 cluster via
the CLI should run `cfy cluster update-profile` in order to update
their local profiles.

Users that are working with the Tier 1 cluster via the GUI will have
to use a different IP when connecting to the manager if the healed
node was the cluster leader. If the healed node was a replica, no
further actions are required.