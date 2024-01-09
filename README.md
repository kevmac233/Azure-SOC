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

Created a third virtual machine running Ubuntu Linux and created a rule to allow any inbound traffic. Similar to the Windows VM with SQL Server, allowing all traffic will make this VM vulnerable:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/4dab131e-c371-4e29-b2d1-7a47d1037923)

## Creating Users in Azure Active Directory

In this section, I created users in Azure Active Directory and assigned a role to one of the users:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/6b2a4cce-50bd-4c56-b7ca-dd517546f909)

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/708e1d2f-47d6-4eb9-857e-809482ca1dfe)


## Creating a Log Analytics Workspace

At this part of the project, I created a Log Analytics Workspace which will act as the log aggregator in Azure:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/9f993ab6-27de-407d-ba8b-28fcdce37b44)

Enabled Microsoft Defender for Cloud plans for Log Analytics Workspace, and configured it to allow data collection for all events to send to Log Analytics Workspace:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/777ddf07-4653-4bed-8b7d-4572660e3237)

## Configuring and Utilizing Microsoft Sentinel (SIEM)

Microsoft Sentinel is a tool in Azure that acts as a Security Information and Event Management (SIEM). This is where all the security alerts will generate, which prioritizes from high to low alerts. It allows you to investigate those alerts as well. Here, I manually created a "Brute Force Attempt" alert rule in Sentinel's Active Rules to test against both the Windows VM and Linux VM:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/4e4dcd87-91ba-4c28-a2bf-5796d2798b01)

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/87247b20-4fe2-4f33-bc52-6ec1552d1235)

## Simulating an attack on Azure Active Directory

This is where I began my attack simulations, and I started with an Azure AD brute force attack. To perform the attack, I remoted back into the Attacker VM and opened Powershell ISE. I ran a premade script that performs a brute force attack by failing to login 11 times and then completes with a successful login attempt:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/6fbd01da-16d6-4064-8a36-f3641cef897a)

This attack was logged and can be observed in Log Analytics Workspace and Microsoft Sentinel. In this case, I assigned the alert to myself to simulate as if I was an analyst investigating the alert:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/8ad4e1dd-5252-46d1-a0ac-4b99c30855dd)

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/c8bd3445-29a3-436a-ae8f-a5b1d646a000)

## Simulating an attack on Windows VM and Linux VM

In this simulation, I conducted a brute force attack from the Attacker VM on both vulnerable VMs. Here, I ran a premade script in Visual Studio Code that attempts to log into the SQL Server of my Windows VM. This generated an alert in Microsoft Sentinel, which I assigned to myself to simulate Incident Response:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/4e45c0c2-f306-4431-864b-95448c0445e3)

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/b313ffe1-fc63-44f6-a37d-31c6923c6929)

Here, I conducted a brute force attack against my Linux VM using Powershell from my local desktop. This generated a security alert in Microsoft Sentinel:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/3f78c214-94fc-463d-b178-4bd505c27307)

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/ff5eebe9-1bf4-4e20-a8e2-afdf6e8c211a)

Another attack I performed malware breakout against my Windows VM. I conducted this by running a premade Powershell script to generate EICAR.txt files. EICAR.txt files do not contain real viruses, but they are designed to help test and check whether your antivirus software is working properly:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/490ee5ef-f4b0-4fef-80ff-e963fb0f5237)

Here is the incident alert generated in Microsoft Sentinel from the malware attack:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/4462bb0b-cdd9-4c55-9e4f-461dce70bd6b)

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/d1e7eced-e49d-408a-8088-14212af68b7e)

## Investigation/Incident Response

After conducting those attacks, I simulated an incident response in compliance with NIST 800-61 to get more detailed information on the attacks. In Microsoft Sentinel, there is a feature called Incidents which allows you to see a visual representation of the attack:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/a8022ef4-9c4d-45e3-9d56-748c2cf39c7e)
![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/59a15ee5-027c-4c93-a34c-5894be85747d)
![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/3c6d213a-81fa-457f-b5f1-b5f57ab0a279)

In Log Analytics Workspace, I ran a KQL (Kusto Query Language) query to grab Malware Detection logs to further investigate. Since the alert was triggered by me, this alert would be considered a False Positive alert:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/52217753-86c2-479c-a76d-d336b193780b)

In Microsoft Defender for Cloud, there is a feature to check the security posture of the systems. It grades you depending on security controls implemented and recommends some to further increase security:

![image](https://github.com/kevmac233/Azure-SOC/assets/125979597/08df282d-4383-4240-9771-b765d9d4941c)

## Conclusion

In this project, a mini honeynet was constructed in Microsoft Azure and log sources were integrated into a Log Analytics Workspace. Microsoft Sentinel was employed to trigger alerts and create incidents based on the ingested logs. Although these attacks were conducted by me, it is crucial to understand the NIST 800-53 and NIST 800-61

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
