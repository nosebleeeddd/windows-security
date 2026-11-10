# Windows Security & Hardening Guide

A practical reference for securing Windows workstations, servers, and enterprise environments through system hardening, access control, network security, endpoint protection, monitoring, and incident response.

This guide covers Windows security architecture, Group Policy, User Account Control, Windows Defender, BitLocker, Windows Firewall, PowerShell logging, registry security, application control, and security auditing.

The goal is to understand not only how to configure Windows security controls, but also how to verify that they work and investigate when they fail.

> **Authorization notice:** Perform configuration changes and security assessments only on systems you own or are authorized to administer. Test hardening configurations in a virtual machine or controlled lab before applying them to production systems.

---

## Table of Contents

1. [Introduction](#introduction)
2. [Windows Security Architecture](#windows-security-architecture)
3. [Initial Security Assessment](#initial-security-assessment)
4. [Group Policy Management](#group-policy-management)
5. [User Account Control (UAC)](#user-account-control-uac)
6. [Windows Firewall Configuration](#windows-firewall-configuration)
7. [BitLocker and Drive Encryption](#bitlocker-and-drive-encryption)
8. [Microsoft Defender Configuration](#microsoft-defender-configuration)
9. [Event Logging and Monitoring](#event-logging-and-monitoring)
10. [PowerShell Security](#powershell-security)
11. [Registry Security](#registry-security)
12. [Network Security](#network-security)
13. [Application Security and Control](#application-security-and-control)
14. [Compliance and Security Benchmarks](#compliance-and-security-benchmarks)
15. [Security Tools and Automation](#security-tools-and-automation)
16. [Incident Response](#incident-response)
17. [Recommended Lab Projects](#recommended-lab-projects)
18. [References](#references)

---

## Introduction

Windows is widely deployed in personal computing, enterprise infrastructure, and Active Directory environments. Its security depends on multiple layers of protection working together.

### Security objectives

| Objective | Description |
|---|---|
| Reduce attack surface | Remove unnecessary exposure and services |
| Protect identities | Secure accounts, credentials, and authentication |
| Enforce least privilege | Limit unnecessary administrative access |
| Protect data | Use encryption and appropriate access controls |
| Monitor activity | Collect and analyze security events |
| Prevent malware | Use endpoint protection and application control |
| Maintain compliance | Apply documented security baselines |
| Improve resilience | Maintain backups, recovery procedures, and incident response plans |

### Windows editions and environments

Security features can differ between Windows editions and versions.

| Environment | Typical use |
|---|---|
| Windows 11 Home | Personal workstation |
| Windows 11 Pro | Professional workstation and business use |
| Windows Enterprise | Managed enterprise endpoints |
| Windows Server | Server infrastructure and domain services |
| Windows Server Core | Minimal server installation |

Some features, such as Group Policy management, BitLocker management, AppLocker, and advanced security policies, have edition-specific requirements.

Always verify feature availability before deploying a configuration.

---

## Windows Security Architecture

Windows security is implemented through multiple operating-system components and security technologies.

### 1. Windows security components

| Component | Purpose |
|---|---|
| Security Reference Monitor | Enforces access-control decisions |
| Local Security Authority (LSA) | Handles local security policy and authentication-related operations |
| Security Accounts Manager (SAM) | Stores local account information |
| Local Security Authority Subsystem Service (LSASS) | Hosts important authentication and security services |
| Windows Defender | Provides antivirus and endpoint protection capabilities |
| Windows Firewall | Controls network traffic |
| User Account Control | Helps prevent unintended administrative changes |
| Windows Security Center | Provides centralized security status information |

### 2. Windows security layers

```text
Applications
     |
     v
User Mode
     |
     v
Windows APIs and Security Services
     |
     v
Security Subsystems
     |
     v
Windows Kernel
     |
     v
Hardware and Firmware
```

### 3. Built-in security technologies

Important Windows security technologies include:

- Windows Defender Antivirus.
- Microsoft Defender Firewall.
- BitLocker Drive Encryption.
- Secure Boot.
- Trusted Platform Module (TPM).
- Credential Guard, where supported.
- Virtualization-Based Security (VBS).
- Hypervisor-Protected Code Integrity (HVCI).
- Windows Hello.
- Application Control.

### 4. Check system information

```powershell
Get-ComputerInfo |
    Select-Object WindowsProductName,
                  WindowsVersion,
                  OsBuildNumber,
                  CsManufacturer,
                  CsModel
```

Check TPM information:

```powershell
Get-Tpm
```

Check Secure Boot status:

```powershell
Confirm-SecureBootUEFI
```

Secure Boot checks require compatible UEFI hardware and firmware configuration.

### 5. Export local security policy

```powershell
secedit /export /cfg "$env:TEMP\SecurityBaseline.inf"
```

Review the exported configuration:

```powershell
Get-Content "$env:TEMP\SecurityBaseline.inf"
```

This provides a useful starting point for examining local security settings.

---

## Initial Security Assessment

Before changing system settings, establish the current security posture.

A baseline makes it easier to identify misconfigurations and verify that hardening changes actually worked.

### 1. Check firewall status

```powershell
Get-NetFirewallProfile |
    Select-Object Name,
                  Enabled,
                  DefaultInboundAction,
                  DefaultOutboundAction
```

### 2. Check Defender status

```powershell
Get-MpComputerStatus |
    Select-Object AntivirusEnabled,
                  RealTimeProtectionEnabled,
                  BehaviorMonitorEnabled,
                  AntivirusSignatureLastUpdated
```

### 3. Check BitLocker

```powershell
Get-BitLockerVolume
```

### 4. Check UAC configuration

```powershell
Get-ItemProperty `
    -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" |
    Select-Object EnableLUA,
                  ConsentPromptBehaviorAdmin,
                  PromptOnSecureDesktop
```

### 5. Check local administrators

```powershell
Get-LocalGroupMember -Group "Administrators"
```

### 6. Check local users

```powershell
Get-LocalUser |
    Select-Object Name,
                  Enabled,
                  LastLogon,
                  PasswordLastSet
```

### 7. Check installed updates

```powershell
Get-HotFix |
    Sort-Object InstalledOn -Descending |
    Select-Object -First 15
```

`Get-HotFix` does not represent every installed update or provide a complete vulnerability assessment. Use Windows Update management and vulnerability-management tooling for comprehensive coverage.

### 8. Initial assessment checklist

- [ ] Windows is supported and receiving security updates.
- [ ] Microsoft Defender is functioning.
- [ ] Firewall profiles are enabled.
- [ ] UAC is enabled.
- [ ] Local administrator membership is reviewed.
- [ ] BitLocker is enabled where appropriate.
- [ ] Secure Boot is enabled where supported.
- [ ] Unnecessary services and applications are reviewed.
- [ ] PowerShell logging is configured.
- [ ] Security event logs are being collected.
- [ ] Recovery procedures and backups are available.

---

## Group Policy Management

Group Policy provides centralized configuration management for Windows systems, especially in Active Directory environments.

It allows administrators to enforce security settings across users and computers.

### 1. Group Policy locations

Important policy categories include:

```text
Computer Configuration
    Windows Settings
        Security Settings
            Account Policies
            Local Policies
            Windows Defender Firewall
            Restricted Groups
            System Services
```

Additional administrative template policies are available under:

```text
Computer Configuration
    Administrative Templates
        Windows Components
            Microsoft Defender Antivirus
            Windows PowerShell
            BitLocker Drive Encryption
            Windows Remote Management
```

### 2. Account policies

Important settings include:

- Password policy.
- Account lockout policy.
- Kerberos policy.
- Authentication requirements.

For domain environments, account policies must be configured with an understanding of domain-level and fine-grained password policies.

### 3. Inspect effective policy

Generate a Group Policy Results report:

```powershell
gpresult /h "$env:TEMP\GPResult.html"
```

Open it:

```powershell
Start-Process "$env:TEMP\GPResult.html"
```

Force a policy refresh:

```powershell
gpupdate /force
```

### 4. Manage Group Policy with PowerShell

Import the module:

```powershell
Import-Module GroupPolicy
```

List Group Policy Objects:

```powershell
Get-GPO -All |
    Select-Object DisplayName,
                  GpoStatus,
                  CreationTime,
                  ModificationTime
```

Generate a GPO report:

```powershell
New-Item -ItemType Directory -Path "C:\SecurityReports" -Force

Get-GPOReport -All `
    -ReportType HTML `
    -Path "C:\SecurityReports\AllGPOs.html"
```

Create a GPO:

```powershell
New-GPO -Name "Workstation Security Baseline" `
    -Comment "Approved workstation security configuration"
```

Linking a GPO changes policy for the target organizational unit. Test the GPO and verify its scope before linking it to production systems.

### 5. Recommended Group Policy controls

| Area | Recommended control |
|---|---|
| Account security | Enforce appropriate password and lockout policies |
| User rights | Restrict sensitive privileges |
| Administrative access | Limit local administrator membership |
| Remote access | Restrict RDP and remote management |
| Windows Defender | Enforce endpoint protection settings |
| Firewall | Apply consistent inbound and outbound policies |
| PowerShell | Enable logging |
| BitLocker | Configure encryption and recovery-key management |
| Auditing | Enable relevant security events |
| Application control | Restrict unauthorized software |

---

## User Account Control (UAC)

UAC helps prevent applications and users from making administrative changes without appropriate elevation.

It is an important defense-in-depth control.

### 1. Check UAC status

```powershell
Get-ItemProperty `
    -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" |
    Select-Object EnableLUA,
                  ConsentPromptBehaviorAdmin,
                  PromptOnSecureDesktop
```

Important values:

| Setting | Purpose |
|---|---|
| EnableLUA | Enables UAC |
| ConsentPromptBehaviorAdmin | Controls administrator elevation prompts |
| PromptOnSecureDesktop | Controls secure-desktop elevation prompts |
| FilterAdministratorToken | Controls Admin Approval Mode for the built-in Administrator account |

### 2. UAC recommendations

- Keep UAC enabled.
- Avoid configuring silent elevation for ordinary administrative activity.
- Use standard user accounts for everyday work.
- Use dedicated administrative accounts.
- Review elevation prompts carefully.
- Do not treat UAC as a complete defense against privilege escalation.

### 3. Configure UAC through policy

Use the Local Security Policy console or Group Policy:

```text
Computer Configuration
    Windows Settings
        Security Settings
            Local Policies
                Security Options
```

Review the policies beginning with:

```text
User Account Control:
```

### 4. Why UAC matters

UAC reduces the risk of unintended system-wide changes. It is not a replacement for least privilege, application control, endpoint protection, or secure account management.

Avoid changing UAC registry values without understanding the effect on the operating system and applications.

---

## Windows Firewall Configuration

Microsoft Defender Firewall provides host-based network traffic filtering.

It supports domain, private, and public network profiles.

### 1. Check firewall profiles

```powershell
Get-NetFirewallProfile
```

### 2. Enable firewall profiles

Run from an elevated PowerShell session:

```powershell
Set-NetFirewallProfile -Profile Domain,Private,Public -Enabled True
```

Verify:

```powershell
Get-NetFirewallProfile |
    Select-Object Name, Enabled
```

### 3. Configure default actions

A common baseline is to block unsolicited inbound connections and allow outbound connections unless a stricter policy is required.

```powershell
Set-NetFirewallProfile `
    -Profile Domain,Private,Public `
    -DefaultInboundAction Block `
    -DefaultOutboundAction Allow
```

Be careful when changing default actions on servers or managed systems. Existing firewall rules and business requirements must be considered.

### 4. Review firewall rules

```powershell
Get-NetFirewallRule |
    Select-Object DisplayName,
                  Enabled,
                  Direction,
                  Action,
                  Profile
```

List enabled inbound rules:

```powershell
Get-NetFirewallRule -Enabled True -Direction Inbound |
    Select-Object DisplayName, Action, Profile
```

### 5. Create a restricted inbound rule

Example: permit an internal management network to reach a specific TCP service.

```powershell
New-NetFirewallRule `
    -DisplayName "Allow Internal HTTPS Management" `
    -Direction Inbound `
    -Protocol TCP `
    -LocalPort 443 `
    -RemoteAddress 192.168.1.0/24 `
    -Action Allow `
    -Profile Private
```

Replace the example subnet with the actual approved management network.

Do not create broad inbound rules for services that do not need to be exposed.

### 6. Block an unnecessary outbound destination port

```powershell
New-NetFirewallRule `
    -DisplayName "Block Outbound Telnet" `
    -Direction Outbound `
    -Protocol TCP `
    -RemotePort 23 `
    -Action Block
```

Validate that the rule does not interfere with legitimate applications.

### 7. Enable firewall logging

```powershell
Set-NetFirewallProfile `
    -Profile Domain,Private,Public `
    -LogBlocked True `
    -LogAllowed False `
    -LogMaxSizeKilobytes 16384
```

Default firewall log location:

```text
%systemroot%\system32\LogFiles\Firewall\pfirewall.log
```

### 8. Review active firewall configuration

```powershell
Get-NetFirewallProfile |
    Format-Table Name, Enabled, DefaultInboundAction, DefaultOutboundAction
```

### 9. Firewall hardening checklist

- [ ] Firewall enabled on all applicable profiles.
- [ ] Unnecessary inbound rules removed.
- [ ] Administrative access restricted to approved networks.
- [ ] Public profile configured conservatively.
- [ ] Firewall logging enabled.
- [ ] Application-specific rules reviewed.
- [ ] Firewall changes documented and tested.

---

## BitLocker and Drive Encryption

BitLocker protects data at rest by encrypting supported Windows volumes.

It is particularly important for laptops and portable systems that could be lost or stolen.

### 1. Check TPM status

```powershell
Get-Tpm
```

### 2. Check BitLocker status

```powershell
Get-BitLockerVolume
```

Alternative:

```powershell
manage-bde -status
```

### 3. Check available encryption methods

BitLocker configuration depends on the Windows edition, hardware, policy, and volume type.

Review the supported encryption configuration before deployment.

### 4. Enable BitLocker

On a compatible system with a prepared recovery process, an example command is:

```powershell
Enable-BitLocker `
    -MountPoint "C:" `
    -EncryptionMethod XtsAes256 `
    -UsedSpaceOnly `
    -TpmProtector
```

This example assumes TPM protection is supported and appropriate for the device.

**Before enabling encryption:**

1. Verify that the device supports BitLocker.
2. Confirm that recovery-key storage is configured.
3. Back up important data.
4. Verify TPM health.
5. Confirm that the recovery process is tested.
6. Ensure that the selected encryption policy meets organizational requirements.

### 5. Recovery-key management

Recovery keys are critical. If the TPM or boot configuration changes, recovery may be required.

In managed environments, recovery information should be securely escrowed to an approved destination such as Active Directory Domain Services or Microsoft Entra ID, depending on the deployment.

Inspect protectors:

```powershell
(Get-BitLockerVolume -MountPoint "C:").KeyProtector
```

Back up a recovery password protector to Active Directory when the environment is configured to support it:

```powershell
Backup-BitLockerKeyProtector `
    -MountPoint "C:" `
    -KeyProtectorId "<RECOVERY-PROTECTOR-ID>"
```

Replace the placeholder with the actual recovery protector ID.

Never place recovery passwords or keys in public repositories, ordinary scripts, or unsecured logs.

### 6. BitLocker verification

```powershell
Get-BitLockerVolume |
    Select-Object MountPoint,
                  VolumeStatus,
                  ProtectionStatus,
                  EncryptionPercentage
```

### 7. BitLocker hardening recommendations

- Use TPM-backed protection where appropriate.
- Store recovery keys securely.
- Verify recovery-key escrow.
- Protect firmware and boot configuration.
- Use approved encryption policies.
- Document recovery procedures.
- Test recovery in a controlled environment.

---

## Microsoft Defender Configuration

Microsoft Defender Antivirus provides malware detection, real-time protection, and other endpoint security capabilities.

### 1. Check Defender status

```powershell
Get-MpComputerStatus |
    Select-Object AntivirusEnabled,
                  RealTimeProtectionEnabled,
                  BehaviorMonitorEnabled,
                  IoavProtectionEnabled,
                  AntivirusSignatureLastUpdated
```

### 2. Update security intelligence

```powershell
Update-MpSignature
```

### 3. Run a quick scan

```powershell
Start-MpScan -ScanType QuickScan
```

Run a full scan:

```powershell
Start-MpScan -ScanType FullScan
```

### 4. Review Defender preferences

```powershell
Get-MpPreference |
    Select-Object DisableRealtimeMonitoring,
                  DisableBehaviorMonitoring,
                  DisableIOAVProtection,
                  MAPSReporting,
                  PUAProtection
```

### 5. Review threat detections

```powershell
Get-MpThreatDetection |
    Select-Object InitialDetectionTime,
                  ThreatID,
                  ActionSuccess,
                  Resources
```

### 6. Review Defender exclusions

```powershell
Get-MpPreference |
    Select-Object ExclusionPath,
                  ExclusionProcess,
                  ExclusionExtension
```

Unnecessary exclusions can create security blind spots. Every exclusion should have a documented business reason and appropriate approval.

### 7. Attack Surface Reduction

Attack Surface Reduction (ASR) rules help reduce behaviors commonly abused by malware and attackers.

They can be configured through Microsoft Defender management tools and Group Policy, depending on the environment.

Example rule identifier:

```text
D4F940AB-401B-4EFC-AADC-AD5F3C50688A
```

This is an ASR rule identifier associated with blocking Office applications from creating child processes.

Before enabling ASR rules:

- Review the current Microsoft documentation.
- Test in audit mode where appropriate.
- Monitor application compatibility.
- Review Defender event logs.
- Deploy progressively.

### 8. Defender recommendations

- Keep security intelligence updated.
- Enable real-time protection.
- Review exclusions.
- Monitor detection events.
- Evaluate ASR rules.
- Use cloud protection where organizational policy permits.
- Integrate endpoint telemetry with centralized monitoring.

---

## Event Logging and Monitoring

Windows Event Logs provide valuable information about system activity, authentication, account management, process execution, and security configuration.

### 1. Important event logs

| Log | Purpose |
|---|---|
| Security | Authentication, account changes, and security auditing |
| System | Operating-system and service events |
| Application | Application-generated events |
| Microsoft-Windows-PowerShell/Operational | PowerShell activity |
| Microsoft-Windows-Windows Defender/Operational | Defender activity |
| Microsoft-Windows-TaskScheduler/Operational | Scheduled task activity |

### 2. List available logs

```powershell
Get-WinEvent -ListLog *
```

### 3. Query recent Security events

```powershell
Get-WinEvent -LogName Security -MaxEvents 50 |
    Select-Object TimeCreated, Id, LevelDisplayName, Message
```

### 4. Important Windows Security Event IDs

| Event ID | Description |
|---|---|
| 4624 | Successful logon |
| 4625 | Failed logon |
| 4634 | Account logoff |
| 4648 | Logon using explicit credentials |
| 4672 | Special privileges assigned to a new logon |
| 4688 | Process creation, when auditing is enabled |
| 4697 | Service installation |
| 4720 | User account created |
| 4722 | User account enabled |
| 4726 | User account deleted |
| 4728 | Member added to a security-enabled global group |
| 4732 | Member added to a security-enabled local group |
| 4740 | User account locked out |
| 4756 | Member added to a security-enabled universal group |
| 4768 | Kerberos authentication ticket requested |
| 4769 | Kerberos service ticket requested |
| 4771 | Kerberos pre-authentication failed |
| 4776 | NTLM credential validation |
| 5140 | Network share accessed |

Some events require specific audit policies to be enabled. Their availability and meaning also depend on the Windows version and environment.

### 5. Enable process creation auditing

From an elevated command prompt:

```powershell
auditpol /set /subcategory:"Process Creation" /success:enable
```

To include command-line information in process creation events, configure the corresponding Group Policy:

```text
Computer Configuration
    Administrative Templates
        System
            Audit Process Creation
                Include command line in process creation events
```

Command-line auditing can expose sensitive information if applications pass secrets as arguments. Protect event logs and review how applications handle credentials.

### 6. Query failed logons

```powershell
$StartTime = (Get-Date).AddHours(-24)

Get-WinEvent -FilterHashtable @{
    LogName = "Security"
    Id = 4625
    StartTime = $StartTime
} -ErrorAction SilentlyContinue |
    Select-Object TimeCreated, Id, Message
```

### 7. Count failed logons

```powershell
$Events = Get-WinEvent -FilterHashtable @{
    LogName = "Security"
    Id = 4625
    StartTime = (Get-Date).AddHours(-24)
} -ErrorAction SilentlyContinue

"Failed logon events: $(@($Events).Count)"
```

### 8. Monitor account creation

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = "Security"
    Id = 4720
    StartTime = (Get-Date).AddDays(-7)
} -ErrorAction SilentlyContinue |
    Select-Object TimeCreated, Message
```

### 9. Monitor PowerShell script activity

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = "Microsoft-Windows-PowerShell/Operational"
    Id = 4104
} -MaxEvents 50 -ErrorAction SilentlyContinue |
    Select-Object TimeCreated, Id, Message
```

Event 4104 is associated with PowerShell Script Block Logging.

### 10. Configure event-log retention

Inspect log configuration:

```powershell
wevtutil gl Security
```

Modify the Security log maximum size, for example:

```powershell
wevtutil sl Security /ms:268435456
```

This sets a 256 MB maximum size. Choose an appropriate size based on event volume and retention requirements.

Avoid enabling overwrite restrictions without considering disk capacity and the possibility of losing future logging when a log fills.

### 11. SIEM integration

Windows event data can be forwarded to centralized monitoring platforms.

Examples:

- Microsoft Sentinel.
- Splunk.
- Elastic Security.
- Wazuh.
- Graylog.

Use Windows Event Forwarding, an approved agent, or a supported collection pipeline.

Centralized collection helps preserve evidence if a local system is compromised or its event logs are altered.

---

## PowerShell Security

PowerShell is a powerful Windows administration interface and an important source of security telemetry.

### 1. Execution policy

Inspect execution policies:

```powershell
Get-ExecutionPolicy -List
```

Set a policy for the current user:

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

Execution policy helps control accidental script execution but is not a security boundary.

Use application control, least privilege, and endpoint monitoring for stronger protection.

### 2. Script Block Logging

Script Block Logging records PowerShell script blocks.

Group Policy location:

```text
Computer Configuration
    Administrative Templates
        Windows Components
            Windows PowerShell
                Turn on PowerShell Script Block Logging
```

### 3. Module Logging

Module Logging records pipeline execution events for configured modules.

Enable it through Group Policy and select the modules that need monitoring.

### 4. Transcription

PowerShell transcription records session activity.

It can be useful for administrative auditing, but transcripts may contain sensitive information.

Store them in an appropriately secured location.

### 5. Check language mode

```powershell
$ExecutionContext.SessionState.LanguageMode
```

Constrained Language Mode can restrict certain PowerShell functionality.

For enterprise enforcement, use supported application-control mechanisms such as Windows Defender Application Control (App Control for Business), rather than relying on changing a session variable.

### 6. Secure scripting practices

- Avoid embedding credentials.
- Validate user-supplied input.
- Avoid executing untrusted strings.
- Use parameter binding.
- Apply least privilege.
- Sign scripts where appropriate.
- Restrict write permissions on script directories.
- Log important administrative activity.
- Test scripts before production deployment.

---

## Registry Security

The Windows Registry contains system, application, user, and security configuration.

Incorrect registry modifications can break applications or operating-system functionality.

Always export the relevant keys and understand the effect of a change before modifying them.

### 1. Registry backup

Example:

```powershell
reg export `
    "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies" `
    "$env:TEMP\PoliciesBackup.reg" /y
```

### 2. Review UAC configuration

```powershell
Get-ItemProperty `
    -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System"
```

### 3. Review startup locations

```powershell
$Paths = @(
    "HKLM:\Software\Microsoft\Windows\CurrentVersion\Run",
    "HKLM:\Software\Microsoft\Windows\CurrentVersion\RunOnce",
    "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run",
    "HKCU:\Software\Microsoft\Windows\CurrentVersion\RunOnce"
)

foreach ($Path in $Paths) {
    if (Test-Path $Path) {
        Write-Output "`n$Path"
        Get-ItemProperty -Path $Path
    }
}
```

### 4. Review Remote Registry service

```powershell
Get-Service -Name RemoteRegistry |
    Select-Object Name, Status, StartType
```

Remote Registry should be enabled only when there is a legitimate administrative requirement.

### 5. SMB security configuration

Inspect SMB server configuration:

```powershell
Get-SmbServerConfiguration |
    Select-Object EnableSMB1Protocol,
                  EnableSMB2Protocol,
                  RequireSecuritySignature
```

Inspect SMB client configuration:

```powershell
Get-SmbClientConfiguration |
    Select-Object EnableSecuritySignature,
                  RequireSecuritySignature
```

Use supported SMB configuration commands and Group Policy to manage protocol security.

### 6. Registry hardening considerations

- Avoid copying registry tweaks from unverified sources.
- Prefer Group Policy or supported PowerShell cmdlets.
- Back up settings before changing them.
- Test changes in a VM.
- Document the original and new values.
- Verify that applications continue functioning.

---

## Network Security

Windows network security includes firewall configuration, secure protocols, DNS configuration, remote access, and network monitoring.

### 1. Inspect network interfaces

```powershell
Get-NetIPConfiguration
```

### 2. Inspect IP addresses

```powershell
Get-NetIPAddress
```

### 3. Inspect DNS configuration

```powershell
Get-DnsClientServerAddress
```

### 4. Inspect network routes

```powershell
Get-NetRoute
```

### 5. Inspect established connections

```powershell
Get-NetTCPConnection -State Established |
    Select-Object LocalAddress,
                  LocalPort,
                  RemoteAddress,
                  RemotePort,
                  OwningProcess
```

### 6. Test a network service

```powershell
Test-NetConnection -ComputerName example.com -Port 443
```

### 7. SMB hardening

Review SMB protocol configuration:

```powershell
Get-SmbServerConfiguration |
    Select-Object EnableSMB1Protocol,
                  EnableSMB2Protocol,
                  RequireSecuritySignature
```

SMB1 is obsolete and should generally be disabled unless a documented legacy requirement exists.

SMB signing and encryption should be configured according to the environment's compatibility and security requirements.

### 8. DNS security

Review DNS client settings:

```powershell
Get-DnsClientServerAddress
```

Consider:

- Approved DNS resolvers.
- DNS filtering.
- Secure DNS policies where supported.
- Monitoring suspicious DNS activity.
- Protecting internal DNS infrastructure.

### 9. LLMNR and NetBIOS

Legacy name-resolution protocols can introduce risks in some Windows networks.

Review whether LLMNR and NetBIOS are required.

For managed environments, configure policies centrally and test application compatibility before disabling legacy protocols.

### 10. IPv6

Do not disable IPv6 simply because it is unfamiliar or unused in an obvious way.

Windows components and enterprise environments may depend on IPv6. Assess actual network requirements and apply appropriate firewall and routing controls instead.

### 11. Remote Desktop security

If Remote Desktop is required:

- Restrict access to approved networks.
- Use Network Level Authentication where supported.
- Require strong authentication.
- Avoid exposing RDP directly to the public internet.
- Monitor logon failures.
- Use VPN or a secure remote-access gateway where appropriate.

Check the current RDP configuration:

```powershell
Get-ItemProperty `
    -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server" `
    -Name fDenyTSConnections
```

---

## Application Security and Control

Application control reduces the ability of unauthorized or untrusted software to execute.

### 1. AppLocker

AppLocker can control which applications, scripts, installers, and other executable content users are permitted to run.

Management tools are available on supported Windows editions.

Review AppLocker policy through:

```text
Local Security Policy
    Application Control Policies
        AppLocker
```

Before enforcing a policy:

1. Build rules for required applications.
2. Test in audit-only mode.
3. Review blocked and allowed activity.
4. Validate application compatibility.
5. Enforce progressively.

### 2. Windows Defender Application Control

Windows Defender Application Control, now part of App Control for Business terminology, provides application-control capabilities.

It can help restrict software to trusted publishers, signed applications, or other approved criteria.

Use Microsoft's current deployment guidance because policy formats and management workflows vary across Windows versions.

### 3. Microsoft Office security

Office applications can be abused through malicious documents and macros.

Recommended controls:

- Block macros from untrusted internet-originated documents where appropriate.
- Use trusted locations sparingly.
- Restrict unnecessary macro execution.
- Keep Office updated.
- Monitor suspicious child processes.
- Use Defender ASR rules where supported.

### 4. Application inventory

Review installed software using uninstall registry entries:

```powershell
$Paths = @(
    "HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*",
    "HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*"
)

Get-ItemProperty $Paths -ErrorAction SilentlyContinue |
    Where-Object DisplayName |
    Select-Object DisplayName, DisplayVersion, Publisher
```

This is a useful inventory source but does not capture every possible application installation.

---

## Compliance and Security Benchmarks

Security baselines provide a repeatable starting point for Windows configuration.

They should be adapted to the environment rather than applied blindly.

### 1. CIS Benchmarks

The Center for Internet Security publishes security benchmarks for Windows and other platforms.

Website:

https://www.cisecurity.org/cis-benchmarks

### 2. Microsoft Security Baselines

Microsoft publishes recommended security configuration baselines for supported Windows versions.

These can be used with Group Policy and Microsoft security-management tooling.

### 3. NIST Cybersecurity Framework

The NIST Cybersecurity Framework organizes cybersecurity risk management into six functions:

| Function | Purpose |
|---|---|
| Govern | Establish cybersecurity strategy and oversight |
| Identify | Understand assets, systems, and risks |
| Protect | Implement safeguards |
| Detect | Identify security events |
| Respond | Manage incidents |
| Recover | Restore operations and improve resilience |

Reference: https://www.nist.gov/cyberframework

### 4. Useful Windows security checks

| Control | Verification |
|---|---|
| Firewall | `Get-NetFirewallProfile` |
| Defender | `Get-MpComputerStatus` |
| BitLocker | `Get-BitLockerVolume` |
| UAC | Inspect system policy values |
| Local administrators | `Get-LocalGroupMember Administrators` |
| Updates | `Get-HotFix` and Windows Update tooling |
| TPM | `Get-Tpm` |
| Secure Boot | `Confirm-SecureBootUEFI` |
| Audit policy | `auditpol /get /category:*` |
| PowerShell logging | Inspect PowerShell Operational events |

### 5. Compliance reporting

A compliance report should include:

- System identity and operating-system version.
- Security control status.
- Policy configuration.
- Evidence collection timestamp.
- Failed checks.
- Exceptions and business justifications.
- Remediation recommendations.

A single PowerShell script cannot establish complete compliance with a security framework. Use the applicable benchmark and its official assessment guidance.

---

## Security Tools and Automation

Windows includes several built-in tools that help administrators and security professionals investigate system behavior.

### 1. Windows Sysinternals

**<Entity value="Sysinternals" category="software" disambig="Microsoft Windows system administration and troubleshooting utilities"/>** provides utilities for investigating processes, files, registry activity, networking, and system behavior.

Useful tools include:

| Tool | Purpose |
|---|---|
| Process Explorer | Detailed process inspection |
| Process Monitor | File, registry, and process activity |
| Autoruns | Startup and autostart inventory |
| TCPView | Network connection inspection |
| PsList | Process information |
| Sigcheck | File signature and version inspection |
| Sysmon | Detailed system activity logging |

Official resources:

https://learn.microsoft.com/en-us/sysinternals/

### 2. Sysmon

System Monitor (Sysmon) is a Windows system service and driver that records detailed system activity to Windows Event Logs.

It can provide visibility into:

- Process creation.
- Network connections.
- File creation.
- Registry changes.
- Driver loading.
- Process access.
- DNS queries, depending on configuration and version.

Sysmon is useful for endpoint detection and incident investigation.

Project:

https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon

### 3. Windows Security Assessment Script

The following script collects a read-only snapshot of several important security settings.

```powershell
function Get-WindowsSecurityAssessment {
    [CmdletBinding()]
    param()

    $UacPath = "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System"

    $Uac = Get-ItemProperty -Path $UacPath -ErrorAction SilentlyContinue

    $Defender = Get-MpComputerStatus -ErrorAction SilentlyContinue

    $Firewall = Get-NetFirewallProfile -ErrorAction SilentlyContinue

    $BitLocker = Get-BitLockerVolume -ErrorAction SilentlyContinue

    $Tpm = Get-Tpm -ErrorAction SilentlyContinue

    [PSCustomObject]@{
        ComputerName = $env:COMPUTERNAME
        Timestamp = (Get-Date).ToUniversalTime().ToString("o")
        UACEnabled = ($Uac.EnableLUA -eq 1)
        DefenderEnabled = $Defender.AntivirusEnabled
        RealTimeProtection = $Defender.RealTimeProtectionEnabled
        FirewallProfilesEnabled = @(
            $Firewall | Where-Object Enabled
        ).Count
        FirewallProfilesDetected = @($Firewall).Count
        BitLockerVolumes = @($BitLocker).Count
        EncryptedVolumes = @(
            $BitLocker |
            Where-Object {
                $_.VolumeStatus -eq "FullyEncrypted"
            }
        ).Count
        TpmPresent = $Tpm.TpmPresent
        TpmReady = $Tpm.TpmReady
    }
}

Get-WindowsSecurityAssessment | Format-List
```

This is an inventory tool, not a complete compliance scanner. Missing permissions, unavailable modules, or unsupported hardware can result in missing values.

### 4. Export the assessment

```powershell
$Report = Get-WindowsSecurityAssessment

$Report |
    ConvertTo-Json -Depth 4 |
    Out-File ".\WindowsSecurityAssessment.json" -Encoding utf8
```

### 5. Automation best practices

- Make scripts repeatable.
- Validate inputs.
- Use structured output.
- Log errors.
- Avoid embedding credentials.
- Use least privilege.
- Test changes before deployment.
- Maintain a rollback plan.
- Version-control scripts.
- Review changes before executing them.

---

## Incident Response

Windows security monitoring is most useful when it supports a defined incident response process.

### 1. Initial response

When a potential compromise is identified:

1. Confirm the alert and identify affected systems.
2. Record the detection time.
3. Preserve relevant logs and evidence.
4. Identify affected users and accounts.
5. Assess the scope of the incident.
6. Follow the organization's containment procedure.
7. Document all actions.

Avoid immediately shutting down or wiping a system unless required by the incident response plan. Doing so may destroy valuable volatile evidence.

### 2. Collect system information

```powershell
New-Item -ItemType Directory `
    -Path "C:\IR" `
    -Force

Get-ComputerInfo |
    Out-File "C:\IR\SystemInfo.txt"
```

### 3. Collect process information

```powershell
Get-CimInstance Win32_Process |
    Select-Object ProcessId,
                  ParentProcessId,
                  Name,
                  ExecutablePath,
                  CommandLine |
    Export-Csv "C:\IR\Processes.csv" -NoTypeInformation
```

### 4. Collect network connections

```powershell
Get-NetTCPConnection |
    Select-Object LocalAddress,
                  LocalPort,
                  RemoteAddress,
                  RemotePort,
                  State,
                  OwningProcess |
    Export-Csv "C:\IR\NetworkConnections.csv" -NoTypeInformation
```

### 5. Export Security events

```powershell
$StartTime = (Get-Date).AddDays(-7)

Get-WinEvent -FilterHashtable @{
    LogName = "Security"
    StartTime = $StartTime
} -ErrorAction SilentlyContinue |
    Export-Csv "C:\IR\SecurityEvents.csv" -NoTypeInformation
```

### 6. Collect registry evidence

Export relevant keys when required by the investigation:

```powershell
reg export HKLM\SOFTWARE "C:\IR\HKLM-Software.reg" /y
```

Registry exports can contain sensitive configuration and account-related information. Protect collected evidence.

### 7. File integrity

Calculate SHA-256 hashes for collected files:

```powershell
Get-FileHash "C:\IR\SecurityEvents.csv" -Algorithm SHA256
```

For an entire collection directory:

```powershell
Get-ChildItem "C:\IR" -File -Recurse |
    Get-FileHash -Algorithm SHA256 |
    Export-Csv "C:\IR\FileHashes.csv" -NoTypeInformation
```

### 8. Containment

Containment should follow the incident response plan and account for business impact.

Possible actions include:

- Isolating an endpoint through approved endpoint-management tooling.
- Disabling compromised accounts.
- Revoking sessions and credentials.
- Blocking confirmed malicious infrastructure.
- Removing unauthorized access.
- Preserving forensic evidence.

Do not deploy a blanket firewall rule or terminate processes indiscriminately. These actions can disrupt business operations and destroy evidence.

### 9. Memory forensics

A process inventory or working-set report is not a memory dump.

If volatile memory acquisition is required, use an approved forensic acquisition tool and preserve the resulting evidence according to chain-of-custody procedures.

### 10. Incident documentation

Record:

- Incident identifier.
- Affected hosts.
- Affected accounts.
- Detection time.
- Relevant event IDs.
- Network indicators.
- Evidence hashes.
- Containment actions.
- Recovery actions.
- Lessons learned.

---

## Recommended Lab Projects

A practical Windows lab makes these concepts easier to understand.

### Beginner

1. Build a Windows 11 virtual machine.
2. Inspect local accounts and groups.
3. Configure UAC and Windows Firewall.
4. Enable BitLocker on a supported test volume.
5. Review Windows Security event logs.
6. Build a basic PowerShell security inventory script.

### Intermediate

1. Configure a Windows security baseline.
2. Build a Group Policy auditing script.
3. Deploy Sysmon in a test VM.
4. Forward Windows events to Wazuh.
5. Create a failed-login monitoring report.
6. Review SMB and remote-access configuration.

### Advanced

1. Build an Active Directory lab with Windows Server.
2. Configure domain security policies.
3. Create a centralized Windows event collection system.
4. Develop an endpoint security assessment framework.
5. Integrate Sysmon with a SIEM.
6. Create an incident response evidence collector.
7. Compare Windows configurations against CIS Benchmarks.

### Suggested lab architecture

```text
                 Host Computer
                      |
                      v
               Hypervisor / VMware
                      |
          +-----------+-----------+
          |                       |
          v                       v
    Windows Server            Windows 11
    Domain Controller         Domain Client
          |                       |
          +-----------+-----------+
                      |
                      v
                Security Monitoring
                      |
                      v
                 Wazuh / SIEM
```

Keep the lab network isolated from production systems and use test accounts and non-sensitive data.

---

## References

### Microsoft documentation

- [Windows Security Documentation](https://learn.microsoft.com/en-us/windows/security/)
- [Windows Security Baselines](https://learn.microsoft.com/en-us/windows/security/operating-system-security/device-management/windows-security-configuration-framework/windows-security-baselines)
- [Microsoft Security Compliance Toolkit](https://learn.microsoft.com/en-us/windows/security/operating-system-security/device-management/windows-security-configuration-framework/security-compliance-toolkit-10)
- [Group Policy Documentation](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-policy/group-policy-overview)
- [User Account Control](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/user-account-control/)
- [Windows Firewall](https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/)
- [BitLocker Documentation](https://learn.microsoft.com/en-us/windows/security/operating-system-security/data-protection/bitlocker/)
- [Microsoft Defender Antivirus](https://learn.microsoft.com/en-us/defender-endpoint/microsoft-defender-antivirus-windows)
- [Windows Event Logging](https://learn.microsoft.com/en-us/windows/win32/eventlog/event-logging)
- [PowerShell Documentation](https://learn.microsoft.com/en-us/powershell/)

### Security benchmarks and frameworks

- [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [MITRE ATT&CK](https://attack.mitre.org/)
- [NIST SP 800-53](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final)
- [NIST SP 800-61](https://csrc.nist.gov/pubs/sp/800/61/r3/final)

### Windows security tools

- [Windows Sysinternals](https://learn.microsoft.com/en-us/sysinternals/)
- [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [Process Monitor](https://learn.microsoft.com/en-us/sysinternals/downloads/procmon)
- [Process Explorer](https://learn.microsoft.com/en-us/sysinternals/downloads/process-explorer)
- [Autoruns](https://learn.microsoft.com/en-us/sysinternals/downloads/autoruns)
- [Wazuh](https://wazuh.com/)

---

## Conclusion

Windows hardening is an ongoing process rather than a one-time configuration task.

A secure Windows environment requires:

- Proper identity and access management.
- Strong authentication.
- Secure system configuration.
- Endpoint protection.
- Network segmentation.
- Encryption.
- Application control.
- Centralized logging.
- Continuous monitoring.
- Tested recovery procedures.

The most important skill is understanding how these controls work together. A firewall cannot compensate for compromised credentials, encryption cannot prevent every form of malware, and logging is not useful if nobody reviews the alerts.

Build a baseline, test your configuration, monitor the results, and document every important change.

**Understand Windows. Harden the endpoint. Monitor the environment. Build security that can be verified.**
