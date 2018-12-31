---
layout: post
title:  "How to Use Ansible Gcp compute inventory plugin"
tags: [gcp, ansible]
excerpt: <p>Apply Ansible commands on your gcp compute hosts by using the gcp_compute inventory plugin.</p>
---

If you use Ansible to manage your infrastructure configuration, you have to maintain an inventory of all the target hosts and host groups you want to manage.
Ansible inventories are often written in `yaml` or `ini` files.

But sometimes, your infrastructure is already managed by an external system (CMDB, cloud platform, ...), so you'll have to to pull your ansible inventory from these external sources.

Ansible let's you manage that with [dynamic inventory][dynamic-inventory]{:target="_blank"}

## Work with Ansible dynamic inventories

A dynamic inventory is a python script that should accept `--list` and `--host <hostname>` arguments. It will compute your inventory by outputing a json dictionnary hots groups, hosts and variables.


Here is an example of an empty expected json response :

```json
{
    "_meta": {
        "hostvars": {}
    },
    "all": {
        "children": [
            "ungrouped"
        ]
    },
    "ungrouped": {}
}
```

You can call it this way :

```shell
# Ping all hosts from the dynamic inventory
ansible -i <my_dynamic_inventory>.py all -m ping
```

As dynamic inventory is a python script, you have full flexibility on how to organize your inventory.
It is very handy when you need to call `apis` to retrieve hosts data from external sources.

### Inventory plugins

Dynamic inventory can be a bit tedious to write from scratch. So since **ansible 2.4**, there is an existing plugin type to standardize the way to implement dynamic inventories : [inventory plugins][inventory-plugins]{:target="_blank"} 

There is a lot of already existing inventory plugins to interface with most common vendors : `gcp`, `aws`, `azure`, `ovh`, `scaleway`..., you can list them all using the following command :

```shell
# List existing inventory plugins 
ansible-doc -t inventory -l
```

Reading existing lugins code is a great way to demistify how you shoud write a custom one if you need.
If you really need to develop a custom one, you can follow this [documentation][develop-inventory-plugin]{:target="_blank"}


## Example with GCP inventory plugin

For the purpose of this article, let's say you have a gcp project with id `myproject` with 3 compute vms in it (2 web servers and 1 db server):

| Vm instance name | Region       | Zone           |
| ---------------- |:------------:| --------------:|
| euw4-gcp-web-1   | europe-west4 | europe-west4-a |
| euw4-gcp-web-2   | europe-west4 | europe-west4-b |
| euw4-gcp-db      | europe-west4 | europe-west4-a |
{: .table .table-striped .table-dark}

We will use the existing [`gcp_compute` inventory plugin](https://github.com/ansible/ansible/blob/devel/lib/ansible/plugins/inventory/gcp_compute.py){:target="_blank"} to manage your gcp compute vms with ansible.

This plugin is embedded in your ansible install, so you are ready to use it.

### Prerequisistes

Ansible `gcp_compute` plugin use gcp apis to authenticate, and rely on the following python libraries for that : `requests` and `google-auth`.
If you have installed pip, you can install these libraries with these commands :

```shell
pip install requests
pip install google-auth
```

Ansible `gcp_compute` propose several authentication methods, but on this article we will use the `serviceaccount` method.
So you will need to create a serviceaccount in your project with permissions to read compute at minima.
Then [generate and download the json service account key](https://cloud.google.com/iam/docs/creating-managing-service-account-keys), we will use it for the rest of this article. 

### Setup

Ansible `gcp_compute` plugin can be configured with a yaml file. Here is a minimal yaml configuration file example:

```yaml
# inventory.compute.gcp.yml
plugin: gcp_compute             # name the plugin you want to use (use `ansible-doc -t inventory -l` to list available plugins)
projects:
  - <gcp_project_id>            # Id of your gcp project
regions:                        # regions from your project you want to fetch inventory from (you can also use zones instead of regions if you target one or several specific zones)        
  - <gcp_project_region>
filters: []
auth_kind: serviceaccount       # gcp authentication kind. with service account you should provide the service account json key file to authenticate
service_account_file: <service_account_file>.json   # Service account json keyfile
```

### Basic usage

You can test the plugin to see the computed inventory with the `ansible-inventory` command, and you should see the following output :

```
➜  ansible-inventory -i inventory.compute.gcp.yml --graph
@all:
  |--@ungrouped:
  |  |--<ip_euw4-gcp-web-1>
  |  |--<ip_euw4-gcp-web-2>
  |  |--<ip_euw4-gcp-db>
```

If you want to see compute vm names instead of ip, you can add `hostnames` attribute in you `yaml` inventory :

```yaml
# inventory.compute.gcp.yml
plugin: gcp_compute
projects:
  - <gcp_project_id>
regions:
  - <gcp_project_region>
hostnames:                # A list of options that describe the ordering for which hostnames should be assigned. Currently supported hostnames are 'public_ip', 'private_ip', or 'name'.
  - name
filters: []
auth_kind: serviceaccount
service_account_file: <service_account_file>.json
```

Now, you'll see the following ouput : 
```
➜  ansible-inventory -i inventory.compute.gcp.yml --graph
@all:
  |--@ungrouped:
  |  |--euw4-gcp-web-1
  |  |--euw4-gcp-web-2
  |  |--euw4-gcp-db
```

If you want full details about your inventory  :

```
➜  ansible-inventory -i inventory.compute.gcp.yml --list
```

It will output your inventory hosts, groups and associated vars as `json`.

You are now ready to use this inventory (here we launch a ping command on all hosts) :

```shell
# Launch ping command on all hosts matched by the inventory
ansible -i inventory.compute.gcp.yml all -m ping
```

### Group your hosts

We see in previous examples that retrieve hosts are ungrouped. 
Let's say I want to group them by zones. You can specify the `keyed_groups` to do so.
`keyed_groups` can be seen as `facets` : it will create host groups dynamically based on the content of your hosts.

```yaml
# inventory.compute.gcp.yml
---
plugin: gcp_compute
projects:
  - <gcp_project_id>
regions:
  - <gcp_project_region>
keyed_groups:
  - key: zone
hostnames:
  - name
filters: []
auth_kind: serviceaccount
service_account_file: <service_account_file>.json
```
If you test now your inventory, you will see that your hosts will be grouped by zones :

```
➜  ansible-inventory -i inventory.compute.gcp.yml --graph
@all:
  |--@_europe_west4_a:
  |  |--euw4-gcp-web-1
  |  |--euw4-gcp-db
  |--@_europe_west4_b:
  |  |--euw4-gcp-web-2
  |--@ungrouped:
  |  |--all:
  |  |--vars:
```

Ok, I am now able to execute different actions for specific zones.

If I want to execute specific actions for `web` servers, I need to group my web servers in a host group named `web-servers`.
I can use the `groups` attribute in my inventory for that. Here I will group the hosts based on their names (if a host is named with the pattern -web- then it will belong to the `web-servers` group) :

```yaml
# inventory.compute.gcp.yml
---
plugin: gcp_compute
projects:
  - <gcp_project_id>
regions:
  - <gcp_project_region>
keyed_groups:
  - key: zone
groups:
  web-servers: "'-web-' in name"
hostnames:
  - name
filters: []
auth_kind: serviceaccount
service_account_file: <service_account_file>.json
```

The graph ouput : 
```
➜  ansible-inventory -i inventory.compute.gcp.yml --graph
@all:
  |--@_europe_west4_a:
  |  |--euw4-gcp-web-1
  |  |--euw4-gcp-db
  |--@_europe_west4_b:
  |  |--euw4-gcp-web-2
  |--@web-servers:
  |  |--euw4-gcp-web-1
  |  |--euw4-gcp-web-2
  |--@ungrouped:
  |  |--all:
  |  |--vars:
```

You can group also group hosts based on host labels instead of vm name if you prefer.

### Combine dynamic inventory with static inventory

If I have to apply variables to my groups and hosts, I can do that in a static inventory and combine this static inventory with my gcp dynamic inventory.

For example, let's say we specify `ansible_become` and `ansible_ssh_user` variables to apply to all hosts.

You can create this static inventory

```yaml
# inventory.yml
all:
  vars:
    ansible_become: true
    ansible_ssh_user: <your_custom_user>

```

Create a directory `inventory` and place your 2 inventories in it :

```shell
➜  tree inventory/
inventory/
├── inventory.gcp.yml
└── inventory.yml
```

Then execute ansible by using `inventory` directory :

```shell
# Launch ping command on all hosts matched by the inventory
ansible -i inventory all -m ping
```

Ansible will combine all inventories combined in this directory.

## Conclusion

With Ansible inventory plugins you can interface with external systems to retrieve your inventories. It is good for maintenance as you maintains only one repository for your inventory data.

A lot of inventory plugins are provided ootb by Ansible but you can build your own if you need to.

If you use gcp, you can use `gcp_compute` inventory plugin to retrieve all your compute vms data.

You can combine multiple inventories (static and dynamic ones), for example if you need to combine custom variables not directly managed in gcp.

[dynamic-inventory]: https://docs.ansible.com/ansible/latest/user_guide/intro_dynamic_inventory.html#dynamic-inventory
[inventory-plugins]: https://docs.ansible.com/ansible/latest/plugins/inventory.html#inventory-plugins
[develop-inventory-plugin]: https://docs.ansible.com/ansible/latest/dev_guide/developing_inventory.html#developing-an-inventory-plugin