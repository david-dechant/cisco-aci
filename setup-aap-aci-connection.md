# Using this repo as a baseline for your own ACI playbooks

This repo is a small, working skeleton for automating Cisco ACI with Ansible: a connection
layer that already authenticates against an APIC, plus two example use cases (`configure_aci.yml`,
`query_aci.yml`) that show the intended pattern. Fork/copy it and extend the pattern rather than
starting from scratch. This guide explains what to reuse as-is, what to change, and how to add
your own objects/roles/playbooks.

## 1. Layout at a glance

```
ansible.cfg                      # inventory path, gathering=explicit
inventory                        # target host(s) + httpapi connection settings (no secrets)
group_vars/all                   # vars shared by every host (ansible_controller)
group_vars/apic                  # credential mapping for the [apic] group (no secrets)
host_vars/apic1                  # per-host business data: tenants, vrfs, bds, epgs, ...
collections/requirements.yml     # cisco.aci, ansible.netcommon, ansible.utils
execution-environment/           # ansible-builder recipe for a custom EE (optional)
roles/configure_application/     # "present/absent" logic: turns host_vars data into API calls
roles/query_apic/                # "query" logic: reads APIC state back into YAML files
verify_connection.yml            # smoke-test playbook: login + tenant query
configure_aci.yml                # thin entrypoint -> roles/configure_application
query_aci.yml                    # thin entrypoint -> roles/query_apic
```

The split that matters is: **connection** (inventory + `group_vars`) is generic and you should
rarely touch it, **data** (`host_vars`) is your source of truth and changes constantly, and
**logic** (`roles/*/tasks`) is the reusable mapping from data to `cisco.aci.*` modules. Top-level
playbooks stay thin — just `hosts:` + a role list.

## 2. Keep this part as-is (the connection layer)

Don't rebuild authentication for each new playbook — every playbook that targets `hosts: apic`
already inherits it:

```5:11:inventory
[apic:vars]
ansible_connection=ansible.netcommon.httpapi
ansible_network_os=cisco.aci.aci
ansible_httpapi_use_ssl=true
ansible_httpapi_validate_certs=false
ansible_httpapi_port=443
```

```1:5:group_vars/apic
# httpapi authenticates with ansible_user / ansible_password.
# Map AAP custom-credential env injectors (ACI_USERNAME / ACI_PASSWORD)
# onto those vars. A Network credential sets ansible_* as extra vars and wins.
ansible_user: "{{ lookup('env', 'ACI_USERNAME') | default('admin', true) }}"
ansible_password: "{{ lookup('env', 'ACI_PASSWORD') | default('', true) }}"
```

Rules to carry forward into anything you add:
- Never put a real password in `inventory`, `group_vars`, or `host_vars` — only in an AAP
  credential or a local env var/vault.
- Any new role should assume `ansible_user`/`ansible_password` are already set by the time its
  tasks run; don't re-authenticate per task.
- `group_vars/all` holds `ansible_controller: localhost`, used by `delegate_to` when a task needs
  to write a local file (see `roles/query_apic`). Reuse that variable for the same purpose instead
  of hardcoding a host.

## 3. The data/logic pattern

`host_vars/apic1` is the source of truth: a plain list-of-dicts per object type.

```1:13:host_vars/apic1
---
# Default target is the private homelab APIC simulator (single tenant, no
# sharing conflicts). If you point ansible_host at the DevNet Always-On
# sandbox instead, change tenant_name (and every tenant: reference below) to
# something unique per team, or you will overwrite other users' data.
tenants:
  - tenant_name: production
    description: 'Production Tenant'
    state: present

vrfs:
  - vrf_name: vrf1
    description: 'Production VRF'
```

`roles/configure_application/tasks/main.yml` loops over each list and maps it 1:1 onto a
`cisco.aci.*` module, tagged both individually and as `all`:

```1:9:roles/configure_application/tasks/main.yml
- name: Configure Tenant
  cisco.aci.aci_tenant:
    tenant: "{{ item.tenant_name }}"
    description: "{{ item.description }}"
    state: "{{ item.state }}"
    output_level: debug  # httpapi plugin enables httpapi_logs for debugging
  loop: "{{ tenants }}"
  tags: [tenants, all]
```

Parent/child relationships (e.g. subnets under a bridge domain, filter entries under a filter)
use `subelements` instead of a separate top-level list:

```30:40:roles/configure_application/tasks/main.yml
- name: Configure Subnet
  cisco.aci.aci_bd_subnet:
    tenant: "{{ item.0.tenant }}"
    bd: "{{ item.0.bd_name }}"
    subnet_name: "{{ item.1.subnet_name }}"
    gateway: "{{ item.1.gateway }}"
    mask: "{{ item.1.mask }}"
    scope: "{{ item.1.scope }}"
    state: "{{ item.1.state }}"
  loop: "{{ bridge_domains | subelements('subnets') }}"
  tags: [subnets, all]
```

Every object also carries its own `state: present|absent` so the same data drives both creation
and teardown, and a `tenant`/`tenant_name` value that must match across lists — that string is
how ACI (and this role) ties objects together, not YAML nesting.

## 4. Recipe: adding a new ACI object type to the existing role

Say you want to manage `cisco.aci.aci_l3out` in addition to what's here.

1. **Add data** to `host_vars/apic1` (or a new `host_vars`/`group_vars` file if it should apply
   more broadly):
   ```yaml
   l3outs:
     - l3out_name: l3out1
       tenant: production
       vrf: vrf1
       domain: l3_dom1
       state: present
   ```
2. **Add a task** to `roles/configure_application/tasks/main.yml` following the existing shape —
   loop over the new list, map each field to the module's parameters, give it its own tag plus
   `all`:
   ```yaml
   - name: Configure L3Out
     cisco.aci.aci_l3out:
       tenant: "{{ item.tenant }}"
       l3out: "{{ item.l3out_name }}"
       vrf: "{{ item.vrf }}"
       domain: "{{ item.domain }}"
       state: "{{ item.state }}"
     loop: "{{ l3outs }}"
     tags: [l3outs, all]
   ```
3. If the new object needs to reference an existing APIC-populated attribute (rather than data
   you defined), do the query first — see the pattern in `roles/query_apic` — and pass its output
   forward as a fact.
4. **Test with tags** before running everything: `ansible-playbook configure_aci.yml --tags l3outs`
   (or the equivalent Job Template with a tag limited on launch).
5. If you also want it queryable, add a matching task to `roles/query_apic/tasks/main.yml` using
   `state: query` + `json_query`, following the same tenant-name filter used for the existing
   objects (see below).

## 5. Recipe: adding a brand-new role/playbook (a new use case)

For something that isn't "configure" or "query" — e.g. a compliance-check role — don't bolt it
onto the existing roles:

1. `roles/<new_role_name>/tasks/main.yml` — write the task logic there.
2. A thin top-level playbook, matching the existing style:
   ```yaml
   ---
   - name: <Describe the use case>
     hosts: apic
     roles:
       - <new_role_name>
   ```
3. If you need somewhere read-only to sanity-check connectivity/data before wiring up real
   logic, model it on `verify_connection.yml` — it's intentionally the smallest possible playbook:
   ```1:15:verify_connection.yml
   ---
   - name: Verify APIC connectivity
     hosts: apic
     gather_facts: false

     tasks:
       - name: Query tenants from APIC
         cisco.aci.aci_tenant:
           state: query
         register: tenant_query

       - name: Show APIC login succeeded
         ansible.builtin.debug:
           msg: >-
             Connected to {{ ansible_host }}.
             Tenant count: {{ tenant_query.current | length }}
   ```
4. If the new role writes files back to a controller/local machine (like `query_apic` does),
   reuse `delegate_to: "{{ ansible_controller }}"` rather than hardcoding `localhost`.

## 6. Where to put new variables

| Scope | File | Example |
| --- | --- | --- |
| Applies to every host in inventory | `group_vars/all` | `ansible_controller` |
| Applies to the whole `apic` group (connection/auth) | `group_vars/apic` | credential mapping |
| Specific to one APIC host (your actual config data) | `host_vars/<hostname>` | `tenants`, `vrfs`, `l3outs`, ... |
| Multiple APIC hosts, shared business data | new `group_vars/<groupname>` | a group var file for a new inventory group |

If you add a second APIC target, add it to `inventory` under `[apic]` (or a new group) and give
it its own `host_vars/<hostname>` file — the role logic doesn't change, only the data.

## 7. Querying the objects you add

`roles/query_apic` mirrors the configure side: `state: query`, extract with `json_query`, filter
by tenant name, write to `/tmp/aci_data/<type>.yml` on `ansible_controller`:

```9:27:roles/query_apic/tasks/main.yml
- name: Query Tenants
  cisco.aci.aci_tenant:
    state: query
  register: tenant_data
  tags: [tenants, all]

- name: Extract Tenants data
  ansible.builtin.set_fact:
    tenants: >-
      {{ tenant_data.current |
         json_query("[?contains(fvTenant.attributes.dn,'production')].{
            name: fvTenant.attributes.name,
            description: fvTenant.attributes.descr
            }"
         )
      }}
  tags: [tenants, all]

- name: Write Tenants data to YAML file
  ansible.builtin.copy:
    content: "{{ tenants | to_nice_yaml }}"
    dest: "/tmp/aci_data/tenants.yml"
  delegate_to: "{{ ansible_controller }}"
  tags: [tenants, all]
```

The `'production'` string in every `contains(...)` filter is hardcoded to match the sample
tenant name — if you rename your tenant (see §8), update these filters too, or your queries will
return nothing.

## 8. If you rename the tenant / point at a shared APIC

The DevNet Always-On sandbox is shared. If you target it instead of a private simulator, rename
`tenant_name`/`tenant` everywhere in your `host_vars` file to something unique, and update the
`contains(...,'production')` filters in `roles/query_apic` to match — otherwise you'll either
collide with another team's objects or your queries will silently return empty results.

## 9. Local iteration before pushing to AAP

```bash
export ACI_USERNAME=admin
export ACI_PASSWORD='<your APIC password>'
ansible-galaxy collection install -r collections/requirements.yml   # first time / after changes
ansible-playbook verify_connection.yml                               # confirm auth works
ansible-playbook configure_aci.yml --tags <your_new_tag> --check     # dry-run just your addition
```

Once it works locally, running it through AAP is just wiring: new/changed playbooks need a
Job Template pointed at this Project + the existing SCM Inventory + the existing APIC credential
— no new connection setup is required. See the "Use this repo as an AAP project" section of
`README.md` for that side of things.
