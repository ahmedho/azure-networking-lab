## 🗂️ Structure
Each step documents one networking concept: design decisions, Portal + CLI implementation, verification, and key learnings.

| Step | Topic | Status |
|---|---|---|
| 1 | VNet & Subnet Design | ✅ |
| 2 | VNet Peering | ✅ |
| 3 | Network Security Groups & ASGs | ✅ |
| 4 | Route Tables & UDRs | ✅ |
| 5 | Azure DNS | ✅ |
| 6 | Load Balancer | ✅ |
| 7 | Application Gateway | ✅ |
| 8 | VPN Gateway | ✅ |
| 9 | Network Watcher & Diagnostics | ✅ |
| 10 | Azure Bastion & NAT Gateway | ✅ |
| 11 | Final Review & Architecture Summary | ✅ |

## 🎓 What This Lab Demonstrates
- End-to-end hub-and-spoke network architecture
- Layer 4 and Layer 7 load balancing (Standard Load Balancer, Application Gateway)
- Secure hybrid connectivity (VPN Gateway, Point-to-Site)
- Defense-in-depth security (NSGs, ASGs, private DNS, no public IPs on workload VMs)
- Native Azure diagnostics and troubleshooting (Network Watcher, effective routes, effective NSG rules)
- Real-world platform changes navigated live: Basic Load Balancer retirement, Application Gateway V1 retirement, default outbound access retirement, NSG Flow Logs retirement
- Strict cost discipline: deploy-verify-delete for every billable resource, resulting in a lab that cost only a few euros total in Azure Student credit

See [`docs/`](./docs) for full step-by-step documentation, including real errors encountered and resolved along the way.

## 🩹 Troubleshooting Cheat Sheet
See [`docs/troubleshooting-cheatsheet.md`](./docs/troubleshooting-cheatsheet.md) for a consolidated reference of every diagnostic issue hit during this lab, organized by symptom.