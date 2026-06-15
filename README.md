# Catalyst Center SWIM Automation for Catalyst 9000 (Virtual)

Operationalize **Software Image Management (SWIM)** on Cisco Catalyst Center using the
`cisco.catalystcenter` Ansible collection. This repository delivers an opinionated,
phase-by-phase upgrade pipeline — import, golden tagging, distribution, activation, post-validation,
and rollback — driven entirely from a single declarative data file.

Validated against **Cisco Catalyst Center 2.3.7.6** with `cisco.catalystcenter` collection
`2.9.0` and `catalystcentersdk` `3.1.6.x`, targeting Cisco Catalyst 9000 Series Virtual switches
(`cat9kv`).

---

## Table of Contents

1. [SWIM on Catalyst Center — methodology](#1-swim-on-catalyst-center--methodology)
2. [The Cisco Catalyst Center Ansible collection](#2-the-cisco-catalyst-center-ansible-collection)
3. [Mapping SWIM operations to workflow modules](#3-mapping-swim-operations-to-workflow-modules)
4. [Repository layout](#4-repository-layout)
5. [Install dependencies](#5-install-dependencies)
6. [Configure credentials (Ansible Vault)](#6-configure-credentials-ansible-vault)
7. [The data model — `vars/images.yml`](#7-the-data-model--varsimagesyml)
8. [The playbooks, explained](#8-the-playbooks-explained)
9. [Run sequence](#9-run-sequence)
10. [Evidence and logging](#10-evidence-and-logging)
11. [Troubleshooting](#11-troubleshooting)
12. [Appendix A: Image distribution (HTTP) server setup](#appendix-a-image-distribution-http-server-setup)
13. [Appendix B: Alternative approach — individual API modules (no Workflow Manager)](#appendix-b-alternative-approach--individual-api-modules-no-workflow-manager)

---

## 1. SWIM on Catalyst Center — methodology

**Software Image Management (SWIM)** is the Catalyst Center function that centralizes the storage,
validation, and lifecycle of network operating-system images. Rather than copying `.bin` files to
devices by hand and reloading them one at a time, SWIM lets you define a *desired* software state
per device family and role, then drives every managed device toward it with built-in pre/post
validation. This automation mirrors the GUI workflow documented in the
[Catalyst Center 2.3.7 User Guide — Manage Software Images](https://www.cisco.com/c/en/us/td/docs/cloud-systems-management/network-automation-and-management/catalyst-center/2-3-7/user_guide/b_cisco_catalyst_center_user_guide_237/b_cisco_dna_center_ug_2_3_7_chapter_0100.html).

### 1.1 The image repository

Catalyst Center maintains a central **image repository** that stores every unique software image,
SMU, subpackage, and ROMMON image by type and version. During import, Catalyst Center performs
**Integrity Verification (IV)** — comparing the software/hardware checksum of the imported image
against a **Known Good Values (KGV)** file — to detect tampering or corruption before the image is
ever pushed to a device.

### 1.2 Golden images and device roles

A **golden image** is a validated, standardized image that an operator designates as the
*compliance target* for a given **device family** — and optionally narrowed to a specific
**device role** (e.g. `CORE`, `DISTRIBUTION`, `ACCESS`, `BORDER ROUTER`, or `ALL`). Designating an
image as golden is how you tell Catalyst Center "this is what these devices *should* be running."
Catalyst Center then continuously compares each device's running image against its golden tag and
flags any device that is **outdated**.

> **Key idea:** the golden tag is the contract. Distribution and activation simply move devices
> toward that contract; compliance reporting measures how far each device is from it.

### 1.3 The upgrade lifecycle

A SWIM upgrade is a sequence of distinct, independently observable operations:

| Stage | What Catalyst Center does |
|---|---|
| **Import** | Pulls the image into the repository (from cisco.com, a URL/file server, or local upload) and runs Integrity Verification. |
| **Golden tag** | Marks the image as the compliance target for a device family + role + site. |
| **Upgrade readiness prechecks** | File-transfer reachability, NTP clock, flash space (with auto-cleanup), config register, crypto/TLS, startup config, image compatibility, and version-support checks. A device that fails prechecks is blocked from upgrade. |
| **Distribute** | Copies (`install add`) the image to device flash. **No reboot** — the running image is untouched. |
| **Activate** | Makes the new image the running image (`install activate` + `install commit`). **This reboots the device.** Can be scheduled, ordered (parallel/sequential), and gated by pre/post activation checks. |
| **Postchecks** | CPU usage, route summary, interface state, and image-activation verification to confirm the network state is unchanged. |

Separating **distribute** from **activate** is deliberate and operationally important: you can
stage the binary onto every device during business hours (zero impact), then activate (reboot)
inside a maintenance window. This repository preserves that separation as two distinct playbooks.

### 1.4 Image distribution server

For at-scale or remote-site distribution, Catalyst Center can hand off image transfer to an
external **image distribution server** (SFTP/SCP/HTTPS), or — as in this lab — pull a remotely
hosted image over HTTP during import. See [Appendix A](#appendix-a-image-distribution-http-server-setup).

---

## 2. The Cisco Catalyst Center Ansible collection

The [`cisco.catalystcenter`](https://galaxy.ansible.com/ui/repo/published/cisco/catalystcenter/)
collection is Cisco's officially supported Ansible content for Catalyst Center. It ships two
distinct families of modules:

- **SDK / "intent" modules** — thin, one-to-one wrappers around individual Catalyst Center Intent
  API endpoints (e.g. `swim_import_local`, `swim_trigger_distribution`). These are granular and
  imperative: you orchestrate the ordering, polling, and idempotency yourself.
- **Workflow Manager modules** (the `*_workflow_manager` family) — Cisco's higher-level,
  **declarative** automation. You describe the *desired end state*, and the module performs the
  multi-step API choreography (lookups, ID resolution, submission, task polling, and verification)
  to reach it. These are the modules Cisco recommends for day-2 operations, and the ones this
  repository is built on.

### 2.1 Why Workflow Manager modules

Every playbook here uses `cisco.catalystcenter.swim_workflow_manager` (and
`inventory_workflow_manager` / `network_compliance_workflow_manager` for supporting phases).
Full parameter reference and annotated examples are available in the
[`swim_workflow_manager` module documentation on Ansible Galaxy](https://galaxy.ansible.com/ui/repo/published/cisco/catalystcenter/content/module/swim_workflow_manager/).
Workflow Manager modules give us:

- **Declarative `config:`** — you pass the same data structures the Catalyst Center GUI builds
  internally (`import_image_details`, `tagging_details`, `image_distribution_details`,
  `image_activation_details`).
- **Built-in task polling** — the module submits the asynchronous Catalyst Center task and blocks
  until it completes, succeeds, or times out, governed by `catalystcenter_api_task_timeout` and
  `catalystcenter_task_poll_interval`.
- **`config_verify: true`** — after applying, the module re-reads Catalyst Center state to confirm
  the change actually landed, rather than trusting the submit response alone.
- **Idempotency** — re-running a play that imports an already-present image, or tags an
  already-golden image, reports `ok` instead of failing or duplicating work.

### 2.2 Connection model

Catalyst Center modules run with `connection: local` against the control node — Ansible talks to
the Catalyst Center REST API over HTTPS; there is no SSH to the appliance. A single placeholder
host (`catalyst_center`, in group `catc`) carries the connection variables. Authentication and
endpoint parameters are supplied to every task from
[`inventory/group_vars/catc/connection.yml`](inventory/group_vars/catc/connection.yml) and the
encrypted vault.

---

## 3. Mapping SWIM operations to workflow modules

Each SWIM stage from [§1.3](#13-the-upgrade-lifecycle) maps to a specific Workflow Manager module
invocation. The table below is the Rosetta Stone for this repository. See the
[`swim_workflow_manager` module docs](https://galaxy.ansible.com/ui/repo/published/cisco/catalystcenter/content/module/swim_workflow_manager/)
for the full parameter schema and the
[annotated examples](https://galaxy.ansible.com/ui/repo/published/cisco/catalystcenter/content/module/swim_workflow_manager/?keywords=swim#examples)
for each `config:` sub-key.

| SWIM operation | Module | `config:` key | Playbook |
|---|---|---|---|
| Import image to repository | `swim_workflow_manager` | `import_image_details` | [`10_import_and_tag.yml`](playbooks/10_import_and_tag.yml) |
| Mark image golden | `swim_workflow_manager` | `tagging_details` | [`10_import_and_tag.yml`](playbooks/10_import_and_tag.yml) |
| Distribute image to devices | `swim_workflow_manager` | `image_distribution_details` | [`20_distribute.yml`](playbooks/20_distribute.yml) |
| Activate image (reload) | `swim_workflow_manager` | `image_activation_details` | [`30_activate.yml`](playbooks/30_activate.yml) |
| Rollback (re-tag + activate prior image) | `swim_workflow_manager` | `tagging_details` → `image_activation_details` | [`35_rollback.yml`](playbooks/35_rollback.yml) |
| Inventory resync (refresh device state) | `inventory_workflow_manager` | `ip_address_list` / `site_name` | [`00_preflight.yml`](playbooks/00_preflight.yml) |
| IMAGE compliance check | `network_compliance_workflow_manager` | `run_compliance_categories: [IMAGE]` | [`00_preflight.yml`](playbooks/00_preflight.yml), [`40_postcheck.yml`](playbooks/40_postcheck.yml) |

### 3.1 The three different "device family" names — a critical distinction

The single most common source of SWIM automation failures is conflating three *different* names
that Catalyst Center uses in three different contexts. This repository keeps them explicit:

| Where | Field | Value on this cluster | Source of truth |
|---|---|---|---|
| **Golden tag** (`tagging_details`) | `device_image_family_name` | `Cisco Catalyst 9000 UADP 8 Port Virtual Switch` (id `999999901`) | `GET /dna/intent/api/v1/image/importation/device-family-identifiers` |
| **Distribute / activate** | `device_family_name` | `Switches and Hubs` | Inventory (network-device API `family`) |
| **Distribute / activate** | `device_series_name` | `Cisco Catalyst 9000 Series Virtual Switches` | Inventory (network-device API `series`) |

> **Why this matters:** tagging is keyed to the **SWIM device-family identifier**, while
> distribution and activation select devices using the **inventory** family/series. Using the
> golden-tag identifier in a distribute call returns *"no eligible devices."* Both names are
> pre-resolved in `vars/images.yml` so operators never have to guess.

---

## 4. Repository layout

```
ansible-dnac-swim/
├── ansible.cfg                       # inventory path, callbacks, vault settings
├── requirements.txt                  # Python deps (ansible-core, ansible-lint, catalystcentersdk)
├── requirements.yml                  # cisco.catalystcenter collection pin (2.9.0)
├── .ansible-lint                     # lint profile
├── inventory/
│   ├── hosts.yml                     # 'catalyst_center' host in group 'catc' (connection: local)
│   └── group_vars/catc/
│       ├── connection.yml            # catalystcenter_* connection + logging + timeout params
│       └── vault.yml                 # ENCRYPTED: catalystcenter_username / _password
├── vars/
│   └── images.yml                    # single source of truth: swim_details data model
├── playbooks/
│   ├── 00_preflight.yml              # resync + baseline IMAGE compliance
│   ├── 10_import_and_tag.yml         # import images + mark golden
│   ├── 20_distribute.yml             # copy image to flash (no reload)
│   ├── 30_activate.yml               # activate image (reloads devices)
│   ├── 35_rollback.yml               # re-tag + activate prior-stable image
│   └── 40_postcheck.yml              # post-activation IMAGE compliance
└── logs/                             # SDK log + per-phase JSON evidence (gitignored)
```

---

## 5. Install dependencies

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml
```

> **Python version:** `catalystcentersdk` requires **Python ≥ 3.12**. Build the virtualenv with a
> 3.12+ interpreter.

---

## 6. Configure credentials (Ansible Vault)

Catalyst Center credentials live in [`inventory/group_vars/catc/vault.yml`](inventory/group_vars/catc/vault.yml)
(`catalystcenter_username`, `catalystcenter_password`) as a **whole-file Ansible Vault** blob.
Nothing is committed in plaintext; the file is decrypted at run time when you supply the vault
password.

Edit the stored credentials in place (you'll be prompted for the vault password):

```bash
ansible-vault edit inventory/group_vars/catc/vault.yml
```

Supply the vault password at run time — interactively:

```bash
ansible-playbook playbooks/00_preflight.yml --ask-vault-pass
```

…or from a password file (keep it out of version control; `.vault_pass` is in `.gitignore`):

```bash
ansible-playbook playbooks/00_preflight.yml --vault-password-file .vault_pass
```

### 6.1 Connection parameters

[`inventory/group_vars/catc/connection.yml`](inventory/group_vars/catc/connection.yml) holds the
non-secret connection settings applied to every task:

| Variable | Value | Purpose |
|---|---|---|
| `catalystcenter_host` | `198.18.129.100` | Catalyst Center IP/FQDN |
| `catalystcenter_port` | `443` | HTTPS port |
| `catalystcenter_version` | `2.3.7.6` | Targets the correct API surface |
| `catalystcenter_verify` | `false` | Skip TLS verification (self-signed lab cert) |
| `catalystcenter_log` / `_log_level` | `true` / `INFO` | SDK logging to `logs/catc-swim.log` |
| `catalystcenter_api_task_timeout` | `3600` | Max seconds to wait on an async task |
| `catalystcenter_task_poll_interval` | `30` | Seconds between task-status polls |
| `validate_response_schema` | `true` | Validate API responses against the SDK schema |

> Activation tasks override the timeout/poll values inline (`7200` / `60`) because device reloads
> take longer than import or distribution.

---

## 7. The data model — `vars/images.yml`

All deployment intent lives in **one file**, [`vars/images.yml`](vars/images.yml), under a single
`swim_details` key. Its structure deliberately mirrors the `swim_workflow_manager` config blocks,
so what you read in the data file is exactly what is sent to Catalyst Center.

```yaml
swim_details:
  import_images:        # → 10: list of { name, import_image_details }
  golden_tag_images:    # → 10: list of { name, tagging_details }
  distribute_images:    # → 20: list of { name, image_distribution_details }
  activate_images:      # → 30: list of { name, image_activation_details }
  rollback_images:
    tag:                # → 35: list of { name, tagging_details }
    activate:           # → 35: list of { name, image_activation_details }
```

| Section | Consumed by | Module config key | Purpose |
|---|---|---|---|
| `import_images` | `10` | `import_image_details` | Image source URL(s). Includes **both** the upgrade and the rollback image, so the rollback target is in the repository *before* any activation. |
| `golden_tag_images` | `10` | `tagging_details` | Mark the upgrade image golden for a site + role, using the **SWIM device-family identifier**. |
| `distribute_images` | `20` | `image_distribution_details` | Copy targeting by site + role + **inventory** family/series. |
| `activate_images` | `30` | `image_activation_details` | Activation targeting + flags (`activate_lower_image_version`, `distribute_if_needed`, `schedule_validate`). |
| `rollback_images.tag` | `35` | `tagging_details` | Re-tag the prior-stable image golden. |
| `rollback_images.activate` | `35` | `image_activation_details` | Activate the prior-stable image (`activate_lower_image_version: true`). |

### 7.1 Design principles

- **`name` is a label, not data.** Every entry carries a human-readable `name` (e.g.
  `cat9kv_throttle_26.2 @ Floor 1 / ALL`) used only for the Ansible `loop_control.label`. It is
  never sent to the module — the playbook passes only the module-config key beneath it.
- **One file, no joins.** Earlier iterations split image identity from a separate rollout-waves
  file and joined them at runtime with Jinja2. That indirection is gone: each section is a flat,
  ready-to-loop list. Playbooks pass `item.<config_key>` straight through to the module.
- **Targets derive from the data, not from CLI args.** Pre-flight and post-check derive their
  site list automatically from `golden_tag_images[*].tagging_details.site_name` — there is no
  separate IP list or `wave_name` to keep in sync.
- **Rollback is first-class.** The rollback image is imported alongside the upgrade image so it is
  always present in the repository, never scrambled for after a failure.

### 7.2 Activation flags worth knowing

| Flag | This repo | Meaning |
|---|---|---|
| `activate_lower_image_version` | `false` (upgrade) / `true` (rollback) | Allow activating an image *older* than what's running. Required for rollback. |
| `distribute_if_needed` | `true` | If the image isn't already on the device, distribute it first as part of activation. |
| `schedule_validate` | `true` | Run Catalyst Center's activation pre-validation before committing. |

### 7.3 Adding a site or role

Append another entry (with its own `name`) to the relevant section(s). No playbook edits required —
the loops and the derived compliance-site list pick it up automatically.

---

## 8. The playbooks, explained

All playbooks target `hosts: catc`, run `connection: local`, load `vars/images.yml`, and stamp a
`run_id` (`YYYYMMDD-HHMMSS`) onto their evidence file. Each phase is independently runnable and
idempotent (except activation/rollback, which reboot devices).

### 8.1 `00_preflight.yml` — resync + baseline compliance

**Purpose:** establish a known-good starting point and a *before* compliance snapshot.

1. **Derive `target_sites`** from `golden_tag_images[*].tagging_details.site_name` (deduplicated).
2. **Resync devices** at those sites (`inventory_workflow_manager`, `device_resync: true`) so
   Catalyst Center's view of running images is current before any decisions are made.
3. **Run IMAGE compliance** (`network_compliance_workflow_manager`,
   `run_compliance_categories: [IMAGE]`) per site to record the baseline.
4. **Evaluate results** — surfaces skipped devices and hard failures, writes evidence, and **fails
   fast** on a resync failure or a compliance task error so you don't upgrade on a broken baseline.

> Compliance categories supported by the module: `INTENT`, `RUNNING_CONFIG`, `IMAGE`, `PSIRT`,
> `EOX`, `NETWORK_SETTINGS`. This phase scopes to `IMAGE` only.

### 8.2 `10_import_and_tag.yml` — import + golden tag

**Purpose:** populate the repository and set the compliance contract.

1. **Import** — loops `swim_details.import_images`, passing `import_image_details` to
   `swim_workflow_manager`. Catalyst Center pulls each `.bin` over HTTP and runs Integrity
   Verification. Both the upgrade and rollback images are imported here.
2. **Golden tag** — loops `swim_details.golden_tag_images`, passing `tagging_details` to mark the
   upgrade image golden for the target site + role using the SWIM device-family identifier.

Idempotent: an already-imported image or already-golden tag returns `ok`.

> **Module reference:**
> [`swim_workflow_manager` — `import_image_details` / `tagging_details` examples](https://galaxy.ansible.com/ui/repo/published/cisco/catalystcenter/content/module/swim_workflow_manager/?keywords=swim#examples)

### 8.3 `20_distribute.yml` — stage the binary (no reload)

**Purpose:** copy the image to device flash with **zero service impact**.

Loops `swim_details.distribute_images`, passing `image_distribution_details` (targeted by
**inventory** family/series). Catalyst Center runs upgrade-readiness prechecks (flash space with
auto-cleanup, reachability, checksum) and copies the image. Devices keep running their current
image. Safe to run during business hours.

> **Module reference:**
> [`swim_workflow_manager` — `image_distribution_details` examples](https://galaxy.ansible.com/ui/repo/published/cisco/catalystcenter/content/module/swim_workflow_manager/?keywords=swim#examples)

### 8.4 `30_activate.yml` — activate (reloads devices)

**Purpose:** make the new image the running image. **This reboots devices** — run inside a
maintenance window.

Loops `swim_details.activate_images`, passing `image_activation_details` with
`distribute_if_needed: true` (distributes first if needed) and `schedule_validate: true`. Uses an
extended timeout (`7200s`) and poll interval (`60s`) to accommodate reloads.

> **Module reference:**
> [`swim_workflow_manager` — `image_activation_details` examples](https://galaxy.ansible.com/ui/repo/published/cisco/catalystcenter/content/module/swim_workflow_manager/?keywords=swim#examples)

#### IOS-XE install-mode three-step sequence

Catalyst Center drives the activation as three sequential CLI operations on each device,
as documented in the
[Catalyst Center 2.3.7 User Guide — View image update workflow](https://www.cisco.com/c/en/us/td/docs/cloud-systems-management/network-automation-and-management/catalyst-center/2-3-7/user_guide/b_cisco_catalyst_center_user_guide_237/b_cisco_dna_center_ug_2_3_7_chapter_0100.html):

> *"For Cisco Catalyst 9000 Series Switches running on IOS-XE software, the software image is
> upgraded in three steps: `install-add` (Distribution), `install-activate` (Activation), and
> `install-commit` (Activation)."*
>
> *"If the device is in Inactive state, the `install-add` command is executed first.
> Subsequently, the `install-activate` and `install-commit` commands are executed."*

| CLI step | Phase | Device state after | Reboots? |
|---|---|---|---|
| `install add` | **Distribute** (phase 20) | `IMG I <new-version>` — Inactive | No |
| `install activate` | **Activate** (phase 30) | `IMG U <new-version>` — Activated & Uncommitted | **Yes** |
| `install commit` | **Activate** (phase 30, post-reload) | `IMG C <new-version>` — Activated & Committed | No |

> **Important for `cat9kv` (virtual switches):** In some virtual/simulated environments,
> Catalyst Center's activate task may report `SUCCESS` after submitting the `install-activate`
> API call, before confirming that the device has actually reloaded. If `show install summary`
> shows `IMG I` (Inactive) after the playbook completes with no errors, the `install-activate`
> step did not trigger the reload. Manually run:
>
> ```
> switch# install activate version <new-version>
> switch# install commit
> ```
>
> on each affected device, then proceed to phase 40.

### 8.5 `40_postcheck.yml` — post-activation compliance

**Purpose:** confirm the upgrade achieved the golden state.

Derives the same site list from `golden_tag_images`, runs an `IMAGE` compliance check per site, and
asserts the workflow executed successfully. The *after* snapshot to compare against `00`'s baseline.

### 8.6 `35_rollback.yml` — return to prior-stable (reloads devices)

**Purpose:** recover from a bad upgrade by activating the previously known-good image.

> **Module reference:**
> [`swim_workflow_manager` — rollback (`activate_lower_image_version: true`) examples](https://galaxy.ansible.com/ui/repo/published/cisco/catalystcenter/content/module/swim_workflow_manager/?keywords=swim#examples)

Guarded by **two explicit confirmation gates** — the play refuses to run without both:

```bash
-e rollback_confirm=YES -e rollback_reload_ack=RELOAD_OK
```

1. **Re-tag** the prior-stable image golden (`rollback_images.tag`).
2. **Activate** the prior-stable image (`rollback_images.activate`) with
   `activate_lower_image_version: true` — required because the rollback image's version is *lower*
   than the currently running upgrade. Reboots devices.

---

## 9. Run sequence

Standard upgrade, end to end:

```bash
ansible-playbook playbooks/00_preflight.yml      --ask-vault-pass   # resync + baseline compliance
ansible-playbook playbooks/10_import_and_tag.yml --ask-vault-pass   # import + golden tag
ansible-playbook playbooks/20_distribute.yml     --ask-vault-pass   # stage binary (no reload)
ansible-playbook playbooks/30_activate.yml       --ask-vault-pass   # activate (RELOADS devices)
ansible-playbook playbooks/40_postcheck.yml      --ask-vault-pass   # post-activation compliance
```

Rollback (double confirmation, reboots devices):

```bash
ansible-playbook playbooks/35_rollback.yml \
  -e rollback_confirm=YES \
  -e rollback_reload_ack=RELOAD_OK \
  --ask-vault-pass
```

> Substitute `--vault-password-file .vault_pass` for `--ask-vault-pass` for non-interactive runs.
> Run `20` during business hours; gate `30` and `35` behind a maintenance window.

---

## 10. Evidence and logging

- **SDK / module log:** `logs/catc-swim.log` (appended; level `INFO`).
- **Per-phase JSON evidence:** every playbook writes a timestamped artifact, e.g.
  `logs/<run_id>-00_preflight.json`, `-10_import_and_tag.json`, `-20_distribute.json`,
  `-30_activate.json`, `-40_postcheck.json`, `-35_rollback.json`. Each captures the full module
  result (registered output) for audit and post-incident review.

---

## 11. Troubleshooting

| Symptom | Likely cause | Resolution |
|---|---|---|
| Import fails / image never appears | File-server URL wrong, or CatC can't reach it | Verify the exact URL from CatC's perspective (see [Appendix A.5](#a5-verify)); confirm `application/octet-stream`. |
| Golden tag "succeeds" but image absent in UI device-family view | THROTTLE/engineering builds have empty `applicableDevicesForImage` | Expected for unverifiable builds — the tag is set via API; the device-family view shows the *running* image only. |
| Distribute reports "no eligible devices" | Used the **golden-tag identifier** as `device_family_name` | Use the **inventory** family (`Switches and Hubs`) + series, not the SWIM identifier. |
| Activation times out | Reload exceeds default timeout | Confirm the phase uses the extended `catalystcenter_api_task_timeout: 7200`. |
| Compliance check skips devices | Devices unreachable or not fully managed | Re-run `00_preflight.yml` to resync; inspect `skipped_devices` in the evidence JSON. |
| `catalystcentersdk` import error | Python < 3.12 | Rebuild the virtualenv with Python 3.12+. |

---

## Appendix B: Alternative approach — individual API modules (no Workflow Manager)

The playbooks in this repository use `swim_workflow_manager` because it is the recommended,
declarative path. However, the `cisco.catalystcenter` collection also ships lower-level,
**individual API modules** — thin wrappers around single REST endpoints that expose every parameter
explicitly. Understanding these modules deepens your knowledge of what the Workflow Manager
actually orchestrates under the hood, and is essential if you need fine-grained control (e.g.
per-device targeting, custom scheduling, or integration with an external ITSM workflow).

This appendix describes how to rebuild the same pipeline using individual modules while keeping
**the same `vars/images.yml` data model** as the source of truth.

---

### B.1 Module families at a glance

The collection ships two generations of individual SWIM modules:

| Generation | Module pattern | API surface | Status |
|---|---|---|---|
| **Legacy (v1 API)** | `swim_import_via_url`, `swim_trigger_distribution`, `swim_trigger_activation`, `golden_image_create` | `/dna/intent/api/v1/...` | Available; works on 2.3.7.x |
| **Modern (v2 API)** | `images_download`, `images_id_sites_site_id_tag_golden`, `network_device_images_id_distribute`, `network_device_images_id_activate` | `/dna/intent/api/v1/images/...` | Newer endpoint; added in CatC 2.3.7+ |

Both generations require you to **manage ID resolution and task polling yourself** — what
`swim_workflow_manager` absorbs internally you must now write explicitly.

---

### B.2 The additional orchestration burden

With individual modules, every step that the Workflow Manager handles automatically becomes a
task you own:

| Responsibility | Workflow Manager | Individual modules |
|---|---|---|
| Look up `imageId` UUID from image name | ✅ Resolved internally | ❌ Must query `images_info` or `swim_image_details_info` |
| Look up `siteId`, `deviceFamilyIdentifier` | ✅ Resolved internally | ❌ Must query site and family APIs |
| Look up device UUIDs from site / family / series | ✅ Resolved internally | ❌ Must query `network_device_by_ip_info` or `intent_network_devices_query` |
| Submit async task and poll until complete | ✅ Built-in (`catalystcenter_api_task_timeout`) | ❌ Must loop on `task_info` until `endTime` or `isError` |
| Verify state after apply | ✅ `config_verify: true` | ❌ Must re-query and assert manually |
| Idempotency (skip if already done) | ✅ Handled by module | ❌ Must guard with `when:` checks on info-module results |

---

### B.3 Sequence diagram — individual modules

The diagram below maps each pipeline phase to the exact individual module(s) required, including
the lookup and polling steps that the Workflow Manager hides:

```mermaid
sequenceDiagram
    autonumber
    participant PB as Ansible Playbook
    participant CatC as Catalyst Center API
    participant Dev as Devices (Floor 1)

    rect rgb(230,240,255)
        Note over PB,CatC: PHASE 00 — Preflight (resync + IMAGE compliance)
        PB->>CatC: network_devices_resync_interval_settings_override<br/>(per site, force_sync)
        CatC-->>PB: taskId
        PB->>CatC: task_info (poll until endTime set)
        CatC-->>PB: task complete
        PB->>CatC: compliance_device<br/>(deviceId list, categories=[IMAGE])
        CatC-->>PB: taskId
        PB->>CatC: task_info (poll until endTime set)
        PB->>CatC: compliance_device_details_info<br/>(filter by deviceId, complianceType=IMAGE)
        CatC-->>PB: compliance results (status per device)
    end

    rect rgb(255,245,225)
        Note over PB,CatC: PHASE 10 — Import image via URL
        PB->>CatC: swim_import_via_url<br/>(sourceURL from import_images[].import_image_details)
        CatC-->>PB: taskId
        PB->>CatC: task_info (poll until endTime / isError)
        CatC-->>PB: task complete; image now in repository
        PB->>CatC: swim_image_details_info or images_info<br/>(filter by name → resolve imageId UUID)
        CatC-->>PB: imageId
        PB->>CatC: golden_image_create<br/>(imageId, siteId, deviceFamilyIdentifier, deviceRole)<br/>from golden_tag_images[].tagging_details
        CatC-->>PB: golden tag applied
    end

    rect rgb(230,255,235)
        Note over PB,CatC: PHASE 20 — Distribute (install add, no reload)
        PB->>CatC: swim_image_details_info<br/>(resolve imageId by name)
        CatC-->>PB: imageId
        PB->>CatC: intent_network_devices_query or network_device_by_ip_info<br/>(filter by site + family + series → resolve deviceId list)
        CatC-->>PB: [deviceId, …]
        PB->>CatC: swim_trigger_distribution<br/>(payload: [{deviceUuid, imageUuid}] per device)<br/>— OR — network_device_images_id_distribute<br/>(id=deviceId, distributedImages=[imageId])
        CatC-->>PB: taskId
        PB->>CatC: task_info (poll until endTime / isError)
        CatC-->>Dev: install add (no reload)
        Dev-->>CatC: image staged
    end

    rect rgb(255,230,230)
        Note over PB,CatC: PHASE 30 — Activate (install activate+commit, REBOOTS)
        PB->>CatC: swim_image_details_info<br/>(resolve imageId by name)
        CatC-->>PB: imageId
        PB->>CatC: intent_network_devices_query<br/>(filter by site + family + series → deviceId list)
        CatC-->>PB: [deviceId, …]
        PB->>CatC: network_device_images_id_readiness_checks<br/>(id=deviceId, imageId) — pre-activation validation
        CatC-->>PB: readiness status per device
        PB->>CatC: swim_trigger_activation<br/>(payload: [{deviceUuid, imageUuidList, activateLowerImageVersion}])<br/>— OR — network_device_images_id_activate<br/>(id=deviceId, installedImages=[imageId])
        CatC-->>PB: taskId
        PB->>CatC: task_info (poll; timeout ≥ 7200 s — devices reload)
        CatC-->>Dev: install activate + install commit
        Dev-->>CatC: reloaded on new image
        CatC-->>PB: task complete
    end

    rect rgb(230,240,255)
        Note over PB,CatC: PHASE 40 — Postcheck (IMAGE compliance)
        PB->>CatC: compliance_device<br/>(deviceId list, categories=[IMAGE])
        CatC-->>PB: taskId
        PB->>CatC: task_info (poll until endTime)
        PB->>CatC: compliance_device_details_info<br/>(filter complianceType=IMAGE)
        CatC-->>PB: Compliant: 6 / 6
    end
```

---

### B.4 Phase-by-phase module reference

#### Phase 00 — Preflight

| Step | Module | Key parameters | Notes |
|---|---|---|---|
| Force device resync | `cisco.catalystcenter.network_devices_resync_interval_settings_override` | `id` (device UUID) | No direct `site_name` input — must resolve device UUIDs from site first using `network_device_by_ip_info` or `intent_network_devices_query` |
| Poll resync task | `cisco.catalystcenter.task_info` | `task_id` | Loop with `until: result.dnacResponse.endTime is defined` |
| Trigger IMAGE compliance | `cisco.catalystcenter.compliance_device` | `deviceUuids: [...]`, `complianceType: IMAGE` | Takes a list of device UUIDs — requires lookup first |
| Poll compliance task | `cisco.catalystcenter.task_info` | `task_id` | |
| Retrieve results | `cisco.catalystcenter.compliance_device_details_info` | `deviceUuid`, `complianceType: IMAGE` | Parse `status` field per device |

#### Phase 10 — Import + golden tag

| Step | Module | Key parameters | Notes |
|---|---|---|---|
| Import image from URL | `cisco.catalystcenter.swim_import_via_url` | `payload: [{sourceURL, urlType: GITHUB/HTTP}]` | `sourceURL` from `import_images[].import_image_details.url_path` |
| Poll import task | `cisco.catalystcenter.task_info` | `task_id` | Import can take several minutes |
| Resolve `imageId` by name | `cisco.catalystcenter.swim_image_details_info` | `name: "{{ image_name }}"` | Returns UUID needed for golden-tag call |
| Resolve `siteId` | `cisco.catalystcenter.sites_info` or `site_by_name_info` | `name: "{{ site_name }}"` | Returns `siteId` UUID |
| Apply golden tag | `cisco.catalystcenter.golden_image_create` | `imageId`, `siteId`, `deviceFamilyIdentifier`, `deviceRole` | `deviceFamilyIdentifier` = `999999901` for cat9kv; `deviceRole: ALL` |

> **Alternative (v2 API):** `images_id_sites_site_id_tag_golden` — takes `id` (imageId) and
> `siteId` path parameters directly, no `deviceFamilyIdentifier` needed at this layer.

#### Phase 20 — Distribute

| Step | Module | Key parameters | Notes |
|---|---|---|---|
| Resolve `imageId` | `cisco.catalystcenter.swim_image_details_info` | `name: "{{ image_name }}"` | |
| Resolve device UUIDs by site/family/series | `cisco.catalystcenter.intent_network_devices_query` | `siteHierarchy`, `family`, `series` | Returns list of device UUIDs to loop over |
| Trigger distribution (legacy) | `cisco.catalystcenter.swim_trigger_distribution` | `payload: [{deviceUuid, imageUuid}]` | Accepts a list — can bulk-submit all 6 devices in one call |
| — or — trigger distribution (modern) | `cisco.catalystcenter.network_device_images_id_distribute` | `id` (deviceId), `distributedImages: [imageId]` | One call per device — loop required |
| Poll distribution task | `cisco.catalystcenter.task_info` | `task_id` | Poll `GET /dna/intent/api/v1/networkDeviceImageUpdates?parentId={taskId}` via `network_device_image_updates_info` for per-device status |

#### Phase 30 — Activate

| Step | Module | Key parameters | Notes |
|---|---|---|---|
| Resolve `imageId` | `cisco.catalystcenter.swim_image_details_info` | `name: "{{ image_name }}"` | |
| Resolve device UUIDs | `cisco.catalystcenter.intent_network_devices_query` | `siteHierarchy`, `family`, `series` | |
| Pre-activation readiness check | `cisco.catalystcenter.network_device_images_id_readiness_checks` | `id` (deviceId), `imageId` | New v2 API — checks flash, NTP, reachability before activating |
| Trigger activation (legacy) | `cisco.catalystcenter.swim_trigger_activation` | `payload: [{deviceUuid, imageUuidList, activateLowerImageVersion: false}]` | Reboots devices |
| — or — trigger activation (modern) | `cisco.catalystcenter.network_device_images_id_activate` | `id` (deviceId), `installedImages: [imageId]` | Combines distribute+activate in one call if needed |
| Poll activation task | `cisco.catalystcenter.task_info` | `task_id` | Extend poll timeout ≥ 7200 s; devices are offline during reload |

#### Phase 40 — Postcheck

Same as Phase 00 compliance steps — `compliance_device` + `task_info` + `compliance_device_details_info`. Assert `status == "COMPLIANT"` for all device UUIDs.

---

### B.5 Mapping `swim_details` keys to individual-module parameters

The `vars/images.yml` `swim_details` keys map to individual-module parameters as follows —
allowing you to reuse the same data model with minimal Jinja2 transformation:

| `swim_details` key | Field | Maps to individual-module parameter |
|---|---|---|
| `import_images[].import_image_details` | `url_path` | `swim_import_via_url` → `payload[].sourceURL` |
| `golden_tag_images[].tagging_details` | `site_name` | → resolve `siteId` via `sites_info` |
| | `device_image_family_name` | → `deviceFamilyIdentifier` in `golden_image_create` (resolve via family-identifier API or hardcode `999999901`) |
| | `device_role` | → `deviceRole` in `golden_image_create` |
| `distribute_images[].image_distribution_details` | `image_name` | → resolve `imageId` via `swim_image_details_info` |
| | `site_name` + `device_family_name` + `device_series_name` | → resolve `deviceId` list via `intent_network_devices_query` |
| `activate_images[].image_activation_details` | `activate_lower_image_version` | → `swim_trigger_activation` payload `activateLowerImageVersion` |
| | `distribute_if_needed` | → only applicable in `network_device_images_id_activate` (`installedImages` combines both) |

---

### B.6 Why this repository uses Workflow Manager instead

| Consideration | Individual modules | `swim_workflow_manager` |
|---|---|---|
| Lines of playbook code | ~150–200 per phase (lookups + polling loops + guards) | ~30 per phase |
| ID resolution | Manual (3–5 extra tasks per phase) | Automatic |
| Task polling | Manual `until:` loop on `task_info` | Built-in |
| Idempotency | Manual `when:` guards on info-module output | Built-in |
| Readability | Low — imperative, stateful | High — declarative config blocks |
| Debuggability | High — every API call visible | Medium — single module call |
| Appropriate when | Fine-grained per-device logic, ITSM integration, custom scheduling | Standard bulk-site upgrade pipeline |

> **Bottom line:** reach for individual modules when you need per-device branching logic,
> dynamic scheduling decisions, or integration with an external state machine. For the standard
> "upgrade this site to this image" use case, `swim_workflow_manager` is always the right choice.

---

## Appendix A: Image distribution (HTTP) server setup

`vars/images.yml` uses `import_image_details.type: remote`, so **Catalyst Center itself** pulls
the `.bin` image over HTTP from the Ubuntu host `198.18.134.28`. The URL must resolve exactly to:

```
http://198.18.134.28/images/cat9kv-universalk9.BLD_V262_THROTTLE_LATEST_20260529_003538.SSA.bin
```

The image must live in an `/images/` web-root directory. The steps below use **nginx**;
Apache or a quick `python3 -m http.server` work too.

### A.1 Install nginx (on 198.18.134.28)

```bash
sudo apt update
sudo apt install -y nginx
systemctl status nginx --no-pager
```

### A.2 Create the image directory and stage the file

```bash
sudo mkdir -p /var/www/html/images
# upload from your workstation:
#   scp cat9kv-universalk9.<release>.SSA.bin user@198.18.134.28:/tmp/
sudo mv /tmp/cat9kv-universalk9.BLD_V262_THROTTLE_LATEST_20260529_003538.SSA.bin \
        /var/www/html/images/
sudo chown -R www-data:www-data /var/www/html/images
sudo chmod 0755 /var/www/html/images
sudo chmod 0644 /var/www/html/images/*.bin
```

### A.3 (Optional) Dedicated server block with directory browsing

```bash
sudo tee /etc/nginx/sites-available/swim-images >/dev/null <<'CONF'
server {
    listen 80 default_server;
    server_name 198.18.134.28;
    root /var/www/html;

    location /images/ {
        autoindex on;
        sendfile on;
        default_type application/octet-stream;
    }
}
CONF
sudo ln -sf /etc/nginx/sites-available/swim-images /etc/nginx/sites-enabled/swim-images
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

### A.4 Open the firewall (if ufw is active)

```bash
sudo ufw status
sudo ufw allow 80/tcp   # only if ufw is active
```

### A.5 Verify

```bash
# on the file server:
curl -I http://localhost/images/cat9kv-universalk9.BLD_V262_THROTTLE_LATEST_20260529_003538.SSA.bin

# from the Catalyst Center side (the route that actually matters):
curl -I http://198.18.134.28/images/cat9kv-universalk9.BLD_V262_THROTTLE_LATEST_20260529_003538.SSA.bin
```

Both must return `HTTP/1.1 200 OK` with a `Content-Length` matching the file size.

> **Gotchas**
> - The URL path must match `vars/images.yml` exactly — wrong subdir or trailing slash fails the import.
> - Serve as `application/octet-stream` so the `.bin` downloads instead of rendering.
> - Use anonymous HTTP (no basic-auth) — CatC remote import expects an unauthenticated URL.
> - Reachability that matters is **CatC → 198.18.134.28:80**, not your workstation; verify with the second `curl`.
> - Confirm integrity after copy: `md5sum /var/www/html/images/*.bin`.
