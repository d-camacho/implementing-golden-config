# Config Context Samples (Golden Config)

This folder contains sample Config Context data for learning how Golden Config templates get their values.

On top of Nautobot object attributes, data can be supplied with Config Context.

Quick mental model:
- **Nautobot object attributes** = built-in device facts (platform, role, interfaces, etc.)
- **Templates** = command structure
- **Config Context** = extra values that templates consume (NTP servers, ACL details, BGP data, etc.)

Focus on scope + precedence:
1. Put shared data in broad contexts.
2. Put differences in narrower contexts.
3. Use local device context only for true exceptions.

References:
- [Nautobot Config Contexts - Hierarchical Rendering](https://docs.nautobot.com/projects/core/en/latest/user-guide/core-data-model/extras/configcontext/#hierarchical-rendering)
- [Nautobot Git Repositories - Implicit Config Contexts](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/gitrepository/#implicit-config-contexts)

---

## Current sample layout

```text
config_contexts/
├── global_defaults.yml
├── locations/
│   ├── LAX.yml
│   └── NYC.yml
├── roles/
│   ├── Access Switch.yml
│   └── Core Switch.yml
└── devices/
    ├── LAX-BR.yml
    └── NYC-BR.yml
```

---

## Explicit vs implicit vs local (using these samples)

### Explicit (root files)

Files at the root of `config_contexts/` are explicit contexts.
Scope is declared in `_metadata`.

Sample in this repo:
- `global_defaults.yml` (applies broadly as your base context)

### Implicit (folder + filename define scope)

Files in scoped folders use implicit matching:
`config_contexts/<filter>/<name>.yml`

Samples in this repo:
- `locations/LAX.yml`
- `locations/NYC.yml`
- `roles/Access Switch.yml`
- `roles/Core Switch.yml`

For implicit files:
- folder defines filter type (`locations`, `roles`, etc.)
- filename maps to object name (for device-types, model)
- `_metadata` is still used for name/weight/description/is_active

### Local (device-specific)

Files in `config_contexts/devices/` are local device context.

Samples in this repo:
- `devices/LAX-BR.yml`
- `devices/NYC-BR.yml`

For local files:
- filename maps to device name
- data is used as-is
- this repo includes `_metadata` in device files for readability and consistent metadata tracking
- local values override inherited context for that device

---

## How these samples layer together

A device can match multiple contexts at once, then Nautobot merges them into one final `config_context`.

Example flow for a core switch in LAX:
1. `global_defaults.yml` (base values)
2. `locations/LAX.yml` (location-specific values)
3. `roles/Core Switch.yml` (role-specific values)
4. `devices/<device-name>.yml` only if there is a local file

### Precedence rules

- Higher `weight` wins on conflicts.
- Local device context wins over other contexts for that device.

### Merge behavior to remember

Per Nautobot hierarchical rendering behavior:
- dictionary keys with **list values** commonly overwrite per key at higher weight
- dictionary keys with **dictionary values** deep-merge additively

Design your data model intentionally depending on whether you want overwrite or additive behavior.

---

## File-by-file purpose in this repo

- `global_defaults.yml`
  - Baseline settings used by all demo devices.
  - Includes reusable shared structures (banner, line vty, global ACL set).

- `locations/LAX.yml` and `locations/NYC.yml`
  - Location-specific operational data (NTP/SNMP/regional ACL data).

- `roles/Access Switch.yml` and `roles/Core Switch.yml`
  - Role-specific ACL content.

- `devices/LAX-BR.yml` and `devices/NYC-BR.yml`
  - Device-level routing data (OSPF/BGP) used only for those exact devices.

---

## Key alignment with `template_samples/`

This is the practical mapping for this repo today:

| Config Context Key | Used by Template | Notes |
|---|---|---|
| `ntp` | `template_samples/ios/ios_ntp.j2`, `template_samples/junos/junos_ntp.j2` | Shared top-level key with platform-specific rendering |
| `acl_global` | `template_samples/ios/ios_acl.j2` | Baseline ACL list |
| `acl_regional` | `template_samples/ios/ios_acl.j2` | Location override/addition ACL list |
| `acl_roles` | `template_samples/ios/ios_acl.j2` | Role-specific ACL list |
| `interfaces` | `template_samples/ios/ios_interfaces.j2` | Optional; only renders if key exists |
| `bootstrap` | `template_samples/junos/junos_bootstrap.j2` | Optional; only renders if key exists |

Other keys in the sample data such as `banner`, `line_vty`, `ospf`, and `bgp` are valid sample context patterns, but they are not rendered by the current minimal sample templates unless you add matching feature templates.

---

## Tips for adding new files

1. Start with global defaults first.
2. Add location or role files only for differences.
3. Keep local device files for one-off exceptions.
4. Use consistent key naming between files and templates.
5. Keep weights simple and readable (e.g., 1000, 2000, 3000).
6. In templates, use defensive access patterns (`config_context.get(...)`).

---

## Troubleshooting checklist

If rendered output is unexpected:
1. Confirm the device name/location/role exactly matches implicit filenames.
2. Confirm which contexts matched the device in Nautobot.
3. Check `_metadata.weight` for conflicts.
4. Check whether a local device file is overriding shared values.
5. Validate template logic for optional/missing keys.
