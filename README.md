# Implementing Golden Config

This repository is a practical reference for implementing **Nautobot Golden Config** using:

- sample Jinja2 templates
- sample Config Context data
- setup documentation
- compliance/remediation guidance

It is designed to show how the pieces fit together end-to-end:  
**Source of Truth data + templates + config contexts + backups/intended + compliance rules**.

---

## What is in this repo

### 1) Platform templates and feature templates

Folder: [`template_samples/`](./template_samples/)

Contains entrypoint templates and feature-specific includes for:
- Cisco IOS
- Juniper Junos

Start here:
- [`template_samples/README.md`](./template_samples/README.md)

---

### 2) Config Context sample data

Folder: [`config_contexts/`](./config_contexts/)

Contains sample explicit, implicit, and local-style context files, including:
- global defaults
- location-specific values
- role-specific values
- device-specific routing data

Start here:
- [`config_contexts/README.md`](./config_contexts/README.md)

---

### 3) Golden Config application setup guide

Document: [`GOLDEN_CONFIG_SETUP.md`](./GOLDEN_CONFIG_SETUP.md)

Covers core setup flow:
- Git repositories and `Provides` values
- Dynamic Group scope
- SoT Aggregation GraphQL query
- Golden Config settings
- device prerequisites
- secrets and credentials
- baseline job flow

---

### 4) Compliance and remediation guide

Document: [`CONFIGURING_COMPLIANCE.md`](./CONFIGURING_COMPLIANCE.md)

Covers:
- how compliance compares intended vs backup
- rule design (`Config to Match`, CLI/JSON)
- ordered vs unordered behavior
- remediation settings and enablement
- starter rule set and workflow

---

## Repository layout

```text
.
├── README.md
├── GOLDEN_CONFIG_SETUP.md
├── CONFIGURING_COMPLIANCE.md
├── config_contexts/
│   ├── README.md
│   ├── global_defaults.yml
│   ├── locations/
│   ├── roles/
│   └── devices/
└── template_samples/
    ├── README.md
    ├── cisco_ios.j2
    ├── juniper_junos.j2
    ├── ios/
    └── junos/
```

---

## Suggested order to use this repo

1. Read [`GOLDEN_CONFIG_SETUP.md`](./GOLDEN_CONFIG_SETUP.md)
2. Review template approach in [`template_samples/README.md`](./template_samples/README.md)
3. Review context layering in [`config_contexts/README.md`](./config_contexts/README.md)
4. Configure and validate compliance with [`CONFIGURING_COMPLIANCE.md`](./CONFIGURING_COMPLIANCE.md)

---

## Notes

- These files are reference samples and should be adapted to your own platforms, naming, and change-control requirements.
- Path templates and platform `network_driver` values must match your actual Nautobot and repository data.

---

## Official documentation

- [Nautobot Golden Config Docs](https://docs.nautobot.com/projects/golden-config/en/stable/)
- [Nautobot Config Context Docs](https://docs.nautobot.com/projects/core/en/latest/user-guide/core-data-model/extras/configcontext/)
- [Nautobot Git Repository Docs](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/gitrepository/)