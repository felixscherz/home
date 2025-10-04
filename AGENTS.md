# Agent Guidelines for Home Infrastructure Automation

## Build/Lint/Test Commands
- **Lint**: `pre-commit run --all-files` (checks YAML, trailing whitespace, nginx formatting)
- **Syntax check**: `ansible-playbook --syntax-check main.yml`
- **Dry run**: `ansible-playbook --check main.yml`
- **Single playbook test**: `ansible-playbook --check --limit <host> --tags <tag> main.yml`

## Code Style Guidelines

### YAML/Ansible Structure
- Use 2-space indentation for YAML files
- Task names: Capitalized, descriptive sentences
- Host groups: lowercase (e.g., `aws`, `home`)
- Variables: snake_case (e.g., `ansible_become_pass`, `server_pubkey`)

### Modules & Imports
- Prefer `ansible.builtin.*` modules
- Use fully qualified collection names (e.g., `ansible.posix.authorized_key`)
- `vars_files` imports at top of playbooks
- `become: true` for privilege escalation tasks

### Error Handling
- Use `register` for command outputs
- `changed_when` for idempotent tasks
- `failed_when` for conditional failures
- `creates` parameter for file generation tasks

### Jinja2 Templates
- Use `{{ variable }}` syntax
- Include `{{ ansible_managed }}` comment in templates
- Template files end with `.j2`

### Security
- Use Ansible Vault for secrets (`!vault` tag)
- Set appropriate file permissions (`mode: '0600'`)
- Avoid hardcoded credentials in playbooks
