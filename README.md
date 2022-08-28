# Example Project

By Matheus Aguiar

Diagrams aunder ./doc

# Introduction

This is an example of a project of a centralized logging system for an environment that takes data from multiple sources.
This system uses Wazuh as the main tool and uses technologies like Kubernets (EKS) for high availability, Lambda functions, APIGW, Cloudwatch, Packer, Terraform, Syslog and Ansible. 

In this repo is only available the documentation. However, if you wish to see the full repo, please, feel free to reach me. 

# Deployment of infrastructure

## VPC + EKS Terraform

The terraform for the VPC + EKS should be deployed first. It will deploy the foundations services that all other components of the solution will connect to.

```bash

cd ./terraform/vpc+eks

```

before continuing, please fill the variables under variables.tf and the keypair name and key on main.tf

```bash

terraform init

terraform apply -auto-approve

```

## APIGW + Lambda Terraform

The terraform for the APIGW + Lambda should be deployed next. It will deploy the api gw services and stages that will call the lambda to store the incoming logs in the loggroup.

```bash

cd ./terraform/apigw

```

before continuing, please fill the variables under variables.tf

```bash

terraform init

terraform apply -auto-approve

```

After that, you should be able to see the webhook url under STAGES in the apigw service in aws.

  

# Wazuh Deployment

After the infra deployment, you should be able to deploy the Wazuh Kubernetes.

make sure that you have the appropriate aws eks configurations done for your current cluster:

```bash

aws eks --region REGION update-kubeconfig --name EKS-Wazuh-live

```

cd ./wazuh/k8s/

```bash

kubectl apply -k envs/eks

```

### Verify deployment

#### Pods

  

The following command allows us to check the pod status. Wait until all the pods are in _Running_ status.

```bash

kubectl -n wazuh get pods

```

#### Services

  

Take a look at the services to review how they are exposed.

```bash

kubectl -n wazuh get services

```

The address under 'dashboard' is the loadbalancer address that you will be able to access your wazuh dashboard.

The service under 'wazuh-master' displays an address that will be used to deploy the Wazuh Agents on the next steps.

  

The credentials are stored in the kubernetes secrets.

  

## Packer for Linux Golden Image

  

Provisions base images to be used in Toptal's infra with ssh keys, base packages, log rotations rules, security configurations and wazuh agent playbook.

  

## Installation

  

Add the HashiCorp GPG key.

  

```bash

curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -

```

Add the official HashiCorp Linux repository.

```bash

sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"

```

Update and install.

```bash

sudo apt-get update && sudo apt-get install packer

```

Verify the installation

```bash

packer --version

```

## Deployment

  

Before continuing, please fill the region, version, and address under `./ami/packer/ansible/group_vars/toptal_linux_vm/vars.yml`

and profile, region, vpc_id, subnet_id and security_group_id

`./ami/packer/aws/toptal-linux-vm.json`

  

```bash

packer build aws/toptal-linux-vm.json

```

After conclusion, the user should deploy one ec2 with the recently created ami where it will work as a log collector and will automatically connect to the Wazuh, via agent, and start sending logs.

  

# Log Collection

The rsyslog 'text file input module', provides the ability to convert any standard text file into a syslog message. This module can read a log file line by line while passing each read line to rsyslog engine rules, which then applies filter conditions and selects which actions need to be carried out. Empty lines are ****not**** processed, as they would result in empty syslog records.

  

The Log collector VM will perform the task of receiving logs, from different sources, sorting, and then sending to wazuh. This architecture will allow more security to Wazuh, not exposing the service to multiple destinations.

  

## Linux system logs

As described on the previous sections, the linux infrastructure should be deployed with the ami created where it will automatically connect to the Wazuh, via agent, and start sending logs.

  

## Windows system logs

Please fill the WAZUH_MANAGER='' and WAZUH_REGISTRATION_SERVER='' variables `./scripts/power_shell/wazuh_agent.ps1` before executing this step.

For the windows environment, the user should run the powershell script wazuh_agent.ps1.

  

It will automatically connect to the Wazuh, via agent, and start sending logs

  

## Active Directory Logs

For this step the AD service should aready be deployed. However, a script is available at ./scripts/power_shell/Invoke-PopulateAD.ps1 to populate the AD with dummy values.

  

### Forwarding logs

To Foward logs to our log collector, it was used the Solar Winds LogForwarder.

https://www.solarwinds.com/free-tools/event-log-forwarder-for-windows

  

After instalation, it will display all the different types of logs to be forward and you should choose the "Directory Services"

  

After, you should configure a syslog server where you will point to out Log Collector Linux VM

  

All the other necessary, system logs, will be delivered by wazuh agent directly to the master.

Wazuh agents do not support direct integration with AD logs.

  

## Web Server Logs

To use Rsyslog to read Apache log file and forward it to remove our log collector, you would create a configuration file like `/etc/rsyslog.d/02-apache2.conf`, with the content below:

```bash

module(load="imfile" PollingInterval="10" statefile.directory="/var/spool/rsyslog") input(type="imfile" File="/var/log/apache2/error.log" Tag="http_error" Severity="error" Facility="local6") local6.error @10.0.6.180:514

```

You should modify the above command to share the specific types of logs that you want to monitor.

  

Verify the correctness of Rsyslog configuration file;

  

`rsyslogd -N1 -f /etc/rsyslog.d/02-apache2.conf`

  

```

rsyslogd: version 8.2006.0, config validation run (level 1), master config /etc/rsyslog.d/02-apache2.conf

rsyslogd: End of config validation run. Bye.

```

If no issues, restart Rsyslog;

  

`systemctl restart rsyslog`

  

## Web Application Logs

Wazuh already has web application logs by default, since it uses Opensearch. However, the same steps should work for any type of log of any application using rsyslog.

  
# Wazuh Log Retention Policies

  

Alerts generated by Wazuh are sent to an Elasticsearch daily index named `wazuh-alerts-4.x-YYYY.MM.DD` by using the default configuration.

  

You can create policies that govern what is the lifecycle of the indices based on different phases.

  

Four phases can be defined in a `Lifecycle Policy`:

  

- `Hot phase`. For recent data that is actively accessed.

- `Warm phase`. Data that you may wish to access, but less often.

- `Cold phase`. Similar to the warm phase but you may also freeze indices to reduce overhead.

- `Delete phase`. Data that reaches this phase is deleted.

  

For more information and a step-by-step guide, please visit https://wazuh.com/blog/wazuh-index-management/
