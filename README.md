# Deploying BIG-IP VE in Azure - Standalone Two NICs

## Contents

- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Important Configuration Notes](#important-configuration-notes)
- [BYOL Licensing](#byol-licensing)
- [Installation Example](#installation-example)
- [Configuration Example](#configuration-example)

## Introduction

This solution uses a Terraform template to launch a two NIC deployment of a cloud-focused BIG-IP VE standalone device in Microsoft Azure. Traffic flows to the BIG-IP VE which then processes the traffic to application servers. This is the standard cloud design where the BIG-IP VE instance is running with a dual interface. Management traffic is processed on NIC 1, and data plane traffic is processed NIC 2.

The BIG-IP VEs have the [Local Traffic Manager (LTM)](https://f5.com/products/big-ip/local-traffic-manager-ltm) module enabled to provide advanced traffic management functionality. In addition, the [Application Security Module (ASM)](https://www.f5.com/pdf/products/big-ip-application-security-manager-overview.pdf) can be enabled to provide F5's L4/L7 security features for web application firewall (WAF) and bot protection.

Terraform is beneficial as it allows composing resources a bit differently to account for dependencies into Immutable/Mutable elements. For example, mutable includes items you would typically frequently change/mutate, such as traditional configs on the BIG-IP. Once the template is deployed, there are certain resources (network infrastructure) that are fixed while others (BIG-IP VMs and configurations) can be changed.

Example...

-> Run once
- Deploy the entire infrastructure with all the neccessary resources, then use Declarative Onboarding (DO) to configure the BIG-IP cluster, Application Services (AS3) to create a sample app proxy, then lastly use Service Discovery to automatically add the DVWA container app to the BIG-IP pool.

-> Run many X
- [Redeploy BIG-IP for Replacement or Upgrade](#Redeploy-BIG-IP-for-replacement-or-upgrade)
- [Reconfigure BIG-IP L1-L3 Configurations (DO)](#Rerun-Declarative-Onboarding-on-the-BIG-IP-VE)
- [Reconfigure BIG-IP L4-L7 Configurations (AS3)](#Rerun-Application-Services-AS3-on-the-BIG-IP-VE)
- [Reconfigure BIG-IP Telemetry Streaming (TS)](#Rerun-Telemetry-Streaming-on-the-BIG-IP-VE)

**Networking Stack Type:** This solution deploys into a new networking stack, which is created along with the solution.

## Version
This template is tested and worked in the following version
Terraform v0.12.26
+ provider.azurerm v2.1
+ provider.local v1.4
+ provider.null v2.1
+ provider.template v2.1

## Prerequisites

- **Important**: When you configure the admin password for the BIG-IP VE in the template, you cannot use the character **#**.  Additionally, there are a number of other special characters that you should avoid using for F5 product user accounts.  See [K2873](https://support.f5.com/csp/article/K2873) for details.
- This template requires a service principal for backend pool service discovery. **Important**: you MUST have "OWNER" priviledge on the SP in order to assign role to the resources in your subscription. See the [Service Principal Setup section](#service-principal-authentication) for details, including required permissions.
- The HA BIG-IP VMs use Azure RBAC role for the failover instead of using Service Prinicipal.
- This deployment will be using the Terraform Azurerm provider to build out all the neccessary Azure objects. Therefore, Azure CLI is required. For installation, please follow this [Microsoft link](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-apt?view=azure-cli-latest)
- If this is the first time to deploy the F5 image, the subscription used in this deployment needs to be enabled to programatically deploy. For more information, please refer to [Configure Programatic Deployment](https://azure.microsoft.com/en-us/blog/working-with-marketplace-images-on-azure-resource-manager/)

## Important Configuration Notes

- Variables are configured in variables.tf
- Sensitive variables like Azure Subscription and Service Principal are configured in terraform.tfvars
  - Note: Passwords and secrets will be moved to Azure Key Vault in the future
- This template uses Declarative Onboarding (DO) and Application Services 3 (AS3) packages for the initial configuration. As part of the onboarding script, it will download the RPMs automatically. See the [AS3 documentation](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/) and [DO documentation](https://clouddocs.f5.com/products/extensions/f5-declarative-onboarding/latest/) for details on how to use AS3 and Declarative Onboarding on your BIG-IP VE(s). The [Telemetry Streaming](https://clouddocs.f5.com/products/extensions/f5-telemetry-streaming/latest/) extension is also downloaded and can be configured to point to Azure Log Analytics. 
- Files
  - appserver.tf - resources for backend web server running DVWA
  - bigip.tf - resources for BIG-IP, NICs, public IPs, network security group
  - main.tf - resources for provider, versions, resource group
  - network.tf - resources for VNET and subnets
  - onboard.tpl - onboarding script which is run by commandToExecute (user data). It will be copied to /var/lib/waagent/CustomData upon bootup. This script is responsible for downloading the neccessary F5 Automation Toolchain RPM files, installing them, and then executing the onboarding REST calls.
  - do.json - contains the L1-L3 BIG-IP configurations used by DO for items like VLANs, IPs, and routes.
  - as3.json - contains the L4-L7 BIG-IP configurations used by AS3 for items like pool members, virtual server listeners, security policies, and more.
  - ts.json - contains the BIG-IP configurations used by TS for items like telemetry streaming, CPU, memory, application statistics, and more.

## BYOL Licensing
This template uses PayGo BIG-IP image for the deployment (as default). If you would like to use BYOL licenses, then these following steps are needed:
1. Find available images/versions with "byol" in SKU name using Azure CLI:
  ```
          az vm image list -f BIG-IP --all

          # example output...

          {
            "offer": "f5-big-ip-byol",
            "publisher": "f5-networks",
            "sku": "f5-big-ltm-2slot-byol",
            "urn": "f5-networks:f5-big-ip-byol:f5-big-ltm-2slot-byol:15.1.002000",
            "version": "15.1.002000"
          },
  ```
2. In the "variables.tf", modify *image_name* and *product* with the SKU and offer from AZ CLI results
  ```
          # BIGIP Image
          variable product { default = "f5-big-ip-byol" }
          variable image_name { default = "f5-big-ltm-2slot-byol" }
  ```
3. In the "variables.tf", modify *license1* with a valid regkey
  ```
          # BIGIP Setup
          variable license1 { default = "" }
  ```
4. In the "do.json", add the "myLicense" block under the "Common" declaration ([full declaration example here](https://clouddocs.f5.com/products/extensions/f5-declarative-onboarding/latest/bigip-examples.html#standalone-declaration))
  ```
        "myLicense": {
            "class": "License",
            "licenseType": "regKey",
            "regKey": "${regKey}"
        },
  ```

## Template Parameters

| Parameter | Required | Description |
| --- | --- | --- |
| prefix | Yes | This value is inserted at the beginning of each Azure object (alpha-numeric, no special character) |
| rest_do_uri | Yes | URI of the Declarative Onboarding REST call |
| rest_as3_uri | Yes | URI of the AS3 REST call |
| rest_do_method | Yes | Available options are GET, POST, and DELETE |
| rest_AS3_method | Yes | Available options are GET, POST, and DELETE |
| rest_vm01_do_file | Yes | Terraform will generate the vm01 DO json file, where you can manually run it again for debugging |
| rest_vm_as3_file | Yes | Terraform will generate the AS3 json file, where you can manually run it again for debugging |
| sp_subscription_id | Yes | This is the service principal subscription ID |
| sp_client_id | Yes | This is the service principal application/client ID |
| sp_client_secret | Yes | This is the service principal secret |
| sp_tenant_id | Yes | This is the service principal tenant ID |
| uname | Yes | User name for the Virtual Machine |
| upassword | Yes | Password for the Virtual Machine |
| location | Yes | Location of the deployment |
| cidr | Yes | IP Address range of the Virtual Network |
| subnet1 | Yes | Subnet IP range of the management network |
| subnet2 | Yes | Subnet IP range of the external network |
| f5vm01mgmt | Yes | IP address for 1st BIG-IP's management interface |
| f5vm01ext | Yes | IP address for 1st BIG-IP's external interface |
| f5privatevip | Yes | Secondary Private IP address for BIG-IP virtual server (internal) |
| f5publicvip | Yes | Secondary Private IP address for BIG-IP virtual server (external) |
| instance_type | Yes | Azure instance to be used for the BIG-IP VE |
| product | Yes | Azure BIG-IP VE Offer |
| bigip_version | Yes | BIG-IP Version |
| image_name | Yes | F5 SKU (image) to deploy. Note: The disk size of the VM will be determined based on the option you select.  **Important**: If intending to provision multiple modules, ensure the appropriate value is selected, such as ****AllTwoBootLocations or AllOneBootLocation****. |
| license1 | No | The license token for the F5 BIG-IP VE (BYOL) |
| host1_name | Yes | Hostname for the 1st BIG-IP |
| ntp_server | Yes | Leave the default NTP server the BIG-IP uses, or replace the default NTP server with the one you want to use |
| timezone | Yes | If you would like to change the time zone the BIG-IP uses, enter the time zone you want to use. This is based on the tz database found in /usr/share/zoneinfo (see the full list [here](https://github.com/F5Networks/f5-azure-arm-templates/blob/master/azure-timezone-list.md)). Example values: UTC, US/Pacific, US/Eastern, Europe/London or Asia/Singapore. |
| dns_server | Yes | Leave the default DNS server the BIG-IP uses, or replace the default DNS server with the one you want to use | 
| DO_URL | Yes | This is the raw github URL for downloading the Declarative Onboarding RPM |
| AS3_URL | Yes | This is the raw github URL for downloading the AS3 RPM |
| TS_URL | Yes | This is the raw github URL for downloading the Telemetry RPM |
| libs_dir | Yes | This is where all the temporary libs and RPM will be store in BIG-IP |
| onboard_log | Yes | This is where the onboarding script logs all the events |

## Installation Example

To run this Terraform template, perform the following steps:
  1. Clone the repo to your favorite location
  2. Modify terraform.tfvars with the required information
  ```
      # BIG-IP Environment
      uname     = "azureuser"
      upassword = "Default12345!"

      # Azure Environment
      sp_subscription_id = "xxxxx"
      sp_client_id       = "xxxxx"
      sp_client_secret   = "xxxxx"
      sp_tenant_id       = "xxxxx"
      location           = "westus2"

      # Prefix for objects being created
      prefix = "mylab123"
  ```
  3. Initialize the directory
  ```
      terraform init
  ```
  4. Test the plan and validate errors
  ```
      terraform plan
  ```
  5. Finally, apply and deploy
  ```
      terraform apply
  ```
  6. When done with everything, don't forget to clean up!
  ```
      terraform destroy
  ```

## Configuration Example

The following is an example configuration diagram for this solution deployment. In this scenario, all access to the BIG-IP VE device is direct to the BIG-IP via the management interface. The IP addresses in this example may be different in your implementation.

![Configuration Example](./images/AzureStandalone2nic.png)

## Documentation

For more information on F5 solutions for Azure, including manual configuration procedures for some deployment scenarios, see the Azure section of [F5 CloudDocs](https://clouddocs.f5.com/cloud/public/v1/azure_index.html). Also check out the [Azure BIG-IP Lightboard Lessons](https://devcentral.f5.com/s/articles/Lightboard-Lessons-BIG-IP-Deployments-in-Azure-Cloud) on DevCentral. This particular standalone example is based on the [BIG-IP Standalone F5 ARM Cloud Template on GitHub](https://github.com/F5Networks/f5-azure-arm-templates/tree/master/supported/standalone/2nic/new-stack/payg).

## Creating Virtual Servers on the BIG-IP VE

In order to pass traffic from your clients to the servers through the BIG-IP system, you must create a virtual server on the BIG-IP VE. In this template, the AS3 declaration creates 2 VIPs: one for public internet facing, and one for private internal usage. It is preconfigured as an example.

***Note:*** These next steps illustrate the manual way in the GUI to create a virtual server
1. Open the BIG-IP VE Configuration utility
2. Click **Local Traffic > Virtual Servers**
3. Click the **Create** button
4. Type a name in the **Name** field
4. Type an address (ex. x.x.x.x/x) in the **Destination/Mask** field
5. Type a port (ex. 443) in the **Service Port**
6. Configure the rest of the virtual server as appropriate
7. Select a pool name from the **Default Pool** list
8. Click the **Finished** button
9. Repeat as necessary for other applications

## Redeploy BIG-IP for Replacement or Upgrade
This example illustrates how to replace or upgrade the BIG-IP VE.
  1. Change the *bigip_version* variable to the desired release 
  2. Revoke the problematic BIG-IP VE's license (if BYOL)
  3. Run command
```
terraform destroy -target azurerm_virtual_machine.f5vm01
```
  3. Run command
```
terraform apply
```
  4. Repeat steps 1-3 on the other BIG-IP VE in order to establish Device Trust.


## Rerun Declarative Onboarding on the BIG-IP VE
This example illustrates how to re-configure the BIG-IP instances with DO. If you need to make changes to the L1-L3 settings of the BIG-IP device, you can follow these steps.
  1. Update do.json as needed
  2. Taint resources and apply
```
terraform taint template_file.vm01_do_json
terraform taint null_resource.f5vm01_DO
terraform apply
```

## Rerun Application Services AS3 on the BIG-IP VE
This example illustrates how to run your own custom AS3 (aka application). You can have a catalog of AS3 apps/templates and repeat these steps as many times as desired.
  1. Update as3.json as needed
  2. Taint resources and apply
```
terraform taint template_file.as3_json
terraform taint null_resource.f5vm_AS3
terraform apply
```

## Rerun Telemetry Streaming on the BIG-IP VE
This example illustrates how to re-configure the BIG-IP instances with TS. If you need to make changes to the push consumers (ex. Azure Log Analytics, Splunk, etc) or other telemetry configs of the BIG-IP device, you can follow these steps.
  1. Update ts.json as needed
  2. Taint resources and apply
```
terraform taint template_file.vm_ts_file
terraform taint null_resource.f5vm01_TS
terraform taint null_resource.f5vm02_TS
terraform apply
```

## Service Principal Authentication
This solution requires access to the Azure API to determine how the BIG-IP VEs should be configured. The following provides information/links on the options for configuring a service principal within Azure if this is the first time it is needed in a subscription.

_Ensure that however the creation of the service principal occurs to verify it only has minimum required access based on the solutions need(read vs read/write) prior to this template being deployed and used by the solution within the resource group selected(new or existing)._

The end result should be possession of a client(application) ID, tenant ID and service principal secret that can login to the same subscription this template will be deployed into. Ensuring this is fully functioning prior to deploying this ARM template will save on some troubleshooting post-deployment if the service principal is in fact not fully configured.

As another ference...ead over to F5 CloudDocs to see an example in one of the awesome lab guides. Pay attention to the [Setting Up a Service Principal Account](https://clouddocs.f5.com/training/community/big-iq-cloud-edition/html/class2/module5/lab1.html#setting-up-a-service-principal-account) section and then head back over here!

### Option #1 Azure Portal

Follow the steps outlined in the [Azure Portal documentation](https://azure.microsoft.com/en-us/documentation/articles/resource-group-create-service-principal-portal/) to generate the service principal.

### Option #2 Azure CLI

This method can be used with either the [Azure CLI v2.0 (Python)](https://github.com/Azure/azure-cli) or the [Azure Cross-Platform CLI (npm module)](https://github.com/Azure/azure-xplat-cli).

_Using the Python Azure CLI v2.0 - requires just one step_
```shell
$ az ad sp create-for-rbac
```

_Using the Node.js cross-platform CLI - requires additional steps for setting up_
https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-authenticate-service-principal-cli

### Option #3 Azure PowerShell
Follow the steps outlined in the [Azure Powershell documentation](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-authenticate-service-principal) to generate the service principal.
# api-protection-bootcamp
