# ansible-windows-banner

Ansible playbook and role to install and configure [Microsoft SystemBanner](https://github.com/awawrzyniak10/SystemBanner) on Windows targets. Designed for use with Ansible Automation Platform (AAP) 2.7, with a survey spec included for Job Template configuration.

SystemBanner displays persistent security classification banners across all monitors on Windows systems, supporting both predefined classification levels (UNCLASSIFIED through TOP SECRET SCI) and fully custom text/color configurations.

## Requirements

- **Ansible**: >= 2.15 / ansible-core 2.16+
- **Collections**: `ansible.windows` >= 2.1.0 (see `collections/requirements.yml`)
- **Target OS**: Windows 10/11, Windows Server 2016/2019/2022/2025
- **Target prerequisites**: .NET Framework 4.7.2+, WinRM configured for Ansible

## Project Structure

```
├── site.yml                          # Main playbook
├── survey_spec.json                  # AAP survey definition (import into Job Template)
├── collections/
│   └── requirements.yml              # Required Ansible collections
└── roles/
    └── system_banner/
        ├── defaults/main.yml         # Default variables
        ├── tasks/main.yml            # Install, configure, and remove tasks
        ├── handlers/main.yml         # Restart handler
        └── meta/main.yml             # Role metadata
```

## Variables

| Variable | Default | Description |
|---|---|---|
| `system_banner_state` | `present` | `present` to install/configure, `absent` to remove |
| `system_banner_version` | `v1.1.0000` | Release version tag |
| `system_banner_msi_source` | GitHub release URL | URL or local/UNC path to the MSI installer |
| `system_banner_classification` | `UNCLASSIFIED` | `UNCLASSIFIED`, `CUI`, `CONFIDENTIAL`, `SECRET`, `TOP SECRET`, `TOP SECRET SCI`, or `CUSTOM` |
| `system_banner_custom_text` | `""` | Banner text (only when classification is `CUSTOM`) |
| `system_banner_position` | `top` | `top` or `top_and_bottom` |
| `system_banner_bg_color` | `#007A33` | Background hex color (only when classification is `CUSTOM`) |
| `system_banner_fg_color` | `#FFFFFF` | Text hex color (only when classification is `CUSTOM`) |

## AAP 2.7 Setup

### 1. Create a Project

- **Source Control Type**: Git
- **Source Control URL**: URL of this repository
- **Update Revision on Launch**: Enabled (recommended)

### 2. Create a Job Template

- **Playbook**: `site.yml`
- **Inventory**: Your Windows hosts inventory
- **Credentials**: Machine credential with WinRM access to targets
- **Enable Survey**: Yes — import `survey_spec.json` or configure manually with the variables above

### 3. Survey Configuration

The survey can be imported from `survey_spec.json` or created manually in the Job Template under **Survey** tab. The survey contains six questions:

#### Question 1 — Banner State

| Field | Value |
|---|---|
| **Name** | Banner State |
| **Description** | Install/configure or remove SystemBanner |
| **Type** | Multiple Choice (single select) |
| **Variable** | `system_banner_state` |
| **Required** | Yes |
| **Default** | `present` |
| **Choices** | `present`, `absent` |

#### Question 2 — Classification Level

| Field | Value |
|---|---|
| **Name** | Classification Level |
| **Description** | Predefined classification marking. Choose CUSTOM to specify your own text and colors. |
| **Type** | Multiple Choice (single select) |
| **Variable** | `system_banner_classification` |
| **Required** | Yes |
| **Default** | `UNCLASSIFIED` |
| **Choices** | `UNCLASSIFIED`, `CUI`, `CONFIDENTIAL`, `SECRET`, `TOP SECRET`, `TOP SECRET SCI`, `CUSTOM` |

#### Question 3 — Custom Banner Text

| Field | Value |
|---|---|
| **Name** | Custom Banner Text |
| **Description** | Text displayed on the banner (only used when Classification Level is CUSTOM) |
| **Type** | Text |
| **Variable** | `system_banner_custom_text` |
| **Required** | No |
| **Default** | *(empty)* |
| **Min Length** | 0 |
| **Max Length** | 1024 |

#### Question 4 — Banner Position

| Field | Value |
|---|---|
| **Name** | Banner Position |
| **Description** | Display the banner at the top of the screen only, or at both top and bottom |
| **Type** | Multiple Choice (single select) |
| **Variable** | `system_banner_position` |
| **Required** | Yes |
| **Default** | `top` |
| **Choices** | `top`, `top_and_bottom` |

#### Question 5 — Background Color

| Field | Value |
|---|---|
| **Name** | Background Color |
| **Description** | Banner background color as hex code (only used when Classification Level is CUSTOM) |
| **Type** | Text |
| **Variable** | `system_banner_bg_color` |
| **Required** | No |
| **Default** | `#007A33` |
| **Min Length** | 4 |
| **Max Length** | 7 |

#### Question 6 — Text Color

| Field | Value |
|---|---|
| **Name** | Text Color |
| **Description** | Banner text/foreground color as hex code (only used when Classification Level is CUSTOM) |
| **Type** | Text |
| **Variable** | `system_banner_fg_color` |
| **Required** | No |
| **Default** | `#FFFFFF` |
| **Min Length** | 4 |
| **Max Length** | 7 |

## Usage Examples

### Predefined classification (CLI)

```bash
ansible-playbook site.yml -i inventory \
  -e system_banner_classification="SECRET" \
  -e system_banner_position="top_and_bottom"
```

### Custom banner (CLI)

```bash
ansible-playbook site.yml -i inventory \
  -e system_banner_classification="CUSTOM" \
  -e 'system_banner_custom_text="COMPANY CONFIDENTIAL - INTERNAL USE ONLY"' \
  -e 'system_banner_bg_color=#B22222' \
  -e 'system_banner_fg_color=#FFFFFF'
```

### Remove SystemBanner

```bash
ansible-playbook site.yml -i inventory \
  -e system_banner_state=absent
```

### Internal MSI source

If targets cannot reach GitHub, host the MSI on an internal file share or web server:

```bash
ansible-playbook site.yml -i inventory \
  -e 'system_banner_msi_source=\\\\fileserver\\share\\SystemBannerSetup.msi'
```

## Notes

- Predefined classification levels use built-in colors defined by the SystemBanner application. Custom colors are only applied when classification is set to `CUSTOM`.
- Most configuration changes (text, classification level) apply automatically while the banner is running. Color changes require a process restart — the handler stops the process and it restarts automatically at next user logon via the autostart registry entry.
- The MSI installer deploys Group Policy ADMX/ADML templates to `C:\Windows\PolicyDefinitions\`, enabling configuration via Local Group Policy Editor as well.
