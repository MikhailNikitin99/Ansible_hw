# Ansible Role: vector-role

Installs and configures [Vector](https://vector.dev/) — an agent for collecting,
transforming and shipping logs and metrics.

The role downloads the official static (musl) release from GitHub, places the
binary into `vector_install_dir`, renders the configuration from a template and
starts the process.

## Requirements

- Ansible 2.10+ (the role uses FQCN module names, `ansible.builtin.*`)
- Linux x86_64 (the download URL is pinned to `x86_64-unknown-linux-musl`)
- Internet access to `github.com` (release download)
- Root privileges on the target host (`become: true`)
- `tar` on the target host (used by `ansible.builtin.unarchive`)

## Role Variables

| Variable             | Default            | Description |
|----------------------|--------------------|-------------|
| `vector_version`     | `"0.58.0"`         | Vector version. Used in the release URL and in temporary file names. |
| `vector_install_dir` | `"/usr/local/bin"` | Directory the `vector` binary is installed into. |
| `vector_config_dir`  | `"/etc/vector"`    | Configuration directory; `vector.yaml` is rendered here. |

All variables live in `defaults/main.yml`, so they can be overridden from a
playbook, `group_vars`/`host_vars` or `--extra-vars`.
`vars/main.yml` is intentionally empty — the role has no platform-specific
constants.

## Dependencies

None.

## Example Playbook

```yaml
- name: Install Vector
  hosts: vector
  become: true
  roles:
    - role: vector-role
      vars:
        vector_version: "0.58.0"
        vector_config_dir: "/etc/vector"
```

Installing a different version:

```yaml
- hosts: vector
  become: true
  roles:
    - role: vector-role
      vars:
        vector_version: "0.58.0"
```

## What the role does

1. `Get Vector version` — runs `vector --version`; if the binary is already
   present, the installation is skipped (the role is idempotent for installs).
2. `Download Vector` — downloads
   `vector-{{ vector_version }}-x86_64-unknown-linux-musl.tar.gz` into `/tmp`.
3. `Unzip Vector package` — extracts the archive into `/tmp/vector_extracted`.
4. `Install Vector bin` — copies the binary to `{{ vector_install_dir }}/vector`.
5. `Clean up temp files` — removes the archive and the extraction directory.
6. `Create config dir` — creates `{{ vector_config_dir }}`.
7. `Deploy Vector config from template` — renders `templates/vector.yaml.j2`
   into `{{ vector_config_dir }}/vector.yaml` and notifies the handler.

## Handlers

| Handler          | Description |
|------------------|-------------|
| `Restart Vector` | Runs `pkill -f vector`, then starts `vector --config {{ vector_config_dir }}/vector.yaml` in the background via `nohup`. |

## Default configuration

`templates/vector.yaml.j2` describes a demo pipeline: the `demo_logs` source
(JSON, once per second) feeds the `console` sink.
`data_dir` is a relative path (`var/lib/vector`), so the directory is created
relative to the process working directory.

## Known limitations

- The role **does not create a systemd unit**: the process is started with
  `nohup` and does not survive a host reboot. Production setups need a unit file
  managed by `ansible.builtin.systemd`.
- The `Restart Vector` handler only fires when the configuration changes. If the
  process dies and the config stays untouched, the role will not bring it back.
- Only `x86_64` is supported.

## License

MIT

## Author Information

Mikhail Nikitin
