# Infrastructure Migration Project Plan
## ABB → 11:11 Systems | South Australia to Sydney

**Version:** 0.1 Draft  
**Status:** In progress  
**Owner:** [Your name / team]  
**Date:** [Date]

---

## 1. Background & Context

Our current hosting provider, ABB, has been acquired by 11:11 Systems. As part of this acquisition, our infrastructure is being migrated from a South Australian datacentre to a new 11:11 facility in Sydney. The migration of virtual machines and network infrastructure is being coordinated jointly by ABB and 11:11.

While the providers are managing the logistics of the migration itself, we have our own parallel objectives:

- Validate application functionality and performance in the new environment as early as possible
- Understand and document the end-state network topology before production cutover
- Confirm our Disaster Recovery posture and close identified gaps
- Understand 11:11's management platform and its operational capabilities

---

## 2. Immediate Concerns (Action Required)

### 2.1 Firewall & DR Configuration — URGENT

We believe our current Fortigate firewall is deployed in a replica configuration rather than an active/passive failover pair. This configuration was understood to have been established to support a Disaster Recovery test scenario. However, the current state has **not been formally confirmed**.

**Required actions:**

| Action | Owner | Priority |
|---|---|---|
| Confirm current Fortigate HA/DR configuration with ABB | [Owner] | Critical |
| Clarify whether DR failover is currently functional or intentionally disabled | [Owner] | Critical |
| Document compute sitting behind the DR firewall | [Owner] | High |
| Determine whether this configuration needs to be remediated before migration | [Owner] | High |

### 2.2 Network Migration Topology — URGENT

We have very limited visibility into how the network migration will be executed. This is a critical gap. We do not currently know:

- How and when the Fortigate gateway will be migrated from SA to Sydney
- How Layer 2 VLANs will be handled during and after the migration
- Whether VLANs will be stretched between the current and new datacentres during transition
- What the intended end-state Layer 3 topology looks like

**An urgent communication to ABB has been drafted — see Section 7.**

---

## 3. Migration Objectives

### 3.1 Primary Objective — Non-Production Environment in Sydney

We need to establish a working instance of our non-production environment in the new 11:11 Sydney datacentre as soon as possible. This serves two purposes:

1. Validate application behaviour and performance in the new infrastructure
2. Establish a topology that closely reflects the intended production end-state

#### Application Latency Requirements

Our core line-of-business application is highly latency-sensitive. It communicates with a Microsoft SQL Server backend via frequent CRUD operations (Create, Read, Update, Delete). Even modest increases in round-trip latency between the application tier and database tier will have a measurable impact on user experience.

| Path | Latency Tolerance | Notes |
|---|---|---|
| SA workstation → Sydney SQL server | Low — must be measured and validated | Primary concern; WAN latency introduced by relocation |
| Sydney app server → Sydney SQL server | Very low — must be co-located in adjacent VLANs | Intra-DC path; gateway must be hosted in 11:11 infra |
| Sydney app server → Sydney DMZ | Standard | Standard east-west traffic |

---

## 4. Requested Network Configuration

### 4.1 VLANs — New Sydney Datacentre

We are requesting the creation of the following VLANs in the new 11:11 Sydney datacentre. These should be provisioned regardless of whether they are delivered via our existing VMware Cloud Director instance or through 11:11's own management platform.

| VLAN Name | Purpose | Subnet | Status |
|---|---|---|---|
| `vnet-np-sql` | Internal non-production database servers | TBD — confirm once provisioning is possible | Requested |
| `vnet-np-app` | Internal non-production application servers | TBD — confirm once provisioning is possible | Requested |
| `vnet-np-dmz` | Non-production DMZ servers | TBD — confirm once provisioning is possible | Requested |

All gateways for the above VLANs are to be hosted within 11:11 infrastructure.

### 4.2 VLAN Connectivity

Once VLANs are provisioned, we require:

- VLANs trunked to the current VMware Cloud Director instance (preferred) **or** to compatible infrastructure in the 11:11 Sydney datacentre, whichever is more expedient
- The `vnet-np-app` and `vnet-np-sql` VLANs to be as close together as possible (same datacentre fabric; low-latency east-west path)
- Confirmation of routing between `vnet-np-app` → `vnet-np-sql` and `vnet-np-app` → `vnet-np-dmz`

### 4.3 Fortigate Gateway Migration — Key Questions

| Question | Notes |
|---|---|
| How will the Fortigate be migrated from SA to Sydney? | Phased cutover? Parallel run? |
| Where will Layer 2 VLANs be hosted during transition? | Stretched VLANs? New segments? |
| What is the sequence of steps for gateway migration? | See proposed sequence below |
| What latency should we expect from the relocated gateway to 11:11 compute? | Requires testing once VMs are in Sydney |

**Proposed gateway migration sequence (to be confirmed with ABB/11:11):**

1. Confirm current Fortigate configuration and export baseline
2. Provision new VLANs in 11:11 Sydney DC
3. Migrate non-production VMs to Sydney VLANs (via vMotion-equivalent)
4. Re-IP servers via console access once on new gateways
5. Validate application connectivity and measure latency
6. Confirm production migration sequence based on findings
7. Migrate Fortigate (or replace with 11:11-hosted gateway) at agreed cutover window

---

## 5. VM Migration — Non-Production Testing

### 5.1 Scope

We intend to migrate a small number of non-production VMs to the new Sydney VLANs to validate the end-state configuration. The servers to be migrated are:

- **1× non-production MSSQL database server** → `vnet-np-sql`
- **1× non-production application server** → `vnet-np-app`

After migration, we will use console access to re-IP the servers to match the new VLAN gateways.

### 5.2 Migration Timeline (Draft)

| Phase | Activity | Duration (est.) | Dependency |
|---|---|---|---|
| **1** | Confirm VLAN provisioning with 11:11 | 1–2 days | 11:11 confirmation |
| **2** | Access management portal; confirm compute location | 1 day | Portal credentials provided |
| **3** | Initiate vMotion-equivalent of DB + App VMs to Sydney | 2–4 hours | VLANs trunked |
| **4** | Re-IP servers via console to new gateway subnets | 1–2 hours | VLAN subnets confirmed |
| **5** | Validate app → DB connectivity (intra-DC) | 2–4 hours | VMs re-IPd |
| **6** | Latency test: SA workstation → Sydney DB | 1 day | Step 5 complete |
| **7** | Document results and confirm production migration plan | 1 day | Steps 5–6 complete |

---

## 6. Disaster Recovery & Backup — Information Required

### 6.1 Disaster Recovery

| Question | Why It Matters |
|---|---|
| What is the RPO and RTO offered by 11:11's DR platform? | Determines our recovery capability in a failure event |
| What triggers a failover to the DR site? | Manual vs automatic; approval required? |
| At what frequency are replica copies taken? | Defines data loss exposure |
| Will VLANs be stretched from the Sydney DC to the DR site? | Determines whether IPs are preserved in a failover |
| Is the DR platform Zerto Cloud Manager? Please confirm. | We need to understand the tooling |
| Can we observe a DR test or failover simulation? | Hands-on understanding of the process |

### 6.2 Backup

| Question | Why It Matters |
|---|---|
| What is the backup platform and retention policy? | Operational compliance |
| Can backups be triggered on demand or only on schedule? | Flexibility for pre-change snapshots |
| Can we tag resources to control backup inclusion/exclusion? | Granular control over what is protected |
| How do we restore individual VMs or volumes? | Recovery process must be documented |

### 6.3 Management Platform & Orchestration

| Question | Why It Matters |
|---|---|
| What is the management portal for the new infrastructure? | We need to understand our operational interface |
| When can our instance be provisioned and credentials issued? | Unblocks all testing activity |
| Does the platform support SAML or OIDC integration? | Required for SSO integration with our IdP |
| Can resources be tagged for backup and billing management? | Operational and financial governance |
| What are the surge/burst compute capabilities? | Understanding capacity ceiling for peak workloads |

---

## 7. Draft Urgent Communication to ABB

> **Subject: Urgent — Network Topology & Migration Sequencing Clarification Required**
>
> Hi [ABB contact],
>
> With the migration to 11:11 Systems now underway, we need to urgently resolve a number of open questions around network topology and migration sequencing. Without this clarity, we are unable to plan our application validation activities or ensure continuity of service.
>
> Specifically, we need the following addressed as a priority:
>
> 1. **Network migration plan** — We do not currently understand how the migration of our Layer 2 VLANs and Fortigate gateway from South Australia to Sydney is intended to occur. Can you provide a high-level migration sequence and confirm how VLANs will be handled during the transition?
>
> 2. **Firewall DR configuration** — We understand our Fortigate may currently be deployed in a replica configuration rather than an active failover pair. Can you confirm the current state and advise whether this was intentional (e.g. for DR testing) and whether it needs to be remediated before migration?
>
> 3. **VLAN provisioning in Sydney** — We would like to have the following VLANs provisioned in the new 11:11 Sydney datacentre as soon as possible to begin non-production validation:
>    - `vnet-np-sql` (non-prod database servers)
>    - `vnet-np-app` (non-prod application servers)
>    - `vnet-np-dmz` (non-prod DMZ)
>
> 4. **Management portal access** — When can we expect to receive access to the 11:11 management portal? We need this to begin provisioning and testing work independently.
>
> We have a latency-critical application that relies on low-latency connectivity between our application and database tiers. It is important that we are able to validate this in the new environment before any production migration occurs.
>
> Could you please arrange a call this week to work through the above? We are available at short notice given the urgency.
>
> Kind regards,  
> [Your name]

---

## 8. Questions Summary

| # | Question | Directed To | Priority |
|---|---|---|---|
| 1 | What is the current Fortigate HA/DR configuration? | ABB | Critical |
| 2 | How will VLANs be handled during the gateway migration? | ABB / 11:11 | Critical |
| 3 | When can we get VLAN provisioning in Sydney? | 11:11 | Critical |
| 4 | When will management portal access be available? | 11:11 | High |
| 5 | Is the DR platform Zerto Cloud Manager? | 11:11 | High |
| 6 | What is the RPO/RTO of the 11:11 DR platform? | 11:11 | High |
| 7 | Does the management platform support SAML/OIDC? | 11:11 | Medium |
| 8 | Can resources be tagged for backup management? | 11:11 | Medium |
| 9 | What are the burst/surge compute capabilities? | 11:11 | Low |

---


