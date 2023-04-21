# Member Prometheus setup for federation
This playbook is to setup a local Prometheus instance that will only scrape the IBP nodes which then can be scraped through Federation.

## Requirements
Setup a VM with at at least:
  - 1 core
  - 256 MB Ram
  - 32 GB Storage

## Installation
Download the Repo
  ```bash
  git clone git@github.com:ibp-network/member-prometheus.git
  ```

This Playbook uses the following two collection:
- [Prometheus](https://github.com/prometheus-community/ansible/)
- [Nginx](https://github.com/nginxinc/ansible-collection-nginx)

Install both collections
```bash
ansible-galaxy collection install -r requirements.yaml --upgrade --force
```

Copy the inventory template
```bash
cp inventory_template.yaml inventory.yaml
```

- Set the correct `ansible_user` for ssh connection
- replace `monibp-001` with your correct hostname
- set the correct `ansible_host`
- set the correct `ibp_name` name
- set any subodmain you like for `alias_prometheus`

### Nginx
The Nginx collection will install a default nginx certificate that will be not valid, if you wish to use your own certificate for it update the following section in the `nginx.yaml` file
```yaml
        nginx_config_upload_ssl_crt:
          - src: molecule/common/files/ssl/molecule.crt
            dest: /etc/ssl/certs
            backup: true
        nginx_config_upload_ssl_key:
          - src: molecule/common/files/ssl/molecule.key
            dest: /etc/ssl/private
            backup: true
```

To install and configure nginx run
```bash
ansible-playbook -i ./inventory.yaml ./nginx.yaml -D
```

This will install nginx on the prometheus host, idea is that communication between your HAproxy and Prometheus instance is also encrypted. The Valid certificate should be on the HAproxy. It's optional to have a valid certificate on the internal nginx. You can without worry use a internal CA for nginx.

### Prometheus

Copy the targets file, uncomment everything and add your nodes to either the `rpc`or `bootnode`section.
```bash
cp ./group_vars/prometheus/targets_template.yaml ./group_vars/prometheus/targets.yaml
```

Install and configure Prometheus
```bash
ansible-playbook -i ./inventory.yaml ./prometheus.yaml -D
```

### Next steps
- configure HAproxy
- create rules to only allow the federation server to access your instance
- Do a PR to append your prometheus instance to the list in `ibpfederation` job [here](https://github.com/ibp-network/prometheus-ansible/blob/main/roles/prometheus/files/prometheus.yml)
```yaml
  - job_name: ibpfederation
    metrics_path: /federate
    honor_labels: true
    scheme: https
    params:
      match[]:
      - '{job="substrate"}'
    static_configs:
    - targets:
      - ibp-monitor.member.com:9090
```