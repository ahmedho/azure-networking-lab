# Azure Networking Troubleshooting Cheat Sheet

A symptom-first reference compiled from real issues encountered while building this lab (Steps 1–10). Organized by what you'd actually observe, not by which Azure service is "responsible" — troubleshooting starts with a symptom, not a hypothesis.

---

## "My VM/resource can't reach the internet"

**Symptom:** `apt-get`, `curl`, or any outbound call times out from inside a VM.

**Likely causes, in order of likelihood:**
1. **No default outbound access** (Azure platform change, March 2026+) — new VNets no longer get implicit internet access. Check: does the subnet have a NAT Gateway, or does the NIC have a Public IP?
```bash
   az network vnet subnet show --resource-group <rg> --vnet-name <vnet> --name <subnet> --query "natGateway"
```
2. **NSG blocking outbound** — check effective rules for outbound deny:
```bash
   az network nic list-effective-nsg --resource-group <rg> --name <nic> --output json
```
3. **UDR routing traffic somewhere with no actual path out** (e.g. forced tunneling to a non-existent NVA)

**Fix:** Attach a NAT Gateway to the subnet (preferred, scalable) or a Public IP directly to the NIC (temporary/single-resource only).

---

## "NSG rule seems right but traffic is still blocked/allowed unexpectedly"

**Symptom:** You configured a rule, but behavior doesn't match what you expect.

**Checklist:**
1. **Confirm the NSG is actually attached** — don't assume, verify:
```bash
   az network vnet subnet show --resource-group <rg> --vnet-name <vnet> --name <subnet> --query "networkSecurityGroup"
   az network nic show --resource-group <rg> --name <nic> --query "networkSecurityGroup"
```
   NSGs can be silently detached (e.g. by a `--remove` command run against the wrong subnet) without any error at the time.
2. **Check priority order** — lower number evaluates first, first match wins. A broad Deny at priority 200 will override anything below it, even a specific Allow at priority 300.
3. **Check for the same NSG attached to multiple subnets unintentionally** — rules meant for one subnet can silently apply to another.
4. **Remember default rules are immutable** — `AllowAzureLoadBalancerInBound` (65001) etc. cannot be edited/reprioritized; conflicts must be resolved with new custom rules at a lower priority number.
5. **Use IP Flow Verify to get a definitive answer** rather than reasoning about it manually:
```bash
   az network watcher test-ip-flow --resource-group <rg> --vm <vm> --nic <nic> --direction Inbound --protocol TCP --local <ip>:<port> --remote <ip>:<port>
```

---

## "ASG reference in an NSG rule fails to create"

**Error:** `Rules using application security groups may only be applied when the ASGs are associated with network interfaces on the same virtual network.`

**Cause:** An ASG with zero NIC members cannot be referenced in a rule — Azure rejects it at creation time.

**Fix:** Associate at least one NIC's IP configuration with the ASG first:
```bash
az network nic ip-config update --resource-group <rg> --nic-name <nic> --name <ipconfig-name> --application-security-groups <asg-name>
```
Find the IP config name first if unsure — it's not always `ipconfig1`:
```bash
az network nic ip-config list --resource-group <rg> --nic-name <nic> --output table
```

---

## "Deny-All NSG rule breaks Load Balancer / Application Gateway health checks"

**Symptom:** Warning on rule creation: `This rule denies traffic from AzureLoadBalancer and may affect virtual machine connectivity.`

**Cause:** A custom Deny-All rule sits above the immutable default `AllowAzureLoadBalancerInBound` (priority 65001), silently blocking health probes.

**Fix:** Add an explicit Allow rule for the `AzureLoadBalancer` service tag at a priority lower than your Deny-All rule:
```bash
az network nsg rule create --resource-group <rg> --nsg-name <nsg> --name Allow-AzureLB-HealthProbe --priority 120 --direction Inbound --access Allow --protocol '*' --source-address-prefixes AzureLoadBalancer --destination-port-ranges '*'
```

---

## "Backend health shows Unhealthy" (Load Balancer or Application Gateway)

**Diagnosis path, in order:**
1. **Is anything actually listening?** SSH/Serial Console into the backend, test locally:
```bash
   curl -I http://localhost:<port>
   sudo ss -tlnp | grep :<port>
```
2. **Host-level firewall?**
```bash
   sudo ufw status
```
3. **NSG blocking the port?** Check effective rules (see NSG section above).
4. **Subnet has no outbound access, so the service never started?** (e.g. package install failed) — check `/var/log/cloud-init-output.log` if using cloud-init.
5. **Read the exact probe error** — it tells you which category:
   - *Timeout* → nothing is listening (compute/app issue)
   - *Cannot connect... check NSG/UDR/Firewall* → something is listening, but network layer is blocking it
   - *403/502 from backend* → app-level rejection, not a network issue

---

## "Effective routes / effective NSG rules command fails or shows nothing useful"

**Error:** `Azure cannot calculate or generate the active, effective route table for a network interface (NIC) unless the virtual machine it is attached to is currently running.`

**Cause:** Several Network Watcher and diagnostic commands (`show-effective-route-table`, `test-ip-flow`, `show-next-hop`) require a NIC to be attached to a **running VM** — a standalone NIC isn't enough, even though other operations (NSG association, ASG membership) work fine on standalone NICs.

**Fix:** Temporarily attach a minimal VM (e.g. `Standard_B2ats_v2`) to the NIC, run the diagnostic, then delete the VM. The NIC and its configuration persist after VM deletion.

---

## "az vm delete doesn't fully clean up"

**Symptom:** Resource group still shows charges/resources after deleting a VM.

**Cause:** `az vm delete` removes the compute resource but **not** its attached NIC or OS disk by default.

**Fix — always check for orphans after deleting a VM:**
```bash
az disk list --resource-group <rg> --query "[].name" --output table
az network nic list --resource-group <rg> --output table
```
Delete anything left over:
```bash
az disk delete --resource-group <rg> --name <disk-name> --yes
az network nic delete --resource-group <rg> --name <nic-name>
```

---

## "Azure CLI: can't clear an assigned resource with an empty string"

**Error:** `InvalidResourceReference` when passing `--<property> ""` to remove an association (e.g. detaching an NSG from a subnet).

**Cause:** Azure CLI interprets an empty string as an attempt to resolve a resource named `""`, not as "clear this field."

**Fix:** Use the generic `--remove <propertyName>` argument instead:
```bash
az network vnet subnet update --resource-group <rg> --vnet-name <vnet> --name <subnet> --remove networkSecurityGroup
```

---

## "Portal shows a resource as created, but CLI can't find it"

**Symptom:** A Portal wizard step appeared to complete, but a subsequent CLI command referencing that resource fails with "not found."

**Cause:** Some Portal blades (e.g. Standard Load Balancer's backend pool picker) have UI limitations that silently prevent a save from actually completing — particularly when the picker only supports certain resource types (e.g. VM-attached NICs) and your actual resources don't qualify.

**Fix:** Don't trust Portal UI state alone — verify with a CLI `list`/`show` command before building on top of it. If missing, create the resource explicitly via CLI, which usually has fewer restrictions than the Portal wizard.

---

## "Public IP creation fails both as new and as existing"

**Symptom:** "Use existing" shows no options; "Create new" with your intended name says it already exists.

**Cause:** A Public IP with that name exists but has an incompatible configuration (e.g. wrong SKU or missing zone-redundancy) for what the current wizard needs — so it doesn't qualify as a valid "existing" option, yet still blocks the name.

**Fix:** Delete the conflicting Public IP and recreate it with the correct SKU/settings explicitly via CLI before retrying.

---

## Platform changes to remember (as of mid-2026)

| Retired / Changed | Replacement | Effective |
|---|---|---|
| Basic Load Balancer | Standard SKU (mandatory) | Sept 30, 2025 |
| Application Gateway V1 | Standard_v2 / WAF_v2 | April 28, 2026 |
| Default outbound internet access for new VNets | NAT Gateway / explicit Public IP required | March 31, 2026 |
| NSG Flow Logs (new creation) | Virtual Network Flow Logs | Blocked since June 30, 2025 |
| Basic Public IP (for VPN Gateway) | Standard SKU Public IP | June 30, 2026 |

Always verify current SKU/feature status against Microsoft Learn before planning a deployment — Azure's networking surface changes frequently, and outdated tutorials/AI training data can reference retired options.