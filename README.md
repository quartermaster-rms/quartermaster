# QUARTERMASTER

**A self-hosted platform for managing shared campus resources** — equipment reservation with hardware interlocks (**dibs**) and inventory with controlled/locked storage (**shiny**) — running on a self-hosted identity and infrastructure stack.

This repository holds the architecture and specifications; it contains no runnable code. Start here, then read the **[implementation guide](IMPLEMENTATION-GUIDE.md)** for the conventions every service follows and each service's `SPEC.md` for its design.

---

## 1. Services and components

### Identity & infrastructure (operated by IT)

| Component | Role |
|---|---|
| **Samba-AD** | The directory and credential store — one username/password (LDAP + Kerberos), and the SMB/Kerberos identity file systems use. |
| **Keycloak** | The identity provider — OIDC single sign-on, MFA, the account lifecycle (email onboarding/reset), and the coarse group roles. Federates to Samba-AD (writable, so password changes reach the directory). Its admin UI is IT-only. |
| **steward_reg** | The registrar — completes each directory identity with the POSIX/Kerberos attributes (uid/gid/home) that file systems need. |
| **steward_truenas** | The NAS provisioner — home datasets and team/shared-folder ACLs on TrueNAS, plus appliance admin for the `sysadmin` group, driven by group membership. |
| **steward_proxmox** | The Proxmox provisioner — users and VM/pool ACLs on Proxmox, plus datacenter admin for the `sysadmin` group, driven by group membership. |

### Domain applications

| Component | Role |
|---|---|
| **dibs** | Equipment — reservation, management, hardware interlocks, and issue reports. **Owns its equipment permissions.** |
| **shiny** | Inventory — stock, receiving/ingest, an immutable transaction ledger, and controlled/locked cabinets. **Owns its inventory permissions.** |

### External systems integrated

- **TrueNAS** — storage; SMB/NFS file shares, and the boot storage for the virtualization layer.
- **Proxmox** — virtualization; hosts the platform's VMs.

---

## 2. Identity and authorization

Authentication is centralized on one credential; authorization is split into a **coarse tier** (central, IT-managed) and a **fine tier** (owned by each app).

- **Authentication** — one credential lives in Samba-AD. Web apps and Proxmox authenticate the browser via **OIDC single sign-on through Keycloak**; TrueNAS file access authenticates via **Kerberos/AD** directly. A service never sees or stores a password.
- **Coarse authorization** — IT manages a small set of Keycloak **group roles**, delivered to apps in the OIDC token. IT does not manage finer detail.
- **Fine authorization** — each domain app owns its detailed permission model internally. dibs decides who may operate which equipment; shiny decides who may check out which inventory. This is invisible to IT.
- **People directory** — because the SSO provider guarantees every user's name and email, each app surfaces a **People** page: a roster, readable by any authenticated user, of the users and admins who use it, each shown with their email and their app-relevant permission level. Admin configuration lives on a separate admin-only Settings surface, where the app's runtime policy (quotas, limits, and the configurable permission model) is set.

### Privilege tiers × department groups

Two independent axes: a **privilege tier** and any number of **department groups**. A person is *one tier* × *zero-or-more department groups*.

| Tier | Who / what | Auth |
|---|---|---|
| **T0 — break-glass** | Local root on TrueNAS and Proxmox; IPMI/console. Hardfault and disaster recovery only. | Local only (no SSO, no directory). Sealed, per-host, rotated after use, audited. |
| **T1 — `sysadmin`** | IT: full admin everywhere (NAS, Proxmox, and god-mode in every app); routine maintenance. | Directory/SSO. **MFA enforced.** |
| **T2 — `admin-<app>`** (`admin-dibs`, `admin-shiny`) | Administers that one app and bootstraps its internal admin. | SSO. |
| **T3 — users** | Authenticated baseline; a home directory on the NAS. | SSO / Kerberos. |

**Department groups `group-*`** (`group-eng`, `group-hr`, …) are the second axis: they grant shared-folder access on the NAS and can gate whole services (e.g. dibs restricted to `group-eng`).

**Rules.** `sysadmin` supersedes every `admin-<app>`. `sysadmin` and `admin-<app>` **bypass department gates** — you cannot lock out your own admins. The department gate is **enforced by each app and configured by that app's admin**, not by IT.

### Why infra admin *is* sysadmin — the trust dependency

Samba-AD runs on a Proxmox VM, and Proxmox boots off TrueNAS. So admin on Proxmox or TrueNAS is effectively control of the whole system — there is **no infra-admin role below `sysadmin`**. And because the identity plane runs on the infrastructure it governs, the **recovery path must not depend on identity**: that is the T0 local break-glass, reachable when identity is down.

The `sysadmin` group's admin on Proxmox and TrueNAS is itself **provisioned by the steward services** (steward_proxmox, steward_truenas) from that group membership — so those provisioners are **sysadmin-minting** and are the most security-audited components on the platform (§3). This deliberately accepts a **privilege inversion** — the identity plane provisions its own governors — contained by that audit, MFA/appliance-2FA on every admin login, tight change-control on `sysadmin` membership, alerting on every admin grant, and T0 as the identity-independent floor.

**Trust layers** (an infrastructure axis, distinct from the privilege tiers): **substrate** (TrueNAS, Proxmox, network, IPMI) → **identity plane** (Samba-AD, Keycloak, the steward services) → **apps** (dibs, shiny). Each layer runs on the one below it.

**Mitigation.** Run at least one Samba-AD node — ideally plus a Keycloak node — on hardware that does **not** boot off the NAS, so identity survives a storage/hypervisor incident. Follow the documented **cold-start order**: NAS → Proxmox → directory → Keycloak → apps.

---

## 3. Communications and security protocols

| Path | Protocol | Notes |
|---|---|---|
| Browser ↔ apps ↔ Keycloak | **OIDC / HTTPS** | Single sign-on; PKCE. |
| Proxmox ↔ Keycloak | **OIDC** realm | Routed through Keycloak so sysadmin logins get MFA. |
| Keycloak ↔ Samba-AD | **LDAPS** | User federation, writable (password write-back to the directory). |
| Clients ↔ TrueNAS | **Kerberos / SMB (encrypted)** | The client network is untrusted, so file traffic is encrypted. |
| steward services ↔ TrueNAS / Proxmox / AD admin APIs | **TLS + privileged token** | IT-managed, admin-scoped, rotated tokens. These services are sysadmin-equivalent; the NAS/Proxmox provisioners actually *mint* the `sysadmin` group's infra admin, so they are audited at the highest level. |
| Interlock nodes ↔ owning app | **Self-securing device port** | TLS (nodes pin the CA) + per-node key + rate limiting; assumes an untrusted network. See the guide. |

**MFA.** Enforced for **all sysadmins**: Keycloak 2FA on every OIDC path (Proxmox admin is routed through OIDC precisely to get it), and TrueNAS's built-in 2FA enabled for its admin. Optional for normal users. Note that direct SMB/Kerberos does not traverse Keycloak, so file-access MFA is enforced at the appliance and workstation, not at Keycloak.

**Break-glass and privilege.** T0 local root is out-of-band by design and remains the identity-independent floor. The steward services and the Keycloak/AD admin credentials are all sysadmin-tier secrets, held only in each deployment's config and rotated. Because steward_truenas and steward_proxmox **provision the `sysadmin` group's admin on TrueNAS and Proxmox**, they are sysadmin-minting authorities, security-audited at the highest level; their API tokens are admin-scoped and must be treated as full break-glass-equivalent secrets.
