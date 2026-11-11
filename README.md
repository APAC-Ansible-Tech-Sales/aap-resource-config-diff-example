# AAP Resource Config Diff Example

Example Configuration as Code (CaC) for Ansible Automation Platform (AAP). Desired Controller resources are defined in Git, then applied by a playbook that **creates/updates what is in code** and **removes objects in the target organization that are not**.

This playbook is designed to **run from AAP** (Job Template / Project sync), not only from a laptop. AAP supplies the execution environment, credentials, and Controller API access used by the configuration roles.

## Why `infra.aap_configuration_extended.object_diff`?

[`infra.aap_configuration.dispatch`](https://github.com/redhat-cop/infra.aap_configuration) alone is additive and idempotent: it creates and updates objects defined in your vars, but it does **not** delete Controller objects that exist only in the UI (or were removed from Git).

[`infra.aap_configuration_extended.object_diff`](https://github.com/redhat-cop/aap_configuration_extended/tree/devel/roles/object_diff) closes that gap. It compares the Controller API to your CaC lists and marks orphans with `state: absent` so `dispatch` can remove them.

Together they enforce **desired state**: Git is the source of truth for the scoped organization.

```text
Git (group_vars) ──► object_diff (API vs code) ──► dispatch (apply present + absent)
```

### Advantages

| Advantage | What it means in practice |
| --- | --- |
| **True desired state** | Objects removed from code are removed from AAP (within the scoped org). |
| **Drift control** | Manual UI changes that create unmanaged resources are cleaned up on the next run. |
| **Org-scoped safety** | Diff queries are limited by `orgs` (this example uses `Engineering`), so other orgs such as `Default` are left alone for org-scoped types. |
| **Works with tags** | Run only the object types you manage (`--tags controller_projects`, `controller_job_templates`, etc.). |
| **Pairs with idempotent apply** | Unchanged objects report `changed=0`; only creates, updates, and deletes from the plan are applied. |
| **Fits AAP automation** | The same playbook can be scheduled or triggered from AAP like GitOps for Controller objects. |

With `include_present_state: true`, the post-diff lists include both objects to keep/create (`state: present`) and orphans to delete (`state: absent`).

## How this example is set up

| Piece | Role |
| --- | --- |
| `playbooks/create_resources.yml` | Loads vars, runs `object_diff`, then `dispatch` |
| `playbooks/group_vars/localhost/` | CaC definitions (projects, credentials, inventories, job templates, vault) |
| `collections/requirements.yml` | `infra.aap_configuration` and `infra.aap_configuration_extended` |
| `orgs: Engineering` | Organization scope for API comparison and apply |
| `protect_not_empty_orgs: true` | Extra guard if organization diff is ever enabled |

### Important behaviours

- **Empty CaC list for a managed type** = “delete everything of that type in the org.” Only tag/dispatch types you fully define in code.
- **Some types are not org-scoped** (for example credential types, instance groups, organizations). Be careful enabling those tags against a shared platform.
- **`object_diff` is presence/absence**, not a field-level report. Attribute changes (description, SCM URL, etc.) are handled by the individual Controller modules during `dispatch`.

## Running from AAP

Create the bootstrap resources below in AAP (UI or CaC) **before** launching this playbook. YAML follows the [infra.aap_configuration roles](https://github.com/redhat-cop/infra.aap_configuration/tree/devel/roles) data model — only fields required to recreate this example are included. Replace secret placeholders with vaulted values or AAP credential inputs.

Suggested create order: **credentials → organizations (galaxy attach) → inventory → project → job template**.

### 1. Credentials

```yaml
---
controller_credentials:
  - name: RedHat Console Hub
    organization: Default
    credential_type: Ansible Galaxy/Automation Hub API Token
    inputs:
      url: https://console.redhat.com/api/automation-hub/
      auth_url: https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token
      token: "{{ hub_token }}"

  - name: AAP Instance
    organization: Default
    credential_type: Red Hat Ansible Automation Platform
    inputs:
      host: https://aap.example.org
      username: admin
      password: "{{ aap_password }}"

  - name: AAP Resource Vault
    credential_type: Vault
    inputs:
      vault_password: "{{ vault_password }}"
```

`AAP Instance` is what the job uses to talk to Controller/Gateway APIs.  
`AAP Resource Vault` unlocks `playbooks/group_vars/localhost/vault.yml`.  
`RedHat Console Hub` is attached to orgs so project sync can pull Galaxy collections from `collections/requirements.yml`.

### 2. Organizations

`Default` usually already exists. Ensure `Engineering` exists (this playbook sets `orgs: Engineering`) and both orgs can use the Hub credential.

```yaml
---
aap_organizations:
  - name: Default
    description: The default organization for Ansible Automation Platform
    default_environment: Default execution environment
    galaxy_credentials:
      - RedHat Console Hub

  - name: Engineering
    galaxy_credentials:
      - RedHat Console Hub
```

On AAP 2.5+, prefer [`gateway_organizations`](https://github.com/redhat-cop/infra.aap_configuration/tree/devel/roles/gateway_organizations) to create orgs; `galaxy_credentials` / `default_environment` are still applied via the organizations role fields above.

### 3. Inventory

The bootstrap job template targets Controller APIs from the EE (`connection: local` in the playbook). A simple inventory is enough (Demo Inventory works if you already have it).

```yaml
---
controller_inventories:
  - name: Demo Inventory
    organization: Default
```

### 4. Project

Point this at the Git repo that contains this example. Collections install from `collections/requirements.yml` on project update.

```yaml
---
controller_projects:
  - name: AAP Resources
    organization: Default
    scm_type: git
    scm_url: https://github.com/APAC-Ansible-Tech-Sales/aap-resource-config-diff-example.git
    scm_branch: main
    scm_clean: true
    scm_update_on_launch: true
    allow_override: true
    default_environment: Default execution environment
```

### 5. Job Template

```yaml
---
controller_templates:
  - name: AAP Resources
    organization: Default
    job_type: run
    inventory: Demo Inventory
    project: AAP Resources
    playbook: playbooks/create_resources.yml
    scm_branch: main
    execution_environment: Default execution environment
    credentials:
      - AAP Instance
      - AAP Resource Vault
    ask_scm_branch_on_launch: true
    ask_tags_on_launch: true
    ask_job_type_on_launch: true
    job_tags: >-
      controller_projects,controller_credentials,controller_inventories,controller_job_templates
```

Launch notes:

- Keep **Privilege Escalation** off unless your EE requires it.
- Use **Prompt on launch** for tags (as above) so you can limit object types while only those types are fully defined in Git.
- Auth for the configuration roles comes from the **AAP Instance** credential on the job (or equivalent `CONTROLLER_*` / `aap_*` env vars in the EE).

## References

- [object_diff role](https://github.com/redhat-cop/aap_configuration_extended/tree/devel/roles/object_diff)
- [infra.aap_configuration roles](https://github.com/redhat-cop/infra.aap_configuration/tree/devel/roles)
- [aap_configuration_extended](https://github.com/redhat-cop/aap_configuration_extended)
