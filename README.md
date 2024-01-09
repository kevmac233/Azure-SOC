# Building a SOC + Honeynet in Azure (Live Traffic)
![Cloud Honeynet / SOC](https://i.imgur.com/ZWxe03e.jpg)

## Introduction

In this project, I build a mini honeynet in Azure and ingest log sources from various resources into a Log Analytics workspace, which is then used by Microsoft Sentinel to build attack maps, trigger alerts, and create incidents. I measured some security metrics in the insecure environment for 24 hours, apply some security controls to harden the environment, measure metrics for another 24 hours, then show the results below. The metrics we will show are:

- SecurityEvent (Windows Event Logs)
- Syslog (Linux Event Logs)
- SecurityAlert (Log Analytics Alerts Triggered)
- SecurityIncident (Incidents created by Sentinel)
- AzureNetworkAnalytics_CL (Malicious Flows allowed into our honeynet)

## Creating a Windows Virtual Machine with SQL Server Installation

To begin this project, I created a Windows virtual machine in Azure with SQL Server installed:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/d461665b-7082-4f4f-94e0-7d8c721ac524)

In Microsoft Azure, I configured the Network Security Group (NSG), which is the firewall built in Azure, by creating a rule to allow all inbound traffic:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/3bc5ddf5-1725-4618-a638-b0d70c58fb77)

After creating the Windows VM, I used the RDP from my local desktop to get onto the VM. Once remoted in, I turned off Windows Defender Firewall to allow any traffic, making this VM vulnerable:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/7165d995-903c-4518-a827-2ee811b30f5d)

The next step was to install the free trial version of SQL Server Evaluation and SQL Management Studio (SSMS). I configured the SQL Server to enable logging and added a "NETWORK SERVICE" permission in Registry Editor to provide full permission for the SQL Server service account:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/a69e66a0-b9ec-4ee8-95ee-55b39ae789db)

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/3aedc271-6742-4a49-b4f3-9c3e01d86f28)


To confirm that logging was enabled, I simulated a "failed" login attempt into the SQL Server and ported into Windows Event Viewer:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/b5ddf696-c1ba-4c36-b980-38d73569382f)

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/b8d9a6a7-e66b-45b6-8959-7b018782947a)

## Creating a Secondary Windows VM to Simulate an Attacker

This next part, I created a second VM which simulated an attacker. I intentionally failed to log into the SQL Server VM by RDP multiple times to generate logs in the Event Viewer of the SQL Server VM:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/db9b9ac9-d1aa-43f4-9c97-4d6899c1402f)

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/ccd782c5-9d49-49e4-88da-5cc76db7c0c3)

Here, I remoted back into the SQL Server VM to check the Event Viewer of the failed login attempts conducted by my Attacker VM. I filtered the log events to 4625 (Event ID for failed RDP in Windows):

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/6e1d16af-a2fd-4e0c-b5b6-4e7f40102232)

Back on the Attacker VM, I installed SSMS and intentionally failed to log into the SQL Server of my other Windows VM multiple times, and then successfully connect to it:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/909fbb96-09f1-464d-ac09-21d192b3af9e)

Remoted back onto the non-attacker VM and verified the successful login from the Attacker VM in Event Viewer. I observed the Source IP addresses, timestamps, etc.:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/ab129f28-6dd1-4a2e-be6b-91f3d5a65f55)

## Creating a Linux VM

Created a third virtual machine running Ubuntu Linux and created a rule to allow any inbound traffic:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/4dab131e-c371-4e29-b2d1-7a47d1037923)

## Creating a Log Analytics Workspace

At this part of the project, I created a Log Analytics Workspace which will act as the log aggregator:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/9f993ab6-27de-407d-ba8b-28fcdce37b44)

Enabled Microsoft Defender for Cloud plans for Log Analytics Workspace, and configured it to allow data collection for all events to send to Log Analytics Workspace:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/777ddf07-4653-4bed-8b7d-4572660e3237)



## Architecture After Hardening / Security Controls
![Architecture Diagram](https://i.imgur.com/YQNa9Pp.jpg)

The architecture of the mini honeynet in Azure consists of the following components:

- Virtual Network (VNet)
- Network Security Group (NSG)
- Virtual Machines (2 windows, 1 linux)
- Log Analytics Workspace
- Azure Key Vault
- Azure Storage Account
- Microsoft Sentinel

For the "BEFORE" metrics, all resources were originally deployed, exposed to the internet. The Virtual Machines had both their Network Security Groups and built-in firewalls wide open, and all other resources are deployed with public endpoints visible to the Internet; aka, no use for Private Endpoints.

For the "AFTER" metrics, Network Security Groups were hardened by blocking ALL traffic with the exception of my admin workstation, and all other resources were protected by their built-in firewalls as well as Private Endpoint

## Attack Maps Before Hardening / Security Controls
![NSG Allowed Inbound Malicious Flows](https://i.imgur.com/1qvswSX.png)<br>
![Linux Syslog Auth Failures](https://i.imgur.com/G1YgZt6.png)<br>
![Windows RDP/SMB Auth Failures](https://i.imgur.com/ESr9Dlv.png)<br>

## Metrics Before Hardening / Security Controls

The following table shows the metrics we measured in our insecure environment for 24 hours:
Start Time 2023-03-15 17:04:29
Stop Time 2023-03-16 17:04:29

| Metric                   | Count
| ------------------------ | -----
| SecurityEvent            | 19470
| Syslog                   | 3028
| SecurityAlert            | 10
| SecurityIncident         | 348
| AzureNetworkAnalytics_CL | 843

## Attack Maps Before Hardening / Security Controls

```All map queries actually returned no results due to no instances of malicious activity for the 24 hour period after hardening.```

## Metrics After Hardening / Security Controls

The following table shows the metrics we measured in our environment for another 24 hours, but after we have applied security controls:
Start Time 2023-03-18 15:37
Stop Time	2023-03-19 15:37

| Metric                   | Count
| ------------------------ | -----
| SecurityEvent            | 8778
| Syslog                   | 25
| SecurityAlert            | 0
| SecurityIncident         | 0
| AzureNetworkAnalytics_CL | 0

## Conclusion

In this project, a mini honeynet was constructed in Microsoft Azure and log sources were integrated into a Log Analytics workspace. Microsoft Sentinel was employed to trigger alerts and create incidents based on the ingested logs. Additionally, metrics were measured in the insecure environment before security controls were applied, and then again after implementing security measures. It is noteworthy that the number of security events and incidents were drastically reduced after the security controls were applied, demonstrating their effectiveness.

It is worth noting that if the resources within the network were heavily utilized by regular users, it is likely that more security events and alerts may have been generated within the 24-hour period following the implementation of the security controls.
