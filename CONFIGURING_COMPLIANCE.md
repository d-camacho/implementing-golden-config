# Configuring Compliance in Golden Config

This guide is focused on Golden Config Compliance and Remediation.


Primary references:
- [Configuration Compliance](https://docs.nautobot.com/projects/golden-config/en/stable/user/app_feature_compliance/)
- [Configuration Compliance (CLI parsing)](https://docs.nautobot.com/projects/golden-config/en/stable/user/app_feature_compliancecli/)
- [Configuration Remediation](https://docs.nautobot.com/projects/golden-config/en/stable/user/app_feature_remediation/)

---

## 1) What compliance actually means

Golden Config compliance compares:
- **Intended config** (what should be configured)
- **Actual backup config** (what is configured)

For compliance, comparison is based on the features specified in `Compliance Rules`.
If syntax/structure differs, it is non-compliant even if a human might treat it as equivalent.

### How Nautobot generates compliance and remediation

When you run a compliance job, Nautobot/Golden Config does this per device:

1. Loads the device **backup config** from the backup repository (`backup_path_template`).
2. Loads the device **intended config** from the intended repository (`intended_path_template`).
3. Applies your **Compliance Rules** (feature + platform + config-to-match roots) to parse both configs into comparable feature blocks.
4. Compares those feature blocks and stores:
   - **Intended** (what should be there)
   - **Actual** (what is there)
   - **Missing** (in intended, not in actual)
   - **Extra** (in actual, not in intended)
   - compliance status (including order-aware behavior when applicable)

Remediation is then generated from that compliance result:

5. Golden Config checks remediation settings for the platform (for example, `HIERCONFIG`).
6. If remediation is enabled on the matching compliance rule, it uses Intended/Missing/Extra data to build **Remediating Configuration** commands.

So in practice, remediation is not a separate source of truth — it is derived from the backup-vs-intended comparison output.

---

## 2) Prerequisites before compliance will work

Before creating rules, make sure these are already working:

1. Backup repository (`Provides = backup configs`)
2. Intended repository (`Provides = intended configs`)
3. `backup_path_template` points to actual backup files
4. `intended_path_template` points to actual intended files

This path alignment is critical (same guidance as the Nautobot demo layout).

Also important:
- Platform `network_driver` values must be supported by the compliance engine for jobs to run successfully.
- At least one compliance rule must exist for a platform to produce meaningful compliance output.

---

## 3) Create Compliance Features

> **Navigation Path:** `Golden Config > Compliance Features > Add`

Create features as logical config sections, for example:
- NTP
- SNMP
- ACL
- BGP

Tip: Keep feature names concise for readability in the compliance table.

---

## 4) Create Compliance Rules (per platform + feature)

> **Navigation Path:** `Golden Config > Compliance Rules > Add`

Each rule maps one feature to one platform.

Typical fields:
- **Feature**
- **Platform**
- **Config Type** (`CLI` or `JSON`)
- **Config to Match** (root elements to parse)
- **(Optional) Enable Remediation** once remediation is configured

---

## 5) How "Config to Match" works (very important)

For `CLI` config type:
- Matching is based on line **start**
- You match **root/parent lines**, not every child line
- Matching is effectively greedy from the beginning of each line

Examples:

- ACL
  - `ip access-list`
  - `ipv6 access-list`
- SNMP
  - `snmp`
  - `no snmp`

For `JSON` config type:
- Match is based on top-level JSON keys
- If no config-to-match keys are provided, full JSON is compared

Rule design tip:
- Too broad = noisy false positives
- Too narrow = misses real drift

---

## 6) Ordered vs unordered behavior

Golden Config can evaluate features where order matters vs features where order should be ignored.

Important:

- With **HIERCONFIG** remediation/comparison workflows, order handling is already feature-aware by default.
- That means you do **not** need to assume "ACL always needs manual ordered enforcement" just to be safe.
- You can still enforce/override ordering behavior per feature in the UI when your environment needs stricter handling.

### When to enforce ordered checks in the UI

Examples:
- ACL entries with sequence-sensitive behavior
- Route policy/order-dependent statements

### When to allow unordered checks

Examples:
- SNMP community lines where order can vary between renders/backups

In compliance details:
- Intended/Actual can both show as matched but not identically ordered
- UI indicators show ordered-state behavior for the result

---

## 7) Configure remediation

Remediation uses compliance results to generate fix commands.

### 7a) Add remediation settings per platform

> **Navigation Path:** `Golden Config > Remediation Settings > Add`

Remediation types:
- **HIERCONFIG** (common for IOS-like hierarchical CLIs)
- **Custom** remediation function (for non-hierarchical or specialized workflows)

### 7b) Enable remediation per compliance rule

> **Navigation Path:** `Golden Config > Compliance Rules`

Edit each rule and enable remediation for the platform/feature pair you want.

### 7c) Then run compliance and validate remediation output

1. Run compliance job
2. **Navigation Path:** `Golden Config > Config Compliance`
3. Open a device/feature detail
4. Check **Remediating Configuration**

Remediation is derived from compliance data (intended/missing/extra) plus remediation settings.

---

## 8) Run a compliance job

1. **Navigation Path:** `Golden Config > Home`
2. Select Execute > Compliance
3. Choose scope
4. Run the job

Read results in:
- **Navigation Path:** `Golden Config > Config Compliance`
- **Overview** (device-level and feature-level compliance summary)
- **Details** (intended, actual, missing, extra, ordering indicators, and remediation output if enabled)

---

## 9) Practical rule-building workflow (recommended)

1. Start with 1 platform + 1 feature (pilot)
2. Use narrow, obvious root matches
3. Configure remediation settings for the platform
4. Enable remediation on the selected pilot compliance rules
5. Run compliance on a small scope
6. Review false positives/false negatives
7. Adjust match roots and ordered/unordered behavior
8. Expand to more features and platforms

---

## 10) Starter rule set (copy/paste examples)

Use this as a starting point for `CLI` config type rules.  
Create one rule per **Platform + Feature** pair.

### Suggested compliance features

- NTP
- SNMP
- ACL
- OSPF
- BGP

### Starter roots by feature

> These are starter roots, not final truth for every environment.  
> Tune based on your real configs and parser results.

#### ACL

```text
ip access-list
ipv6 access-list
access-list
```

#### SNMP

```text
snmp
no snmp
snmp-server
```

#### NTP

```text
ntp
```

#### OSPF

```text
router ospf
ip ospf
```

OSPF tuning note:
- If you see noisy results on interface-level OSPF commands, start with `router ospf` only, validate, then add `ip ospf` if needed.

#### BGP

```text
router bgp
neighbor
```

BGP tuning note:
- Start with `router bgp` as the primary root.
- Add `neighbor` only if your backups/intended render style requires it for accurate extraction.
- If you see repeated false positives around neighbors or AF blocks, keep roots narrower and validate parent/child parsing in the compliance detail view.

### Ordered vs unordered defaults (good first pass)

- **Default first pass**: trust feature-aware behavior from HierConfig/Golden Config.
- **Then tune per feature in UI** only where your results show a need.
- Common tuning examples:
  - enforce order: sequence-based ACL/policy logic
  - allow unordered: SNMP/community-style declarations where order is not semantically important
  - BGP/OSPF: prefer narrower roots first to reduce parser noise before broadening matches

If your Nautobot/Golden Config version labels this option differently, use the equivalent order-aware setting on the compliance rule.

### Platform notes (`cisco_ios`, `arista_eos`)

- Start with identical feature names across platforms for reporting consistency.
- Keep rule roots platform-aware if syntax differs.
- If one platform needs extra roots (for example, EOS-specific forms), add those in that platform's rule rather than broadening all platforms.

---

## 11) Common pitfalls

- Backup/intended path templates do not map to real files
- Rule platform does not match device platform/network_driver
- Config-to-match roots are too generic
- Using ordered checks for naturally unordered feature data
- Running compliance before remediation is configured (if you expect remediation output in the same run)

---

## 12) Quick checklist

- [ ] Backup and intended repositories are configured and populated
- [ ] Backup and intended path templates resolve correctly
- [ ] Compliance features are created
- [ ] Platform/feature compliance rules are created
- [ ] Root match lines are tested and tuned
- [ ] Ordered vs unordered behavior is set intentionally per feature
- [ ] Remediation settings exist for target platforms
- [ ] Remediation is enabled on validated rules
- [ ] Compliance job runs successfully on pilot scope
