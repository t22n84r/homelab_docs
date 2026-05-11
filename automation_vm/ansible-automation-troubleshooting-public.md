# Ansible Automation Troubleshooting — Public Notes

## Purpose

Troubleshooting notes for Ansible automation/tooling issues encountered while building the remote access gateway. This is separate from the gateway VM's Ansible project setup.

## Issue: `ansible: command not found`

Cause:

- Ansible was installed inside a Python virtual environment, not globally.

Fix pattern:

```bash
source /home/automation/venvs/ansible/bin/activate
```

Reusable pattern:

```bash
sudo -u automation -H bash -lc '
source /home/automation/venvs/ansible/bin/activate &&
cd /path/to/ansible/project &&
ansible <target> -m ping
'
```

## Issue: Deprecated YAML Callback

Error:

```text
The 'community.general.yaml' callback plugin has been removed.
```

Old config:

```ini
stdout_callback = yaml
```

Fixed config:

```ini
stdout_callback = ansible.builtin.default
result_format = yaml
```

## Issue: Docker Go Template Conflicts with Ansible/Jinja

Problem command pattern:

```bash
docker volume ls --format '{{.Name}}'
```

Ansible interprets `{{.Name}}` as Jinja and fails.

Fix options:

- Avoid Go templates in ad-hoc Ansible shell commands.
- Use plain output plus grep.
- Put complex commands in a script or playbook template.

Safer example:

```bash
docker volume ls | grep postgres || true
```

## Issue: Remote Tmp Warning

Warning:

```text
Module remote_tmp /root/.ansible/tmp did not exist and was created with a mode of 0700
```

Clean fix:

```yaml
- name: Ensure root Ansible remote tmp directory exists
  ansible.builtin.file:
    path: /root/.ansible/tmp
    state: directory
    owner: root
    group: root
    mode: "0700"
```

## Rule of Thumb: When to Use `sudo -u automation`

Use `sudo -u automation -H bash -lc` when running automation-owned tools:

- `terraform`
- `ansible`
- SSH tests using automation user's keys
- Commands that rely on automation user's venv/config

Do not use it for privileged local file ownership fixes that require root, such as `chown`.

## Useful Command Pattern

```bash
sudo -u automation -H bash -lc '
source /home/automation/venvs/ansible/bin/activate &&
cd /srv/automation/projects/ansible/<project> &&
ansible-playbook playbooks/<playbook>.yml --vault-password-file /srv/automation/secrets/ansible-vault.pass
'
```
