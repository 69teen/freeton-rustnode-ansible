### This ansible script is designed for deploying a rust-based freeton node on dedicated or virtual servers.

---
## Compatibility

#### Minimum OS versions:
Ubuntu: 18.04

#### Ansible version
This has been tested on Ansible 2.9.х

## Requirements
This playbook requires root privileges or sudo.
Ansible ([What is Ansible](https://www.ansible.com/resources/videos/quick-start-video)?)

---
## Deployment: quick start
0. [Install Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) to the managed machine
###### Example: install ansible on [Ubuntu](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-ansible-on-ubuntu)
`sudo apt update`\
`sudo apt install ansible sshpass git`

1. Download or clone this repository

`git clone https://github.com/itgoldio/freeton-rustnode-ansible.git`

2. Go to the playbook directory

`cd freeton-rustnode-ansible/`

3. Edit the inventory file

###### Specify the IP addresses in freeton_node and monitoring_server section and connection settings (`ansible_user`, `ansible_ssh_pass` ...) for your environment. If you run  the playbook locally, don't change the default settings.
###### Example:

`vim inventory`

`[freeton_node]`\
`11.22.33.44 ansible_user='root' ansible_ssh_pass='secretpassword'`\
`22.33.44.55 ansible_user='user' ansible_ssh_pass='supersecretpassword' ansible_become=true`\
`[monitoring_server]`\
`33.44.55.66`

###### There is no need to specify more than one monitoring server. All agents will connect only to the first one.

4. Edit the variable files vars/[freeton_node.yml](./vars/freeton_node.yml), vars/[system.yml](./vars/system.yml) and vars/[monitoring.yml](./vars/monitoring.yml)

`vim vars/freeton_node.yml`\
`vim vars/system.yml`\
`vim vars/monitoring.yml`

### Be careful and don't forget to change standard passwords in [monitoring.yml](./vars/monitoring.yml) file.

5. Run playbook:

`ansible-playbook deploy_freeton_node.yml`

##### If you run playbook locally, use -c local parameter

`ansible-playbook deploy_freeton_node.yml -c local`

##### To skip monitoring server installation role, specify '-t basic'

`ansible-playbook deploy_freeton_node.yml -c local -t basic`

---
## Variables
See the vars/[freeton_node.yml](./vars/freeton_node.yml), vars/[system.yml](./vars/system.yml) and vars/[monitoring.yml](./vars/monitoring.yml) files for more details.

---
## Usage
By default, after deploying the freeton node it gets started automatically (change this behavior in vars/[freeton_node.yml](./vars/freeton_node.yml)).
You can control the running status of the freeton node using systemd commands:
`systemctl status freeton`\
`systemctl stop freeton` \
`systemctl start freeton`

Freeton node stores logs in `/opt/freeton/logs` directory (change this by specifying freeton_node_log_dir var in vars/[freeton_node.yml](./vars/freeton_node.yml)).
You can change log rules in file roles/freeton_node_deploy/freeton_node_deploy/[log_cfg.yml.j2](./roles/freeton_node_deploy/templates/log_cfg.yml.j2)

Binary files located in `/opt/freeton/bin`, all other locations stored in vars/[freeton_node.yml](./vars/freeton_node.yml)


Monitoring services will start at the host specified in monitoring_server inventory section.
You can access Grafana the monitoring observability solution with pre-installed dashboards via a web-browser:

`http://<monitoring_server_ip>:3000/`

Also you can access Chronograf - InfluxDB data observation tool - via a web-browser:

`http://<monitoring_server_ip>:8888/`

---
## Extras
When you change global network config, for example, you running rustnet.ton.dev network, and want to change it to fld.ton.dev. You should change freeton_node_global_config_URL variable in vars/[freeton_node.yml](./vars/freeton_node.yml) and run playbook again with tag 'flush' in addition with tag 'basic':

`ansible-playbook deploy_freeton_node.yml -c local -t basic,flush`

##### Do not use flush tag without any other tags
---
## Monitoring
You can see our grafana dashboard by address `http://<monitoring_server_ip>:3000/`

![dashboadr](docs/grafana-1.png?raw=true "grafana-1")

- show current timediff and timediff graph
- Current Election state (Started/Stopped)
- Current Election date start (date or undefine if elector don’t started)
- Current Election date stop (date or undefine if elector don’t started)
- Info, that node already in participant list
- info, that node already win election and wait new validation cycle
- count of unsign transaction between wallet to depool
- Wallet balance
- depool balance
- proxy-1 balance
- proxy-2 balance
- many system graphs, we change it after testing node in rustcup

You can see our chronograf dashboard by address `http://<monitoring_server_ip>:8888/`

![dashboadr](docs/chronograf-1.png?raw=true "chronograf-1")

You can see or change credentials in vars/[monitoring.yml](./vars/monitoring.yml)

---
## Alerting
You can use grafana alerts for alerting

1. Create notification channel

Go to `http://<monitoring_server_ip>:3000/alerting/notifications` and create notification channel.
You can use telegram, slack, email, webhook and others
2. Create alert 
- Go to dashboard
- Edit dashboard panel
- select tab Alert

more info here: https://grafana.com/docs/grafana/latest/alerting/create-alerts/

---
## Automate staking through depool
You need to put your keys and data to `/home/freeton/ton-keys`:
- $HOSTNAME.addr file with wallet address
- depool.addr file with depool address
- msig.keys.json file with private key for wallet
- *[Optional]* msig2.keys.json file with private key for second sign
- tik.addr wallet addr for send ticktock request to depool. You can put $HOSTNAME.addr data to here
- tik.keys.json file with private key for tik.addr

TIP: you can change default location for keys in vars/[freeton_node.yml](./vars/freeton_node.yml) 

Add several scripts to crontab use
`crontab -e -u freeton`

### Automate ticktock depool
add to cron\
`*/3 * * * * /bin/bash && export PATH=$PATH:/opt/freeton/scripts &&  cd /opt/freeton/scripts && ton-depool-ticktok.sh >> /opt/freeton/logs/crontab-ton-depool-ticktok.log`\
it will ticktock dpool once by election cycle

### Automate send validation request
add to cron\
`*/10 * * * * /bin/bash && export PATH=$PATH:/opt/freeton/scripts &&  cd /opt/freeton/scripts && ton-depool-validation-request.sh >> /opt/freeton/logs/crontab-ton-depool-validation-request.log`\
it will ticktock dpool once by election cycle

### *[Optional]* Sing transaction use secondary key
If you use wallet with RegConfirm = 2, you can sing transaction from wallet to depool use secondary key
add to cron\
`*/10 * * * * /bin/bash && export PATH=$PATH:/opt/freeton/scripts &&  cd /opt/freeton/scripts && ton-wallet-transaction-confirm.sh >> /opt/freeton/logs/crontab-ton-wallet-transaction-confirm.log`

---
## Scripts
All scripts will be added to PATH for freeton user. Monitoring use several scripts.
### [ton-env.sh](./roles/monitoring_agent/files/../templates/ton-env.j2)
Script main script. It store all shared variables that used in other scripts. Please, fill it first.
### [ton-check-env.sh](./roles/monitoring_agent/files/scripts/ton-check-env.sh)
Script system script. It check that ton-env.sh fill correct. Don't use it directly.
### [ton-depool-balance.sh](./roles/monitoring_agent/files/scripts/ton-depool-balance.sh)
Script show depool balance in nanotokens.
### [ton-depool-proxy-1-balance.sh](./roles/monitoring_agent/files/scripts/ton-depool-proxy-1-balance.sh)
Script show depool proxy 1 balance in nanotokens.
### [ton-depool-proxy-2-balance.sh](./roles/monitoring_agent/files/scripts/ton-depool-proxy-2-balance.sh)
Script show depool proxy 2 balance in nanotokens.
### [ton-depool-ticktok.sh](./roles/monitoring_agent/files/scripts/ton-depool-ticktok.sh)
Script send ticktock command from wallet to depool.
### [ton-depool-validation-request.sh](./roles/monitoring_agent/files/scripts/ton-depool-validation-request.sh)
Validator script for elections through depool.
### [ton-election-date.sh](./roles/monitoring_agent/files/scripts/ton-election-date.sh)
Script return date election in unix time or return 0 if election isn't active or return -1 is something wrong
### [ton-election-date-end.sh](./roles/monitoring_agent/files/scripts/ton-election-date-end.sh)
Script return date end election in unix time or return -1 if election isn't active.
### [ton-election-date-start.sh](./roles/monitoring_agent/files/scripts/ton-election-date-start.sh)
Script return date started election in unix time or return -1 if election isn't active.
### [ton-election-state.sh](./roles/monitoring_agent/files/scripts/ton-election-state.sh)
Script return "ACTIVE" if elecetion is active or "STOPPED" otherwise
### [ton-node-diff.sh](./roles/monitoring_agent/files/scripts/ton-node-diff.sh)
Script return time diff in unix time for local node or "-1" if console or node doesn't work
### [ton-node-participant-state.sh](./roles/monitoring_agent/files/scripts/ton-node-participant-state.sh)
Script return "ACTIVE" if node already in participants list. It return "NOT_FOUND" if election is active, but node absent in participants list. Script return "UNKNOWN" if election stopped.
### [ton-wallet-balance.sh](./roles/monitoring_agent/files/scripts/ton-wallet-balance.sh)
Script show wallet balance in nanotokens.
### [ton-wallet-transaction-confirm.sh](./roles/monitoring_agent/files/scripts/ton-wallet-transaction-confirm.sh)
Script find unsigned transaction from wallet to depool and signet it use secondary key. It use ton-env.sh vars by default. You can send wallet addr and depool addr directly as arguments.
### [ton-wallet-transaction-count.sh](./roles/monitoring_agent/files/scripts/ton-wallet-transaction-count.sh)
Script show count unsigned transaction from wallet to depool.
### [ton-node-validate-current.sh](./roles/monitoring_agent/files/scripts/ton-node-validate-current.sh)
Script return info about current validation state.
- Unknown - something wrong (node not working and etc)
- True - node can validate at this time
- False - node can't validate at this time
### [ton-node-validate-next.sh](./roles/monitoring_agent/files/scripts/ton-node-validate-next.sh)
Script return info about next validation round.
- Unknown - something wrong (node not working and etc)
- True - node can validate in next round
- False - node can't validate in next round

---

## Support
We can help you in telegram chats
- RU: https://t.me/itgoldio_support_ru
- EN: https://t.me/itgoldio_support_en

## Changelog
Here: [changelog.md](./changelog.md)
