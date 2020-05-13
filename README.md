
Ansible victoriametrics role
=========

Ansible role for [VictoriaMetrics](https://victoriametrics.com/)

This role can manage:

* Single-node installations
* Cluster installations
* Vmutils installations

Or even all of that at once on single node, if for some reason you need it.

## Compatibility
This role is compatible with any modern systemd-based distro.  

## Single node

Installing victoriametrics in single mode is as easy as setting
```
victoriametrics_singlenode: true
```
And running this role. It will create user, systemd config, create folder for data and start listening on 127.0.0.1:8428 for both write and read requests

Role variables related to single-node setup:

| Variable name                     | Default value                        | Description                              |
| --------------------------------- | ------------------------------------ | ---------------------------------------- |
| victoriametrics_singlenode        | `false`                              | installs single node if set to true      |
| victoriametrics_user              | `victoriametrics`                    | victoriametrics system user              |
| victoriametrics_group             | `{{ victoriametrics_cluster_user }}` | victoriametrics system group             |
| victoriametrics_version           | `1.34.3`                             | victoriametrics version                  |
| victoriametrics_retention_period  | `3`                                  | data retention period in months          |
| victoriametrics_storage_data_path | `/var/lib/victoria-metrics/`         | path to victoriametrics data dir         |
| victoriametrics_args              | `{}`                                 | optional args to pass to victoriametrics |
| victoriametrics_http_listen_addr  | `127.0.0.1:8428`                     | port to listen on                        |

## Cluster

Role variables related to cluster:
| Variable name                                       | Default value                 | Description                                        |
| --------------------------------------------------- | ----------------------------- | -------------------------------------------------- |
| victoriametrics_cluster                             | `false`                       | indicates this node is part of cluster             |
| victoriametrics_cluster_user                        | `{{ victoriametrics_user }}`  | victoriametrics system user                        |
| victoriametrics_cluster_group                       | `{{ victoriametrics_group }}` | victoriametrics system group                       |
| victoriametrics_cluster_version                     | `1.34.3`                      | victoriametrics version                            |
| victoriametrics_cluster_vmstorage                   | `false`                       | installs and configures vmstorage if set to true   |
| victoriametrics_cluster_vmstorage_retention_period  | `3`                           | vmstorage retention period (months)                |
| victoriametrics_cluster_vmstorage_storage_data_path | `/var/lib/victoria-metrics/`  | directory where vmstorage stores metrics           |
| victoriametrics_cluster_vmstorage_http_listen_addr  | `0.0.0.0:8482`                | address where vmstorage http should listen         |
| victoriametrics_cluster_vmstorage_vminsert_addr     | `0.0.0.0:8400`                | address where vmstorage should listen for vminsert |
| victoriametrics_cluster_vmstorage_vmselect_addr     | `0.0.0.0:8401`                | address where vmstorage should listen for vmselect |
| victoriametrics_cluster_vmstorage_args              | `{}`                          | additional arguments for vmstorage                 |
| victoriametrics_cluster_vminsert                    | `false`                       | installs and configures vminsert if set to true    |
| victoriametrics_cluster_vminsert_http_listen_addr   | `0.0.0.0:8480`                | address where vminsert http should listen          |
| victoriametrics_cluster_vminsert_storage_node       | `- localhost:8400`            | array of vmstorage nodes to connect                |
| victoriametrics_cluster_vminsert_args               | `{}`                          | additional arguments for vminsert                  |
| victoriametrics_cluster_vmselect                    | `false`                       | installs and configure vmselect if set to true     |
| victoriametrics_cluster_vmselect_http_listen_addr   | `0.0.0.0:8481`                | address where vmselect http should listen          |
| victoriametrics_cluster_vmselect_storage_node       | `- localhost:8401`            | array of vmstorage nodes to connect                |
| victoriametrics_cluster_vmselect_args               | `{}`                          | additional arguments for vmselect                  |

Installing cluster is a bit more complex. You will need setting at least this parameters to true:
```yaml
victoriametrics_cluster: true
victoriametrics_cluster_vmstorage: true
victoriametrics_cluster_vminsert: true
victoriametrics_cluster_vmselect: true
```

This will install vmstorage, vminsert, vmselect, create directory for vmstorage, configure vmselect and vminsert to connect to local vmstorage node.  Essentially you will get single-node "cluster".   
But if you're looking at cluster settings you're probably looking to spread them among multiple servers, so here is an example on how to do that:

Example inventory:
```ini
[victoriametrics:children]
victoriametrics-vmstorage
victoriametrics-vmselect
victoriametrics-vminsert
[victoriametrics-vmstorage]
storage_node1
storage_node2
storage_node3
[victoriametrics-vminsert]
insert_node1
insert_node2
[victoriametrics-vmselect]
select_node1
select_node2
``` 
In victoriametrics group_var:
```yaml
victoriametric_cluster: true
victoriametrics_cluster_vmselect_storage_node:
  - storage_node1:8400
  - storage_node2:8400
  - storage_node3:8400
victoriametrics_cluster_vminsert_storage_node:
  - storage_node1:8401
  - storage_node2:8401
  - storage_node3:8401
```
In victoriametrics-vmstorage group_vars:
```yaml
victoriametrics_cluster_vmstorage: true
```

In victoriametrics-vminsert group_vars:
```yaml
victoriametrics_cluster_vminsert: true
```

In victoriametrics-vmselect group_vars:
```yaml
victoriametrics_cluster_vmselect: true
```

This will end up with 2 select and 2 insert nodes, connected to 3 storage nodes. Of course this is just an example and you're free to use those vars wherever you want - in playbook vars, directly in inventory or in host_vars.

## vmutils

Setting

```yaml
victoriametrics_vmutils: true
```

Will install victoriametrics vmutils package (vmbackup, vmrestore, vmagent, vmauth, vmalert). It inherits version from victoriametrics_version.  


## vmauth
```yaml
victoriametrics_vmauth: true
victoriametrics_vmauth_users:
  - username: "local-single-node"
    password: "password"
    url_prefix: "http://localhost:8428"
  - username: "cluster-select-account-123"
    password: "password"
    url_prefix: "http://vmselect:8481/select/123/prometheus"
  - username: "cluster-insert-account-42"
    password: "password"
    url_prefix: "http://vminsert:8480/insert/42/prometheus"
```

## vmagent
```yaml
victoriametrics_vmagent: true
victoriametrics_vmagent_global:
  evaluation_interval: 15s
  scrape_interval: 15s
  scrape_timeout: 10s
victoriametrics_vmagent_external_labels:
  environment: production
victoriametrics_vmagent_remotewrite_url: http://localhost:8428/api/v1/write
victoriametrics_vmagent_remotewrite_tmpdatapath: /var/lib/vmagent/tmp
victoriametrics_vmagent_scrape_configs:
  - job_name: "telegraf-pull"
    metrics_path: /metrics
    basic_auth:
      username: username
      password: password
    static_configs:
      - targets:
        - localhost:9755
```

## vmalert 
```yaml
victoriametrics_vmalert: true
victoriametrics_vmalert_alert_rules:
  - alert: Instance down
    expr: 'up == 0'
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: '{% raw %}Exporter down{% endraw %}'
      description: '{% raw %}Prometheus node {{ $labels.instance }} is down{% endraw %}'
```