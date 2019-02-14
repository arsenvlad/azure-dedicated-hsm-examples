# Azure Dedicated HSM Examples
Simple examples of deploying Azure Dedicated HSM. See https://azure.microsoft.com/en-us/services/azure-dedicated-hsm/ for more information regarding pre-requisites, pricing, and detailed docs.

NOTE: These deployment steps assume that you already have a VNet with sufficient address space to create additional required subnets: one for Express Route gateway and one for Dedicated HSM devices. Also, your subscription must be already onboarded for the Azure Dedicated HSM service as described here https://docs.microsoft.com/en-us/azure/dedicated-hsm/tutorial-deploy-hsm-cli#prerequisites.

### Check how VNet resource looks now
```
az network vnet show -g avtest1 -n avtest1-vnet -o json

az network vnet subnet list -g avtest1 --vnet-name avtest1-vnet -o table
```

### Create GatewaySubnet for the required Express Route Gateway (if it does not yet exist)
```
az network vnet subnet create -g avtest1 --vnet-name avtest1-vnet --name GatewaySubnet --address-prefixes 10.112.1.0/24
```

### Create subnet for the dedicated HSM devices with the required delegation
(as you can see in the following doc, currently Network Security Groups (NSG) and User Defined Routs (UDR) are not supported on delegated subnets https://docs.microsoft.com/en-us/azure/dedicated-hsm/networking#subnets)
```
az network vnet subnet create -g avtest1 --vnet-name avtest1-vnet --name HsmSubnet --address-prefixes 10.112.2.0/24 --delegations Microsoft.HardwareSecurityModules/dedicatedHSMs
```

### Create Express Route Gateway (this may take 25-35 minutes)
```
az network vnet subnet show -g avtest1 --vnet-name avtest1-vnet --name GatewaySubnet --query "id" -o tsv

az group deployment create -g avtest1 --template-file create-expressroute-gateway.json --parameters existingGatewaySubnetID=ID_GOES_HERE
```

### Create HSM devices (for HA pair make sure to place first HSM in "stamp1" and send HSM in "stamp2")
```
az network vnet subnet show -g avtest1 --vnet-name avtest1-vnet --name HsmSubnet --query "id" -o tsv

az group create -n avtest1-hsm -l eastus2
az group deployment create -g avtest1-hsm --template-file create-hsm.json --parameters hsmStampID=stamp1 existingHsmSubnetID=ID_GOES_HERE
```

### Get HSM IP address
```
az resource list -g avtest1-hsm --query "[*].id" -o tsv

az resource show --ids /subscriptions/eeb84c6a-7753-4a8a-b39e-d66612be62a0/resourceGroups/avtest1-hsm/providers/Microsoft.HardwareSecurityModules/dedicatedHSMs/avtest1-hsm1 --query "properties.networkProfile.networkInterfaces[*].privateIpAddress" -o table
```

### Ping HSM device from VM in the same VNet
```
[azureuser@aveastus2vm1 ~]$ ping 10.112.2.4
PING 10.112.2.4 (10.112.2.4) 56(84) bytes of data.
64 bytes from 10.112.2.4: icmp_seq=1 ttl=62 time=1.38 ms
64 bytes from 10.112.2.4: icmp_seq=2 ttl=62 time=1.16 ms
64 bytes from 10.112.2.4: icmp_seq=3 ttl=62 time=1.30 ms
64 bytes from 10.112.2.4: icmp_seq=4 ttl=62 time=1.11 ms
64 bytes from 10.112.2.4: icmp_seq=5 ttl=62 time=1.18 ms
64 bytes from 10.112.2.4: icmp_seq=6 ttl=62 time=1.14 ms
64 bytes from 10.112.2.4: icmp_seq=7 ttl=62 time=1.24 ms
64 bytes from 10.112.2.4: icmp_seq=8 ttl=62 time=1.22 ms
```

### SSH into the HSM device 
(default password is PASSWORD and will need to be changed on first login)
```
ssh tenantadmin@10.112.2.4
```

### Change password and make sure to record it
```
[azureuser@aveastus2vm1 ~]$ ssh tenantadmin@10.112.2.4
The authenticity of host '10.112.2.4 (10.112.2.4)' can't be established.
RSA key fingerprint is 2b:02:1d:f7:11:4e:15:4d:5b:db:17:a0:f5:84:2c:f2.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.112.2.4' (RSA) to the list of known hosts.
tenantadmin@10.112.2.4's password:
Last login: Sun Feb 10 23:17:02 2019

Luna Network HSM Command Line Shell v7.2.0-220. Copyright (c) 2018 SafeNet. All rights reserved.


*****************************************************
**                                                 **
**   For security purposes, you must change your   **
**   password.                                     **
**                                                 **
**   Please ensure you store your new password     **
**   in a secure location.                         **
**                                                 **
**               DO NOT LOSE IT!                   **
**                                                 **
*****************************************************

Changing password for user tenantadmin.

You can now choose the new password.

The password must be at least 8 characters long.
The password must contain characters from at least 3 of the following 4 categories:
    - Uppercase letters (A through Z)
    - Lowercase letters (a through z)
    - Numbers (0 through 9)
    - Non-alphanumeric characters (such as !, $, #, %)

New password:
Retype new password:
passwd: all authentication tokens updated successfully.
Password change successful.

[local_host] lunash:>

```

### View HSM Status
```
[local_host] lunash:>hsm show


   Appliance Details:
   ==================
   Software Version:                7.2.0-220

   HSM Details:
   ============
   HSM Label:
   Serial #:                           572413
   Firmware:                           7.0.3
   HSM Model:                          Luna K7
   HSM Part Number:                    808-000066-001
   Authentication Method:              Password
   HSM Admin login status:             Not Logged In
   HSM Admin login attempts left:      HSM IS ZEROIZED !!!!
   RPV Initialized:                    No
   Audit Role Initialized:             Yes
   Remote Login Initialized:           No
   Manually Zeroized:                  No
   Secure Transport Mode:              No
   HSM Tamper State:                   No tamper(s)

   Partitions created on HSM:
   ==============================
   There are no partitions.


   Number of partitions allowed:        10
   Number of partitions created:        0

   FIPS 140-2 Operation:
   =====================
   The HSM is NOT in FIPS 140-2 approved operation mode.

   HSM Storage Information:
   ========================
   Maximum HSM Storage Space (Bytes):   33554432
   Space In Use (Bytes):                0
   Free Space Left (Bytes):             33554432

   Environmental Information on HSM:
   =================================
   Battery Voltage:                     3.093 V
   Battery Warning Threshold Voltage:   2.750 V
   System Temp:                         35 deg. C
   System Temp Warning Threshold:       75 deg. C



Command Result : 0 (Success)
```

## Network Security Groups
As you can see in the following doc, currently (February 2019) Network Security Groups (NSG) and User Defined Routs (UDR) are not supported on delegated subnets https://docs.microsoft.com/en-us/azure/dedicated-hsm/networking#subnets

Therefore, current approach to restrict access to the HSM subnet is by applying outbound NSG rules to all other subnets (on a per subnet basis) where VMs are running to control which VMs can/cannot communicate with the HSM subnet. It is also possible to apply NSG rules or Application Security Rules to individual NICs instead of at the subnet level, but it would mean each new NIC needs the rules and they shouldn't conflict with subnet rules.
