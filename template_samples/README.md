# Golden Config Templating Guide

If you're new to Golden Config templating, the biggest win is this:

**Keep templates simple, reusable, and safe when data is missing.**

This guide shows a practical way to structure templates so they are easy to understand and maintain.

---

## Quick mental model

Think of Golden Config templating as 3 layers:

1. **Template selector**: picks which top-level file to use (usually by platform).
2. **Entrypoint template**: a small file that includes feature templates.
3. **Feature templates**: focused blocks (NTP, AAA, ACL, SNMP, etc.) with defensive checks.

If you design around those 3 layers, your templates scale much better.

---

## 1) Start with one template per platform

For multi-vendor environments, platform is usually the first split.

Example:

```text
templates/
├── cisco_ios.j2
├── cisco_nxos.j2
└── juniper_junos.j2
```

Each of these is an **entrypoint** template for that platform.

---

## 2) How Golden Config picks a template

Golden Config uses `jinja_path_template` under Template Configuration.

A common value is:

```jinja2
{{ obj.platform.network_driver }}.j2
```

So if a device has platform/network driver `cisco_ios`, Golden Config renders `cisco_ios.j2`.

### Important

- If a device has no platform/network driver set, template selection can fail.
- If the computed file does not exist, config generation fails.

---

## 3) Keep the entrypoint small

A huge all-in-one template gets hard to manage fast.

Instead, keep your entrypoint mostly as include statements:

```jinja2
{% include "./ios/ios_ntp.j2" %}
!
{% include "./ios/ios_interfaces.j2" %}
!
{% include "./ios/ios_ospf.j2" %}
!
{% include "./ios/ios_bgp.j2" %}
!
{% include "./ios/ios_snmp.j2" %}
!
{% include "./ios/ios_acl.j2" %}
```

This makes it easier to read, test, and update one feature at a time.

---

## 4) Example folder layout

```text
templates/
├── cisco_ios.j2            # Entrypoint for IOS
├── juniper_junos.j2        # Entrypoint for Junos
├── ios/
│   ├── ios_acl.j2
│   ├── ios_bgp.j2
│   ├── ios_interfaces.j2
│   ├── ios_ntp.j2
│   ├── ios_ospf.j2
│   └── ios_snmp.j2
└── junos/
    ├── junos_bootstrap.j2
    └── junos_ntp.j2
```

---

## 5) Using variables in templates

You can render values from:

- Nautobot/SoT data exposed in render context (device attributes, interfaces, role, etc.)
- Config Context data
- Or both

### Example: device interfaces

```jinja2
{# Loop through interfaces from SoT/ORM-derived context #}
{% for intf in all_interfaces %}
interface {{ intf["name"] }}
{% endfor %}
```

Expected output:

```text
interface GigabitEthernet0
interface GigabitEthernet1
```

---

### Example: Config Context for NTP

Template:

```jinja2
{% if config_context.get("ntp") %}
{% set ntp = config_context["ntp"] %}
{% for server in ntp.get("servers", []) %}
ntp server {{ server["ip"] }}{% if server.get("prefer") %} prefer{% endif %}
{% endfor %}
{% endif %}
```

Config Context (YAML):

```yaml
ntp:
  servers:
    - ip: "192.168.1.1"
      prefer: true
    - ip: "172.16.3.33"
```

Rendered output:

```text
ntp server 192.168.1.1 prefer
ntp server 172.16.3.33
```

---

## 6) Defensive templating (super important)

Defensive templating means templates do not crash when optional data is missing.

That makes one template reusable across many devices with different config context data.

### Pattern A: guard the feature block

```jinja2
{% if config_context.get("ntp") %}
{% set ntp = config_context["ntp"] %}
...
{% endif %}
```

### Pattern B: safely handle optional keys

```jinja2
{% if ntp.get("key") and ntp.get("auth") %}
ntp authentication-key {{ ntp["key"] }} md5 {{ ntp["auth"] }}
{% endif %}

{% if server.get("prefer") %}
ntp server {{ server["ip"] }} key {{ ntp["key"] }} prefer
{% else %}
ntp server {{ server["ip"] }} key {{ ntp["key"] }}
{% endif %}
```

### Pattern C: role-based conditionals

```jinja2
{% if obj.role and obj.role.name == "core_switch" %}
ip tacacs source-interface Loopback0
{% endif %}
```

---

## 7) Common mistakes to avoid

- Building one massive template file for everything.
- Accessing nested keys directly without checking if they exist.
- Mixing platform logic all over many files instead of splitting by entrypoint.
- Using inconsistent variable names (for example `int` in loop and `intf` in output).
- Not setting platform/network driver values correctly in Nautobot.

---

## 8) Suggested workflow when creating new templates

1. Create one entrypoint template for the platform.
2. Add feature templates one by one (`ntp`, `aaa`, `snmp`, etc.).
3. Wrap each feature with a top-level guard (`config_context.get(...)`).
4. Use `.get()` for optional sub-keys.
5. Test render with:
   - full context
   - partial context
   - no feature context

If your template still renders safely in all 3 cases, you are in a good spot.
