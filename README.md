# Ansible Role: SonarQube

Runs SonarQube Server with PostgreSQL on Docker Compose behind Nginx, and
manages its admin settings through the Web API.

- Sets the kernel parameters the embedded Elasticsearch needs.
- Publishes SonarQube on `127.0.0.1` only, with Nginx (optionally TLS) as
  the entry point.
- Upgrades by changing `sonarqube_version`: the containers are recreated,
  the database is migrated and the role waits until SonarQube is `UP`.
- Optionally authenticates against LDAP or Active Directory, including
  group sync.
- Sets the admin password, server base URL, groups, permission templates,
  quality gates and quality profiles.

## Requirements

- Ubuntu 24.04 or EL 9, tested with Molecule on both.
- ansible-core 2.17 or later, the minimum for `community.docker` 5.
- Docker Engine with the Compose plugin, for example from
  [geerlingguy.docker](https://github.com/geerlingguy/ansible-role-docker).
  The role stops early if `docker compose version` fails.
- The `ansible.posix` and `community.docker` collections.
- About 8 GB of RAM for SonarQube and PostgreSQL.

## Role Variables

Only `sonarqube_db_password` is required. See
[defaults/main.yml](defaults/main.yml) for every variable and its default.

### SonarQube and PostgreSQL

| Variable | Default | Description |
| --- | --- | --- |
| `sonarqube_version` | `26.10.0.133078` | Image version; change it to upgrade |
| `sonarqube_edition` | `community` | Image tag suffix |
| `sonarqube_install_dir` | `/opt/sonarqube` | Holds `docker-compose.yml` and `.env` |
| `sonarqube_listen_address` | `127.0.0.1` | Host address SonarQube is published on |
| `sonarqube_port` | `9000` | Host port SonarQube is published on |
| `sonarqube_env` | `{}` | Extra container environment, such as `SONAR_WEB_JAVAADDITIONALOPTS` |
| `sonarqube_db_image` | `postgres:17` | PostgreSQL image |
| `sonarqube_db_data_dir` | `/var/lib/postgresql/data` | Use `/var/lib/postgresql` for postgres 18+ |
| `sonarqube_db_password` | `""` | Required; must not contain a single quote |

### Nginx

| Variable | Default | Description |
| --- | --- | --- |
| `sonarqube_nginx_proxy` | `true` | Install and configure Nginx |
| `sonarqube_server_name` | `sonarqube.example.com` | Nginx `server_name` |
| `sonarqube_ssl_enabled` | `false` | Serve HTTPS and redirect HTTP to it |
| `sonarqube_ssl_cert` | `example.com.crt` | Certificate with its full chain, looked up like a `copy` src |
| `sonarqube_ssl_key` | `example.com.key` | Private key; may be ansible-vault encrypted |
| `sonarqube_client_max_body_size` | `100m` | Largest analysis report a scanner can upload |

### LDAP

| Variable | Default | Description |
| --- | --- | --- |
| `sonarqube_ldap_enabled` | `false` | Authenticate users against LDAP |
| `sonarqube_ldap_url` | `""` | For example `ldap://dc.example.com:3268` |
| `sonarqube_ldap_bind_dn` | `""` | Account used to search the directory |
| `sonarqube_ldap_bind_password` | `""` | Written to the `0600` `.env`; must not contain a single quote |
| `sonarqube_ldap_user_base_dn` | `""` | Where users are searched |
| `sonarqube_ldap_user_request` | `(&(objectClass=user)(sAMAccountName={login}))` | `{login}` is what people type on the login page |
| `sonarqube_ldap_group_base_dn` | `""` | Set it to sync group memberships at login |
| `sonarqube_ldap_group_request` | `(&(objectClass=group)(member={dn}))` | Group search filter |
| `sonarqube_ldap_group_id_attribute` | `sAMAccountName` | Must match the SonarQube group name |
| `sonarqube_ldap_ignore_startup_failure` | `true` | Start SonarQube even when LDAP is unreachable |

The defaults suit Active Directory. For OpenLDAP, use filters such as
`(&(objectClass=inetOrgPerson)(uid={login}))` and
`(&(objectClass=groupOfNames)(member={dn}))`.

### Admin Settings

These need `sonarqube_admin_password`, which replaces the built-in
`admin/admin` password on the first run. It must have at least 12
characters with an uppercase letter, a lowercase letter, a digit and a
special character.

```yaml
sonarqube_admin_password: "{{ vault_sonarqube_admin_password }}"

# Created if missing; listed global permissions are added, never removed
sonarqube_groups:
  - name: sonar-admins
    permissions: [admin]

# Each listed template keeps exactly these permissions
sonarqube_permission_templates:
  - name: company-default
    default: true
    groups:
      sonar-users: [user, codeviewer]
      sonar-admins: [admin]
    project_creator: [admin]

# Each listed gate keeps exactly these conditions
sonarqube_quality_gates:
  - name: Company way
    default: true
    conditions:
      - {metric: new_coverage, op: LT, error: "80"}
      - {metric: new_duplicated_lines_density, op: GT, error: "3"}

# Extends Sonar way unless parent is set, e.g. to "Sonar way core" on 26.10+;
# list only the rules that differ
sonarqube_quality_profiles:
  - name: Company way
    language: java
    default: true
    activate: [java:S1176]
    deactivate: [java:S106]
```

`sonarqube_url` (`https://` or `http://` plus `sonarqube_server_name`)
becomes the server base URL used in email and pull request links.

Quality profiles extend their parent as
[SonarSource recommends](https://docs.sonarsource.com/sonarqube-server/latest/quality-standards-administration/managing-quality-profiles/creating-a-quality-profile/),
so new rules arrive with SonarQube upgrades.

## Dependencies

None. Install Docker before this role runs.

## Example Playbook

```yaml
- hosts: sonarqube
  become: true
  roles:
    - geerlingguy.docker
    - role: zhan9san.sonarqube
      vars:
        sonarqube_db_password: "{{ vault_sonarqube_db_password }}"
        sonarqube_admin_password: "{{ vault_sonarqube_admin_password }}"
        sonarqube_server_name: sonarqube.example.com
        sonarqube_ssl_enabled: true
        sonarqube_ssl_cert: example.com.crt
        sonarqube_ssl_key: example.com.key
```

## Tags

| Tag | Runs |
| --- | --- |
| `install` | Kernel parameters, Docker Compose stack, upgrades |
| `admin` | Admin password, base URL, groups, templates, gates and profiles |
| `nginx` | Nginx and the TLS certificate |

## Upgrading

Change `sonarqube_version` and run the role. Database migrations cannot be
rolled back, so back up the database or snapshot its volume first.

## Testing

The Molecule scenario installs the previous SonarQube release, upgrades it
to the default one, checks that a second run changes nothing, and verifies
Nginx, LDAP against a local OpenLDAP and every admin setting.

```bash
sudo sysctl -w vm.max_map_count=524288
pip install ansible molecule-plugins[docker]
MOLECULE_DISTRO=ubuntu2404 molecule test
MOLECULE_DISTRO=rockylinux9 molecule test
```

## Limitations

- Group permissions are only added. Permission templates and quality gates
  are synced exactly, but removing one from the list does not delete it.
- Group names need at least 3 characters, the minimum the lookup API
  searches for.
- Reading a group's or template's current permissions uses internal Web
  API endpoints, which SonarQube may change without notice.

## License

MIT
