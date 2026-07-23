# Lab 06 — AWS to Azure Migration with Azure Migrate

## Objective

Perform a lift-and-shift migration of a Windows Server EC2 instance from AWS to Azure using Azure Migrate (modernized experience) and the Azure Site Recovery replication engine. All infrastructure was deployed via Terraform and torn down after migration.

---

## Architecture

### AWS (Source)
- **EC2 instance**: `ec2-migrate-source-May` — Windows Server 2022, t3.medium, us-east-1a
- **VPC** with public subnet, internet gateway, and security groups
- **Elastic IP** on the internet gateway (EC2 itself had no public IP on its NIC)

### Azure (Target)
| Resource | Name | Purpose |
|---|---|---|
| Resource Group (source infra) | `rg-migrate-source-may` | Appliance VMs |
| Resource Group (target) | `rg-migrate-target-may` | Migrated VM and Azure Migrate artifacts |
| Discovery Appliance VM | `vm-mig-appl-may` | Standard D2s v3 — discovers on-prem/AWS inventory |
| Replication Appliance VM | `vm-mig-repl-may` | Standard D8s v3 — manages replication and data transfer |
| Virtual Network | `vnet-migrate-may` | Target network for migrated VM |
| Subnet | `snet-migrate` | Target subnet |
| NSG | `nsg-migrate-repl-may` | Controls inbound traffic to Replication Appliance |
| Recovery Services Vault | `migrate-project-javen-MigrateVault-353603957` | Modernized RSV used by Azure Migrate |
| Azure Migrate Project | `migrate-project-javen` | Central migration hub |
| Log Analytics Workspace | `law-migrate-may` | Telemetry and monitoring |

---

## Infrastructure Deployment

All infrastructure was deployed with Terraform from the local machine:

```bash
# AWS side
cd aws-side
terraform init
terraform apply

# Azure side
cd azure-side
terraform init
terraform apply
```

> **Note**: The Azure Migrate project itself was created manually in the portal — it cannot be fully managed via Terraform due to Azure Migrate API limitations.

---

## Key Components Explained

### Mobility Service Agent (MSA)
The MSA (InMage Scout VX Agent) is installed on the source machine (EC2). It runs two critical processes:
- **svagents** — parent Windows service
- **s2 / sentinel** — child process that reads disk data and pushes it to the Replication Appliance over port 9443

### cxps (Content Transfer Process Server)
Runs on the Replication Appliance. Listens on port **9443** to receive disk replication data from the MSA's s2 process. Data flows **EC2 → cxps on RA**, not the other way around.

### RCM Proxy (port 443)
The registration/control channel between MSA and RA. Configured via `config.json`. The `NatIpAddress` field in this file tells MSA to use the RA's **public IP** for this channel rather than its private IP.

---

## Migration Process

### Step 1 — Set Up Azure Migrate

1. Created Azure Migrate project in portal (`migrate-project-javen`)
2. Configured the **modernized** Recovery Services Vault (`migrate-project-javen-MigrateVault-353603957`)
   - ⚠️ Used the modernized RSV, not the classic one (`rsv-migrate-may`) — the classic format is incompatible with the modernized Azure Migrate experience

### Step 2 — Configure Replication Appliance

In Azure Migrate → Servers → Replication appliances, registered `vm-mig-repl-may` with the project.

### Step 3 — Manual MSA Installation

Azure Migrate couldn't push the MSA to the EC2 automatically because:
- The EC2's NIC only knows its **private IP** (10.x.x.x)
- The RA has no direct routable path to AWS private IPs
- No VPN or ExpressRoute between AWS and Azure

**Workaround — Mac as relay via RDP drive sharing:**

1. On Replication Appliance (`vm-mig-repl-may`), opened the config manager at `https://localhost:44368`
2. Downloaded `<guid>_config.json`
3. **Manually added `NatIpAddress`** to the config file (missing from every downloaded config):

```json
{
  "SubscriptionId": "538175c3-a8e4-4968-ba70-8f5ff9fc4726",
  "BiosId": "ec2a3641-35e2-59ea-0eec-6affa2a29d4e",
  "Fqdn": "EC2AMAZ-29L8SCG",
  "RcmProxyTransportSettings": {
    "IpAddresses": ["10.1.1.5", "10.0.1.143"],
    "NatIpAddress": "20.127.13.53",
    "Port": 443
  }
}
```

4. Transferred file: RA → Mac Desktop (via RDP `\\tsclient\Desktop\`) → EC2 (via separate RDP session)
5. On EC2, ran registration in `cmd.exe` (not PowerShell):

```cmd
"C:\Program Files (x86)\Microsoft Azure Site Recovery\agent\UnifiedAgentConfigurator.exe" /SourceConfigFilePath "C:\Temp\da540471-3d04-4903-b6aa-94997fa7ba57_config.json" /CSType CSPrime
```

### Step 4 — Open Port 9443

The s2 process on EC2 pushes replication data to cxps on the RA over **TCP 9443**. This port was blocked at two layers:

**Azure NSG** (`nsg-migrate-repl-may`):
- Added inbound rule: Name `allow-9443`, Priority 1030, Port 9443, Protocol TCP, Action Allow

**Windows Firewall** on `vm-mig-repl-may`:
```cmd
netsh advfirewall firewall add rule name="ASR-9443" dir=in action=allow protocol=TCP localport=9443
```

> ⚠️ Both must be open — the NSG and the OS-level firewall are independent. Opening one without the other is not enough.

### Step 5 — Fix Hostname Resolution (Critical)

After opening port 9443, replication still failed. Checking the s2 logs on EC2 revealed:

```
Connecting to: repl-may    → DNS fails
Connecting to: 10.1.1.5    → timeout (private IP, unreachable from AWS)
Connecting to: 10.0.1.143  → connection refused (private IP, unreachable from AWS)
```

The MSA's data transport layer resolves `repl-may` by hostname, not by using `NatIpAddress` (which only affects the RCM registration channel on port 443).

**Fix**: Added a hosts file entry on the EC2 to resolve `repl-may` to the RA's public IP:

File: `C:\Windows\System32\drivers\etc\hosts`
```
20.127.13.53  repl-may
```

Then restarted the MSA service:
```cmd
net stop svagents
net start svagents
```

Status changed: `0% Synchronized` → `Waiting for first recovery point` → **Protected** ✅

### Step 6 — Skip Test Migration

Test migration was attempted twice. Each time:
- A fresh cxps connection attempt was triggered
- The s2 process crashed
- Replication went back to Critical (Error 327001)
- s2_curr log files stayed at 0 KB (s2 not running)

**Decision**: Skip test migration entirely and go straight to Migrate cutover.

### Step 7 — Run Migrate (Failover)

In Azure Migrate → EC2AMAZ-29L8SCG → Migrate:
- Shut down machines before migration: **No** (EC2 would shut itself down)
- Replicate additional changes: **None**

Migration failed twice due to vCPU quota issues:

**Error 28044 — Total regional vCPU limit reached**
- Subscription limit: 10 vCPUs in East US
- In use: vm-mig-appl-may (2) + vm-mig-repl-may (8) = 10
- Target VM needed 2 more → attempted 12, exceeded limit
- Fix: Deallocated `vm-mig-appl-may` (discovery complete, no longer needed) + requested quota increase to 12

**Error 28075 — Standard DSv3 Family quota reached**
- All VMs (both appliances + target) were in the `standardDSv3Family`
- Family-specific quota was also at 10
- Fix: Changed target VM size from `Standard_D2s_v3` → `Standard_D2s_v4` (different quota family, same x64 architecture, 2 vCPUs)

**Migration successful** — all job steps green ✅:
- Prerequisites check for failover ✅
- Start failover ✅
- Starting Azure virtual machine ✅

---

## Errors Reference

| Error ID | Message | Root Cause | Fix |
|---|---|---|---|
| — | MSA push failed | EC2 has no public IP reachable from Azure | Manual install via Mac relay |
| 327001 | Can't connect to Process Server | Port 9443 blocked on NSG + Windows Firewall | Opened TCP 9443 on both |
| 327001 (recurring) | Can't connect to Process Server | s2 resolving `repl-may` to unreachable private IPs | Added `20.127.13.53 repl-may` to EC2 hosts file |
| 327001 (post-test) | Can't connect to Process Server | Test migration triggered fresh cxps connection, crashed s2 | Skipped test migration |
| 28044 | Subscription core limit reached | 10/10 vCPUs used, target needs 2 more | Deallocated discovery appliance + quota increase |
| 28075 | DSv3 Family core limit reached | All VMs in same quota family | Changed target to D2s_v4 (Dsv4 family) |

---

## Commands Reference

### On Replication Appliance (vm-mig-repl-may)

```powershell
# Open port 9443 in Windows Firewall
netsh advfirewall firewall add rule name="ASR-9443" dir=in action=allow protocol=TCP localport=9443

# Restart ASR services
net stop rcmreplicationagent && net start rcmreplicationagent
net stop dra && net start dra
```

### On EC2 (EC2AMAZ-29L8SCG)

```cmd
# Register MSA with Replication Appliance
"C:\Program Files (x86)\Microsoft Azure Site Recovery\agent\UnifiedAgentConfigurator.exe" /SourceConfigFilePath "C:\Temp\<config-guid>_config.json" /CSType CSPrime

# Restart MSA service (triggers s2 restart)
net stop svagents
net start svagents
```

### Azure CLI — RSV Cleanup

```bash
# List ASR fabrics
az site-recovery fabric list --vault-name <vault-name> --resource-group <rg> --output table

# List protection containers
az site-recovery protection-container list --vault-name <vault-name> --resource-group <rg> --fabric-name <fabric> --output table

# List protected items
az site-recovery protected-item list --vault-name <vault-name> --resource-group <rg> --fabric-name <fabric> --protection-container-name <container> --output table

# Delete protected item
az site-recovery protected-item delete --vault-name <vault-name> --resource-group <rg> --fabric-name <fabric> --protection-container-name <container> --replicated-protected-item-name <item-name> --yes

# Force delete RSV
az backup vault delete --name <vault-name> --resource-group <rg> --force
```

---

## Lessons Learned

**1. The MSA data transport layer resolves hostnames independently of NatIpAddress.**
`NatIpAddress` in config.json only affects the RCM registration channel (port 443). The s2/cxps data transport layer (port 9443) resolves the RA hostname (`repl-may`) via DNS or hosts file independently. When DNS fails, s2 falls back to private IPs — which are unreachable from AWS. The fix is a hosts file entry on the source machine, not a config change.

**2. The NSG and Windows Firewall are independent — both must be open.**
It's easy to assume that opening the NSG is sufficient. The OS-level Windows Firewall on the RA is a completely separate barrier. Port 9443 must be open on both.

**3. Test migration can destabilize an active replication.**
Each test migration attempt creates a fresh cxps connection which can disrupt the replication state and crash the s2 process. If replication is fragile (e.g., relying on a hosts file workaround for public IP routing), skip test migration and go straight to Migrate.

**4. Azure vCPU quota has two levels: total regional and per VM family.**
Requesting a total regional vCPU increase is not enough — each VM family also has its own quota. If all your VMs are in the same family (e.g., Dsv3), you'll hit the family limit even after increasing the regional total. Changing to a different VM family (Dsv4) sidesteps this entirely.

**5. Azure Migrate creates resource locks on its storage accounts.**
When tearing down, Azure Migrate places a lock on the replication storage account. Terraform destroy will fail with a ScopeLocked error. Remove the lock manually in the portal before running destroy.

**6. The modernized Azure Migrate experience requires its own RSV.**
The classic RSV (`rsv-migrate-may`) is incompatible with the modernized Azure Migrate experience. The correct vault is the one auto-created by Azure Migrate (`migrate-project-javen-MigrateVault-353603957`).

---

## Teardown

### Azure

```bash
# Remove protected items from RSV first (see Commands Reference above)
# Then destroy Azure infrastructure
cd azure-side
terraform destroy
```

Manual cleanup required for Azure Migrate artifacts not tracked by Terraform:
- Remove SAS lock on seed disk (`Disk Export → Cancel export`)
- Remove RSV resource lock (`Settings → Locks`)
- Delete RSV protected items via CLI
- Force delete RSV via CLI

### AWS

```bash
cd aws-side
terraform destroy
```

If EC2 persists after destroy: AWS Console → EC2 → Instance State → Terminate instance.

---

## Resources

- [Azure Migrate documentation](https://learn.microsoft.com/en-us/azure/migrate/)
- [Mobility Service Agent installation](https://learn.microsoft.com/en-us/azure/site-recovery/vmware-azure-install-mobility-service)
- [Troubleshoot ASR replication issues](https://learn.microsoft.com/en-us/azure/site-recovery/vmware-azure-troubleshoot-replication)
- [Azure VM vCPU quota](https://learn.microsoft.com/en-us/azure/quotas/per-vm-quota-requests)
