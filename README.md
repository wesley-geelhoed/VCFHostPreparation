# HostPrep.ps1

A PowerShell script that automates ESXi host preparation for commissioning into **VMware Cloud Foundation 9 / SDDC Manager**.

Instead of manually preparing a dozen hosts one by one, drop a text file with your host FQDNs, run the script, and get a ready-to-use HTML commissioning report with SHA-256 thumbprints when it's done.

![HostPrep console output](https://www.hollebollevsan.nl/wp-content/uploads/2026/03/HostPrep-Console-Screenshot.jpg)

---

## What it does

For each host in your list, the script runs through the following steps in order:

1. **DNS validation** — forward (A record) and reverse (PTR) lookup, flags mismatches before anything else runs
2. **Connect** — connects directly to the ESXi host using the root account via PowerCLI
3. **NTP** — verifies the required NTP servers are configured and that `ntpd` is running and set to start automatically
4. **Advanced Settings** — sets `Config.HostAgent.ssl.keyStore.allowSelfSigned = true`, required by SDDC Manager
5. **Optional Advanced Settings** — applies any extra settings you have enabled in the `$OptionalAdvancedSettings` block at the top of the script
6. **Certificate regeneration** — reads the TLS certificate from port 443 and checks whether the CN matches the host FQDN. If not, it temporarily enables SSH, runs `/sbin/generate-certificates` via Posh-SSH, disables SSH again, then reboots the host and waits for it to come back online before continuing
7. **Password reset** *(optional)* — resets the root account password to a new VCF 9 compliant value. Always runs last so the existing credential remains valid throughout

After all hosts are processed, a **colourised summary table** is printed and a **self-contained HTML report** is saved next to the script.

![HostPrep HTML commissioning report](https://www.hollebollevsan.nl/wp-content/uploads/2026/03/HostPrep-Report-Screenshot.jpg)

---

## Requirements

| Requirement | Notes |
|---|---|
| PowerShell 5.1+ | Included with Windows 10/Server 2016 and later |
| VMware PowerCLI | `Install-Module -Name VMware.PowerCLI -Scope CurrentUser` |
| Posh-SSH | Optional — needed for automated cert regeneration. `Install-Module -Name Posh-SSH -Scope CurrentUser` |

**One-time PowerCLI setup** (run once per user account, then never again):

```powershell
Set-PowerCLIConfiguration -Scope User -ParticipateInCEIP $false -Confirm:$false
```

This permanently suppresses the VMware CEIP warning. The script handles the rest automatically on every subsequent run.

---

## Usage

Basic run — all prompts are interactive:

```powershell
.\HostPrep.ps1
```

With custom NTP servers:

```powershell
.\HostPrep.ps1 -NtpServers "ntp1.example.com","ntp2.example.com"
```

Simulate all steps without making any changes:

```powershell
.\HostPrep.ps1 -DryRun
```

Collect thumbprints and generate the HTML report without making any changes:

```powershell
.\HostPrep.ps1 -WhatIfReport
```

### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `-NtpServers` | `string[]` | `pool.ntp.org` | One or more NTP server addresses |
| `-DryRun` | `switch` | — | Simulate all steps, no changes made |
| `-WhatIfReport` | `switch` | — | Connect and collect thumbprints only, generate report |
| `-LogPath` | `string` | Desktop | Path for the transcript log file |
| `-ReportPath` | `string` | Next to script | Path for the HTML report file |

### Host list file

The script prompts for the path to a plain text file with one FQDN per line. Lines starting with `#` are treated as comments and ignored.

```
esxi01.vcf.lab
esxi02.vcf.lab
# esxi03.vcf.lab  (skipped)
esxi04.vcf.lab
```

---

## Optional Advanced Settings

Near the top of the script, just below the version metadata, there is an `$OptionalAdvancedSettings` block. Each entry is disabled by default — set `Enabled = $true` to apply it on every host.

| Setting | Description | Value type |
|---|---|---|
| `Config.HostAgent.plugins.hostsvc.esxAdminsGroup` | AD group whose members get full ESXi admin access. Change the value to match your AD group name. | string |
| `LSOM.lsomEnableRebuildOnLSE` | Enables automatic vSAN rebuild when a device is flagged as Latency Sensitive Equipment | integer (1/0) |
| `DataMover.HardwareAcceleratedMove` | Enables SSD TRIM — ESXi issues UNMAP commands to compatible SSDs | integer (1/0) |
| `DataMover.HardwareAcceleratedInit` | Enables SSD TRIM — ESXi issues UNMAP commands to compatible SSDs | integer (1/0) |

> **Note:** Re-running the script with a different `esxAdminsGroup` value will overwrite the setting on all hosts. Verify the group name before enabling across a full deployment.

---

## HTML commissioning report

The report is saved next to the script as `HostPrep_<timestamp>_Report.html`. It is designed to sit open in a browser alongside the SDDC Manager commissioning wizard.

For each host it shows:

- **SSL thumbprint** in `SHA256:<base64>` format (exactly as the SDDC Manager UI expects) with a one-click copy button
- **Certificate expiry** — highlighted amber within 90 days, red within 30
- **DNS status** — forward and reverse lookup result
- **Per-step status** — cert regen, reboot, NTP, advanced settings, optional settings, password reset
- **Overall pass/fail indicator**

If a certificate was regenerated during the run, the thumbprint shown reflects the new certificate read after the host came back online.

---

## DNS validation

Before connecting to each host, the script validates DNS:

| Result | Meaning |
|---|---|
| `OK` | A record resolves and PTR matches the FQDN |
| `WARN: PTR mismatch` | A record resolves but PTR points to a different name |
| `WARN: No PTR record` | A record resolves but no PTR exists |
| `FAILED` | Forward lookup failed entirely |

DNS issues are flagged but do not block the remaining preparation steps. Fix any warnings before commissioning — SDDC Manager requires correct forward and reverse DNS.

---

## Summary table

After all hosts are processed a colourised per-host table is printed to the console:

| Colour | Meaning |
|---|---|
| 🟢 Green | OK |
| 🔴 Red | FAILED or Timeout |
| ⬜ Dark gray | Skipped |
| 🟡 Yellow | Manual action required, Partial, or Warning |

---

## VCF 9 password requirements

If a password reset is requested, the new password is validated against VCF 9 rules before any host is touched:

- 15 to 40 characters
- At least 1 lowercase letter
- At least 1 uppercase letter — not as the first character
- At least 1 digit — not as the last character
- At least 1 special character from: `@ ! # $ % ? ^`
- Only letters, digits, and `@ ! # $ % ? ^` permitted
- At least 3 of the 4 character classes must be present

---

## Reboot timeout handling

If a host does not come back online after a certificate-triggered reboot (hardware fault, boot loop, etc.), the script:

- Sets `Rebooted = Timeout` in the summary and report
- Prints a prominent red banner immediately in the console
- Skips the disconnect so no extra noise is generated
- Records the full error in the `Error` column

The remaining hosts in the list continue to be processed.

---

## Blog post

Full write-up with background and screenshots:  
[Automating ESXi Host Preparation for VCF 9 with PowerShell – HolleBollevSAN](https://www.hollebollevsan.nl/automating-esxi-host-preparation-for-vcf-9-with-powershell/)

---

## Author

Paul van Dieen — [hollebollevsan.nl](https://www.hollebollevsan.nl)
