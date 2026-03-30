---
name: ansible
description: Write well-structured Ansible playbooks and roles following idempotency, security hardening, role-based organization, and ansible-lint best practices
---

Analyze the current project and write production-grade Ansible code. If playbooks or roles already exist, review and improve them.

## Requirements

**Inspect the project first:**
- Check for existing playbooks, roles, inventory files, and `ansible.cfg`
- Identify what systems are being configured (web servers, databases, Kubernetes nodes, etc.)
- Detect the target OS/distribution

**Directory structure — always use roles:**

```
.
├── ansible.cfg
├── inventory/
│   ├── production/
│   │   ├── hosts.yml          # Production inventory (YAML format)
│   │   └── group_vars/
│   │       └── all.yml        # Non-sensitive group vars
│   └── staging/
│       └── hosts.yml
├── playbooks/
│   └── site.yml               # Top-level playbook that imports role playbooks
├── roles/
│   └── <role-name>/
│       ├── defaults/
│       │   └── main.yml       # Default variable values (lowest precedence)
│       ├── vars/
│       │   └── main.yml       # Role-internal variables (higher precedence)
│       ├── tasks/
│       │   └── main.yml       # Task entrypoint; use import_tasks for sub-tasks
│       ├── handlers/
│       │   └── main.yml       # Handlers triggered by notify
│       ├── templates/
│       │   └── *.j2           # Jinja2 templates
│       ├── files/             # Static files for copy module
│       └── meta/
│           └── main.yml       # Role dependencies and metadata
└── requirements.yml           # Galaxy role/collection dependencies
```

**Mandatory rules:**

### Idempotency
- Every task must be idempotent — re-running the playbook on a configured system must produce no changes
- Use state-based modules (`apt`, `yum`, `service`, `user`, `file`, `template`) — avoid `command`/`shell` unless there is no module alternative
- When `command`/`shell` is unavoidable, add `changed_when` and `failed_when` conditions

### Task authoring
- Every task must have a descriptive `name` in sentence case
- Add `tags` to every task for selective execution (e.g. `tags: [install, nginx]`)
- Use `no_log: true` on any task that handles passwords, tokens, or secrets
- Use `block`/`rescue`/`always` for error handling in multi-step operations
- Prefer `become: true` at the task or block level — never globally unless all tasks require privilege escalation

### Handlers
- Use handlers for service restarts and reloads — never restart a service directly in a task
- Name handlers descriptively: `Restart nginx`, `Reload systemd daemon`
- Use `listen` for handlers that should be triggered by multiple notifiers

### Variables and secrets
- Define defaults in `defaults/main.yml` — these are overridable
- Store secrets in Ansible Vault — never plaintext in vars files or inventory
- Reference vault-encrypted variables with a `vault_` prefix convention: `vault_db_password`
- Use `vars/main.yml` for role-internal constants that should not be overridden

### Templates (Jinja2)
- Validate template syntax with `ansible-lint` or `--check`
- Always include a comment at the top: `# Managed by Ansible — do not edit manually`
- Use `{{ variable | default('fallback') }}` for optional values

### `ansible.cfg`
```ini
[defaults]
inventory       = inventory/
roles_path      = roles/
stdout_callback = yaml
retry_files_enabled = False
host_key_checking = True

[privilege_escalation]
become          = False
become_method   = sudo
```

### ansible-lint compliance
- No `warn` suppression without a comment explaining why
- No deprecated modules (use FQCN: `ansible.builtin.copy` not `copy`)
- No `ignore_errors: true` — handle errors properly with `block/rescue`

## Output format

Provide all files in full with fenced code blocks labeled by path:

```yaml
# roles/nginx/tasks/main.yml
...
```

After the code, include:
1. A **Run** section: exact `ansible-playbook` commands with `--check` (dry-run) and full-run variants
2. A **Vault** section: commands to create and edit the vault file
3. A **Security notes** section flagging any `become` usage, `no_log` omissions, or shell tasks and why they were necessary
