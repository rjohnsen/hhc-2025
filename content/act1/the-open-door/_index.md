+++
date = '2025-11-08T16:30:23+01:00'
draft = false
hidden = false
title = 'The Open Door'
weight = 11
+++

![Lucas Goose](/images/act1/goose-pixel.png)

## Objective

Difficulty: 1/5

Help Goose Lucas in the hotel parking lot find the dangerously misconfigured Network Security Group rule that's allowing unrestricted internet access to sensitive ports like RDP or SSH.

## Goose Lucas mission statement

> 

## The Open Door Terminal

![Start](/images/act1/act1-opendoor-1.png)

```bash
az group list
```

![Group list](/images/act1/act1-opendoor-2.png)

```bash
az group list -o table
```

![Group list table](/images/act1/act1-opendoor-3.png)

```bash
az network nsg list -o table
```

| Location | Name                   | ResourceGroup       |
| -------- | ---------------------- | ------------------- |
| eastus   | nsg-web-eastus         | theneighborhood-rg1 |
| eastus   | nsg-db-eastus          | theneighborhood-rg1 |
| eastus   | nsg-dev-eastus         | theneighborhood-rg2 |
| eastus   | nsg-mgmt-eastus        | theneighborhood-rg2 |
| eastus   | nsg-production-eastus  | theneighborhood-rg1 |

![NSG list](/images/act1/act1-opendoor-4.png)

```bash
az network nsg show -o table --name nsg-web-eastus --resource-group theneighborhood-rg1
```

![NSG show ](/images/act1/act1-opendoor-5.png)

```bash
az network nsg rule list --name nsg-mgmt-eastus --resource-group theneighborhood-rg2 | less
```

![Rules list](/images/act1/act1-opendoor-6.png)

### Viewing suspect rules

#### Rules theneighborhood-rg2 _ nsg-mgmt-eastus

```bash
az network nsg rule list  --name nsg-mgmt-eastus --resource-group theneighborhood-rg2 | less
```

Rules for resource group "theneighborhood-rg2" and NSG name "nsg-mgmt-eastus"

* Allow-AzureBastion
* Allow-Monitoring-Inbound
* Allow-DNS-From-VNet
* Deny-All-Inbound
* Allow-Monitoring-Outbound
* Allow-AD-Identity-Outbound
* Allow-Backup-Outbound

Inspecting the each rule by the followign command: 

```bash
az network nsg rule show --nsg-name nsg-mgmt-eastus --resource-group theneighborhood-rg2 -n <rulename>
```

None of these yielded something interesting.

#### Rules theneighborhood-rg2 _ nsg-dev-eastus

```bash
az network nsg rule list  --name nsg-dev-eastus --resource-group theneighborhood-rg2 | less
```

Rules for resource group "theneighborhood-rg2" and NSG name "nsg-dev-eastus"

* Allow-HTTP-Inbound
* Allow-HTTPS-Inbound
* Allow-DevOps-Agents
* Allow-Jumpbox-Remote-Access
* Deny-All-Inbound

Inspecting the each rule by the followign command: 

```bash
az network nsg rule show --nsg-name nsg-dev-eastus --resource-group theneighborhood-rg2 -n <rulename>
```

None of these yielded something interesting.

#### Rules theneighborhood-rg1 _ nsg-web-eastus

```bash
az network nsg rule list  --name nsg-web-eastus --resource-group theneighborhood-rg1 | less
```

Rules for resource group "theneighborhood-rg1" and NSG name "nsg-web-eastus"

* Allow-HTTP-Inbound
* Allow-HTTPS-Inbound
* Allow-AppGateway-HealthProbes
* Allow-Web-To-App
* Deny-All-Inbound

Inspecting the each rule by the followign command: 

```bash
az network nsg rule show --nsg-name nsg-web-eastus --resource-group theneighborhood-rg1 -n <rulename>
```

None of these yielded something interesting.

#### Rules theneighborhood-rg1 _ nsg-db-eastus

```bash
az network nsg rule list  --name nsg-db-eastus --resource-group theneighborhood-rg1 | less
```

Rules for resource group "theneighborhood-rg1" and NSG name "nsg-db-eastus"

* Allow-App-To-DB
* Allow-AD-Trusted-Subnet
* Deny-All-Inbound

Inspecting the each rule by the followign command: 

```bash
az network nsg rule show --nsg-name nsg-db-eastus --resource-group theneighborhood-rg1 -n <rulename>
```

None of these yielded something interesting.

#### Rules theneighborhood-rg1 _ nsg-production-eastus

```bash
az network nsg rule list  --name nsg-production-eastus --resource-group theneighborhood-rg1 | less
```

Rules for resource group "theneighborhood-rg1" and NSG name "nsg-production-eastus"

* Allow-HTTP-Inbound
* Allow-HTTPS-Inbound
* Allow-AppGateway-HealthProbes
* Allow-RDP-From-Internet
* Deny-All-Inbound

Inspecting the each rule by the followign command: 

```bash
az network nsg rule show --nsg-name nsg-production-eastus --resource-group theneighborhood-rg1 -n <rulename>
```

As I were traversing and inspecting these rules, I caught one of interest. A RDP related one:

```bash
az network nsg rule show --nsg-name nsg-production-eastus --resource-group theneighborhood-rg1 -n Allow-RDP-From-Internet
```

![Interesting rule](/images/act1/act1-opendoor-7.png)

## Goose Lucas mission debrief

> Ha! 'Properly protected' they said. More like 'properly exposed to the entire internet'! Good catch, amigo.



