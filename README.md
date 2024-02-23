# Fortinet OnDemand Labs - Lab Provisioning and Usage Tracking

This repository contains Azure Runbooks and Lab definitions to manage Azure based Lab Environments and to track Azure and Non-Azure based Lab ***utilization***.

These lab environments are utilized for internal and external training events.

Lab environments are defined, managed, and tracked through Azure Automation. Lab ***progress*** is monitored via Google Analytics.

## Components

The process consists of several components

- **Azure Automation Account**

- **Azure Runbooks**
  - ManageTrainingUser.ps1 - Tracks lab allocation and Provisions lab (if the lab is Azure based)
  - ManageTrainingEnvs.ps1 - Deprovisions lab environments after the lab duration has expired
    - Runs Hourly: deletes/deprovisions Labs and Users from Azure when lab duration has been exceeded

- **Azure Storage Account**
  - Storage Container
    - File staging, files that may be required when a lab is provisioned
    - Lab definition files
  - Storage Table - A non-relation structured data table (AKA NoSQL) used to store lab tracking data
    - Captured fields
      - Timestamp: Date and Time lab in launched
      - Environment: AWS, Azure, GCP, etc.
      - Username (Email Address): Email address of requestor
        - Lab email (if any) is delivered to this address
        - Domain of Email may allow or restrict provisioning of lab
      - Customer: Name of Customer (if any) related to this lab utilization
      - SmartTicket: Smart Ticket Number (if any) related to this lab utilization

- **Azure Vault**
  - Stores support values, domains, subscriptions, api keys, etc.

## Lab Environment Definitions

Lab environments are defined by a JSON structured file that indicates multiple aspects of the Lab.

Labs can be one of two types

- Azure Provisioned Environments
- All Other Environments

### Azure Provisioned Environments

Azure Provisioned Environments are defined in a JSON structured file containing attributes whose values are used to manage user accounts and resources. Lab definition files are stored in the above described Azure Storage Account.

The Azure Runbook ManageTrainingUser.ps1, retrieves the requested lab definition file from the Storage Account and processes the request based on the attributes in the lab definition file.

> Azure and non-Azure lab environments will always create a Lab utilization record in the above described Azure Storage Table. For a non-Azure environment after the utilization record is created no other actions are performed.

#### Azure Lab Definition Attributes

Lab provisioning and utilization tracking is done via an Azure Runbook adn Azure Storage Table. Lab definition attributes must minimally indicate Provider Environment and Lab Name. Two attributes fortiLabEnv and fortiLabName are used to create a Lab utilization record in the Storage Table described in the Azure Components. When Azure is specified as the provider, additional attributes are required to define the resources to be provisioned for the lab.

Azure lab definition attributes describe the lab name, number of allowed users, username prefix, Azure Tenant where lab is provisioned, length of lab, and all lab required lab resources.

The lab definition shown below describes a lab environment that can be provisioned for up to 30 users. Usernames are a combination of the next available number in the Id Range and the username prefix. Usernames fortilab21 - fortilab50 would be available for provisioning. The username combined with the Tenant Domain provide a login account for Azure. The Azure login account is temporary and will be removed when teh lab duration has been reached.

A lab request from a user in an allowed domain will receive an email with their Azure login credentials. The Azure login account will only have access to the Resource Group(s) described in the lab definition. User provisioning the same lab will not have access to other user's environments, unless an Administrator providers a user access to interact with other environments.


| Attribute | Value | Description | |
|---|---|---|---|
| fortiLabEnv           | Azure                                                    | Provider Environment                                              ||
| fortiLabName          | FORTILAB                                                 | Lab Name, Must be Unique                                          ||
| fortiRestricted       | ["fortinet.com", "fortinet-us.com", "concourselabs.com"] | List Email Domains Lab is Restricted to, empty means all allowed  ||
| userIdNumberRange     | 21:50                                                    | Lab User ID Range - is combined with userNamePrefix               ||
| userNamePrefix        | fortilab                                                 | Lab User ID Prefix - with range 21:50, fortilab21 - fortilab50    ||
| userTenantDomain      | tenant-01                                                | Key mapped to deployment Tenant in Azure Vault                    ||
| labDuration           | 6                                                        | Number of lab duration days, when passed lab is deleted           ||
| userResourceGroups    | List of Resource Group definition Attributes             | List of Resource Groups and resources to provision                ||
| - suffix              | workshop-fortilab                                        | Combined with username to create Resource Group name              ||
| - location            | eastus                                                   | Azure Region                                                      ||
| - storage             | true                                                     | Create Storage Account in Resource Group                          ||
| - sharename           | cloudshellshare                                          | Name of File share to create in Storage Account                   ||
| - bastion             | true or false                                            | Create Bastion Host in Resource Group                             ||
| - utilityVnetName     | vnet-utility                                             | Azure Virtual Network Name                                        ||
| - utilityVnetCIDR     | 192.168.100.0/24                                         | Azure Virtual Network Address Space                               ||
| - utilitySubnetName   | utility                                                  | Azure Virtual Network Subnet Name                                 ||
| - utilitySubnetPrefix | 192.168.100.64/26                                        | Azure Virtual Network Subnet Address Space                        ||
| - bastionSubnetName   | AzureBastionSubnet                                       | Azure Virtual Network Bastion Subnet - must be AzureBastionSubnet ||
| - bastionSubnetPrefix | 192.168.100.0/26                                         | Azure Virtual Network Bastion Subnet Address Space                ||

```json
{
  "fortiLabEnv": "Azure",
  "fortiLabName": "FORTILAB",
  "fortiRestricted": ["fortinet.com", "fortinet-us.com", "concourselabs.com"],
  "userIdNumberRange": "21:50",
  "userNamePrefix": "fortilab",
  "userTenantDomain": "tenant-01",
  "labDuration": "6",
  "userResourceGroups": [
    {
      "suffix": "workshop-fortilab",
      "location": "eastus",
      "storage": true,
      "sharename": "cloudshellshare",
      "bastion": true,
      "utilityVnetName": "vnet-utility",
      "utilityVnetCIDR": "192.168.100.0/24",
      "utilitySubnetName": "utility",
      "utilitySubnetPrefix": "192.168.100.64/26",
      "bastionSubnetName": "AzureBastionSubnet",
      "bastionSubnetPrefix": "192.168.100.0/26"
    },
    {
      "suffix": "utility-fortilab",
      "location": "westus",
      "storage": false,
      "bastion": false
    }
  ]
}
```

### All Other Environments

These two attributes fortiLabEnv and fortiLabName are used to create a Lab utilization record in the Storage Table described in the Azure Components. However, any user and or user environment provisioning is managed by another process.

```json
{
  "fortiLabEnv": "AWS",
  "fortiLabName": "AWSFGTCNF"
}
```

## Lab Utilization Tracking

An Azure Storage Table is utilized to track lab requests. Each lab requests for any environment creates a record as described earlier in this document.

An Azure lab can be provisioned by the same user multiple times, each time a request is received, lab availability is determined. If an available user in the Id Range is found, a lab will be provisioned.  This ability allows an instructor to pre provision lab environments.
