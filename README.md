# Ansible Role: aide

A lean file integrity check with AIDE. It watches **the system** - programs, libraries,
configuration, boot files, root's keys and crontabs - not the data and not the services.

Once a week the host compares these files with its baseline and sends **one mail, and only
for changes that no package update explains**. A file that changed because its package was
updated is recognised (its content is exactly what the package ships) and left out. After
each run the new state becomes the baseline, so every change is reported once.

## Why not the distribution's setup

Debian's own AIDE configuration checks far more paths with eight checksums each, every night,
and reports every change of every package update. The result is a long daily report that
nobody reads. This role uses its own small configuration and its own database beside it; the
distribution's files stay untouched, its nightly run is switched off.

## Requirements

- Debian 12 or 13, Ubuntu 22.04, 24.04 or 26.04 - every release of this role is tested on each
  of them in a fresh virtual machine (the role uses `apt` and reads the `dpkg` log)
- root on the target (`become: true`)
- for the weekly mail: the host can send mail with the `mail` command. The role installs no
  mail software.

## Install

```bash
ansible-galaxy role install MojiX-Tech.aide
```

## Example playbook

```yaml
- hosts: all
  become: true
  roles:
    - role: MojiX-Tech.aide
      vars:
        aide_report_email: admin@example.com
```

With options:

```yaml
- hosts: kubernetes_nodes
  become: true
  roles:
    - role: MojiX-Tech.aide
      vars:
        aide_report_email: admin@example.com
        aide_extra_paths:
          - /opt/cni/bin
        aide_extra_excludes:
          - /etc/cni/net\.d
```

## What is watched

`/boot`, `/etc`, `/usr/bin`, `/usr/sbin`, `/usr/lib*`, `/usr/local/bin`, `/usr/local/sbin`,
`/bin`, `/sbin`, `/lib*`, `/root/.ssh`, root's shell start files, `/var/spool/cron`.

Per entry: permissions, owner, group, file type, link target, capabilities and the SHA-256 of
the content. Not the time stamps - they change with every update and say nothing the checksum
does not.

Left out are files that change by themselves or are generated from other watched files (for
example `ld.so.cache`, `initrd`, Let's Encrypt renewals, `known_hosts`). The full lists are in
[`defaults/main.yml`](defaults/main.yml).

## Variables

| Variable | Default | Meaning |
|---|---|---|
| `aide_report_email` | `root` | receiver of the weekly mail |
| `aide_report_enabled` | `true` | weekly run on or off |
| `aide_report_weekday` / `_hour` / `_minute` | Sunday, 11, spread per host | when the weekly run starts |
| `aide_extra_paths` | `[]` | more paths to watch (regular expressions) |
| `aide_extra_excludes` | `[]` | more paths to leave out |
| `aide_paths` / `aide_excludes` | see defaults | the baseline lists themselves |
| `aide_attributes` | `p+u+g+ftype+l+caps+sha256` | what is compared per entry |
| `aide_db_keep` | `3` | older baselines kept beside the current one |
| `aide_run_timeout` | `2h` | a run is stopped after this time |
| `aide_disable_distribution_check` | `true` | switches the distribution's nightly AIDE run off |
| `aide_config` / `aide_db` / `aide_report_log` | `/etc/aide/aide-lean.conf`, `/var/lib/aide/aide-lean.db.gz`, `/var/log/aide/aide-lean-report.log` | where the role keeps its files |

## The weekly mail

`/usr/local/bin/aide-report` runs from root's crontab with the lowest CPU and disk priority
and stays silent when nothing notable happened. Subjects:

- `[AIDE] Weekly Report - <host>` - files were added, removed or changed and no package
  update explains it. The mail lists them; the details per file (attributes, checksums) stay
  in the report log on the host.
- `[AIDE] FAILED - <host>` - AIDE could not finish; nothing was compared.

By hand:

```bash
aide-report            # the weekly run, now
aide-report --init     # new baseline without a comparison, no mail
```

## Good to know

- The role writes a baseline at the end of its first run, and a new one whenever the watched
  paths or excludes changed.
- An Ansible run that changes files under `/etc` or `/usr/local/bin` shows up in the next
  mail of that host. That is correct - it is a change no package explains.
- The baseline lives on the same host. Someone with root can rewrite it; the role detects
  changes, it is not tamper-proof storage.

## Tags

`aide_install`, `aide_config`, `aide_report`

## Licence

Apache License 2.0 - see [LICENSE](LICENSE).

## Where this role comes from

This repository is written by the releases of
[MojiX-Tech/Automation](https://github.com/MojiX-Tech/Automation), where the role is developed
and tested. Please open issues and pull requests there. The same role is part of the
collection `mojix_tech.automation`.
