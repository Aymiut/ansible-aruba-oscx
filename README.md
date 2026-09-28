# Automating an Aruba OS-CX switch fleet with Ansible

Configure about twenty Aruba OS-CX switches from an inventory: VLANs, LACP link aggregation
(LAGs), VSF stacking and software version reporting.

**This repository is a generic reconstruction.** The original project was carried out at SPIE ICS
for a healthcare facility: that code and data belong to the client and cannot be published. Here,
everything is fictional: twenty example switches, addresses in `10.50.10.0/24`, credentials in an
encrypted file. What is real is the project structure, the tasks and the pitfalls hit in
production.

## What it does

| Playbook       | Effect                                                                            |
| -------------- | --------------------------------------------------------------------------------- |
| `versions.yml` | Read-only: collects the software version of each switch and flags mismatches.     |
| `vlans.yml`    | Creates the VLANs declared in `group_vars` across the whole fleet.                |
| `lags.yml`     | Creates the LACP LAGs on the switches that declare some, and assigns their VLANs. |
| `vsf.yml`      | Declares the VSF stack member and its links, through the CLI.                     |
| `site.yml`     | Version check, then VLANs, then LAGs.                                             |

## Structure

```
inventory/
├─ hosts.yml                   20 switches: core, aggregation, access
├─ group_vars/aoscx/
│  ├─ main.yml                 connection, default credentials, fleet VLANs, expected version
│  └─ vault.yml.example        template for the encrypted file (the real one is not versioned)
└─ host_vars/
   ├─ sw-acc-02.yml            LAG ports and stack member for this switch
   ├─ sw-acc-03.yml            credentials for the legacy batch
   └─ …                        one file per switch
roles/aoscx/tasks/             vlans, lags, vsf, versions
playbooks/                     one playbook per operation, plus site.yml
tests/                         check of the LAG guard, no hardware needed
```

The fleet has two sets of credentials: the one in `group_vars/aoscx/main.yml` applies everywhere,
and the few switches from the oldest batch override it in their `host_vars`. A switch that declares
neither a LAG nor a stack is left untouched by the corresponding tasks.

## Usage

```bash
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml

cp inventory/group_vars/aoscx/vault.yml.example inventory/group_vars/aoscx/vault.yml
ansible-vault encrypt inventory/group_vars/aoscx/vault.yml

ansible-playbook playbooks/versions.yml --ask-vault-pass          # read-only
ansible-playbook playbooks/site.yml --check --ask-vault-pass      # dry run
ansible-playbook playbooks/site.yml --ask-vault-pass              # apply
```

## Lessons from production

- **The stack member number.** In a stack, a port is written `member/module/port`. Targeting the
  wrong member raises no error: the playbook simply reconfigures a switch that is already in
  service. So the role refuses to apply if a port doesn't match the member declared in
  `host_vars`. This is what `tests/stack-member-guard.yml` tests.
- **Aruba ports are administratively `down` by default.** A LAG gets created without any error and
  never comes up. The tasks explicitly enable the member ports, then the LAG.
- **Changing the management address cuts the current session.** There is no module for this
  operation, and the CLI connection drops out from under you. Renumbering is not in this
  repository: in the field, it is done by keeping the old address on another VLAN interface during
  the cutover.
- **Check version compatibility first.** An `ansible-core` that is too recent for the collection
  makes the connection fail with no clear error message. This blocked the original project for
  several weeks, while the hypervisor or Python looked like the obvious suspects.
- **`--check` before every apply.** Across a fleet, a mistake spreads as fast as a good
  configuration.

## Limitations

- VSF stacking can't be tested in simulation: the EVE-NG switch images don't support it.
- This generic version has not been run against real hardware. The CI checks the inventory, the
  playbook syntax and the LAG guard, without hardware.
