# Nautobot Golden Config Setup Guide

This guide covers the core setup for Golden Config with project-specific details removed.

Quick setup flow:
1. Define the device scope
2. Point Golden Config to your repos
3. Make sure templates and data can render intended configs
4. Then move on to compliance/remediation

For compliance and remediation details, use:
- [CONFIGURING_COMPLIANCE.md](./CONFIGURING_COMPLIANCE.md)

---

## What Golden Config needs to work

At a high level, Golden Config needs:
1. A device scope - set of devices (defined by a Dynamic Group) that settings apply to.
2. Repositories - one for each function is recommended to reduce blast radius for rollbacks/reverts:
   - Backups
   - Intended
   - Templates
   - Config Context
3. A GraphQL SoT query - Nautobot query for objects/attributes related to the device.
4. Golden Config settings that tie all of that together

---

## Step 1) Register Git repositories

> **Navigation Path:** `Extensibility > Git Repositories`

Register repositories for:

| Repository Purpose | `Provides` Value |
|---|---|
| Backup configs repository | `backup configs` |
| Intended configs repository | `intended configs` |
| Jinja templates repository | `jinja templates` |
| Config context repository | `config contexts` |

Golden Config discovers repository usage from the `Provides` values.

---

## Step 2) Define device scope (Dynamic Group)

Golden Config jobs run against a set of devices created by the Dynamic Group.

1. **Navigation Path:** `Organization > Dynamic Groups > Add`
2. Create a group name (for example: `Golden Config Managed Devices`)
3. Add filters (tenant, location, role, tag, status, etc.). For example, if tenant is selected, settings will apply to all devices that belong to that tenant.
4. Save, then click **Refresh Members**

Tip: Start with a small pilot scope first.

> **Note:** Dynamic Group membership is not automatically recalculated every time device data changes.  
> After changing group filters, device attributes, tags, role/location assignments, or tenant associations, click **Refresh Members** again before running Golden Config jobs.

---

## Step 3) Create the SoT Aggregation GraphQL query

Golden Config uses a GraphQL query to gather device attributes used by templates.

> **Navigation Path:** `Extensibility > GraphQL Queries > Add`

Create a query like:

Note: the query must start with the first two lines of the example below.

```graphql
query ($device_id: ID!) {
  device(id: $device_id) {
    name
    primary_ip4 {
      address
    }
    platform {
      network_driver
    }
    role {
      name
    }
    location {
      name
    }
    tenant {
      name
    }
    interfaces {
      name
      mgmt_only
    }
    config_context
  }
}
```

If templates reference additional fields, add them here.

---

## Step 4) Configure Golden Config Settings

> **Navigation Path:** `Golden Config > Settings > Add`

Configure:

| Field | Example |
|---|---|
| **Name** | `Default Golden Config Settings` |
| **Dynamic Group** | `Golden Config Managed Devices` |
| **Backup Repository** | your backup repository |
| **Backup Path Template** | `{{ obj.location.name }}/{{ obj.name }}.cfg` |
| **Intended Repository** | your intended repository |
| **Intended Path Template** | `{{ obj.location.name }}/{{ obj.name }}` |
| **Template Repository** | your templates repository |
| **Jinja Path Template** | `{{ obj.platform.network_driver }}.j2` |
| **SoT Aggregation Query** | the GraphQL query from Step 3 |

### Important checks

- `jinja_path_template` must resolve to real template files.
- Backup and intended path templates must match repository layout.
- Devices must have platform/network driver values that map to those template filenames.

---

## Step 5) Verify required device data

Before running jobs, confirm devices in scope have:

- Platform (with correct `network_driver`)
- Role
- Location (if your path template uses location)
- Primary IP (if needed by templates/workflows)
- Any other fields referenced in your templates

Missing required fields usually causes render failures or device exclusion.

---

## Step 6) Configure Config Context data

Templates typically consume:
- Nautobot object attributes (role, platform, interfaces, etc.)
- Config Context values (NTP, ACL, BGP/OSPF data, etc.)

If using Git-managed config contexts:
- root files = **explicit** contexts
- scoped folders (like `locations/`, `roles/`) = **implicit** contexts
- `devices/` files = **local** device context overrides (this repo includes `_metadata` in those files for consistency)

See [config_contexts/README.md](./config_contexts/README.md) for the sample structure used in this repo.

---

## Step 7) Configure credentials (Secrets + Secret Group)

Golden Config jobs that connect to devices typically use `nautobot-plugin-nornir` with Nautobot Secrets.

### 7a) Create secrets

> **Navigation Path:** `Secrets > Secrets > Add`

Create:
- one secret for username
- one secret for password

Test each with **Check Secret**.

### 7b) Create a Secret Group

> **Navigation Path:** `Secrets > Secret Groups > Add`
- Access Type: `Generic`
- Add secret mapping for:
  - `Username`
  - `Password`

Attach this Secret Group where your Golden Config/Nornir workflow expects credentials.

### 7c) Confirm Nornir credentials provider

If using Nautobot Secrets, verify `nautobot_config.py` includes:

```python
PLUGINS_CONFIG = {
    "nautobot_plugin_nornir": {
        "nornir_settings": {
            "credentials": "nautobot_plugin_nornir.plugins.credentials.nautobot_secrets.CredentialsNautobotSecrets",
        }
    }
}
```

---

## Step 8) Run the baseline job flow

After setup, a common flow is:
1. Backup configs
2. Generate intended configs
3. Run compliance
4. Generate remediation and config plans

Compliance/remediation setup details are documented in:
- [CONFIGURING_COMPLIANCE.md](./CONFIGURING_COMPLIANCE.md)

---

## Quick validation checklist

- [ ] Four Git repositories are registered with correct `Provides` values
- [ ] Dynamic Group exists and members are refreshed
- [ ] SoT Aggregation GraphQL query includes required fields
- [ ] Golden Config Settings are linked to the correct repos and query
- [ ] `jinja_path_template` resolves to real template files
- [ ] Devices in scope have required data (platform/role/location/primary IP as needed)
- [ ] Config context data is present and scoped correctly
- [ ] Secrets and Secret Group are working
- [ ] Backup and intended jobs run successfully

---

## Common Issues

- Template path mismatch (`network_driver` does not match template filename)
- Dynamic Group filters too broad or too narrow
- GraphQL query missing fields referenced by templates
- Backup/intended path templates not matching repository layout
- Overusing local device context instead of fixing shared context design
