# Ansible Role: lighthouse-role

Installs nginx and deploys [Lighthouse](https://github.com/VKCOM/lighthouse) —
a web UI for ClickHouse (static assets, no backend required).

## Requirements

- Ansible 2.10+
- Debian/Ubuntu or the RHEL family (nginx is installed with
  `ansible.builtin.package`)
- Internet access to `github.com` (git clone)
- Root privileges (`become: true`)
- `git` on the target host

## Role Variables

| Variable                       | Default                             | Description |
|--------------------------------|-------------------------------------|-------------|
| `lighthouse_folder`            | `"/var/www/Lighthouse"`             | Directory the Lighthouse repository is cloned into; also used as nginx `root`. |
| `lighthouse_nginx_port`        | `85`                                | Port nginx listens on (`listen ... default_server`). |
| `lighthouse_nginx_config_path` | `"/etc/nginx/conf.d/default.conf"`  | Path of the nginx server block file. |

Variables are defined in `defaults/main.yml` and can be overridden from a
playbook, `group_vars`/`host_vars` or `--extra-vars`.

> To avoid overwriting the distribution's default config, set e.g.
> `lighthouse_nginx_config_path: /etc/nginx/conf.d/lighthouse.conf` — but then
> remove the old `default.conf`, otherwise nginx ends up with two
> `default_server` blocks on the same port
> (`[emerg] a duplicate default server`).

## Dependencies

None. nginx is installed by this role.

## Example Playbook

```yaml
- name: Install Lighthouse
  hosts: lighthouse
  become: true
  roles:
    - role: lighthouse-role
      vars:
        lighthouse_nginx_port: 85
        lighthouse_folder: "/var/www/Lighthouse"
```

Using a non-default port:

```yaml
- hosts: lighthouse
  become: true
  roles:
    - role: lighthouse-role
      vars:
        lighthouse_nginx_port: 8080
```

## What the role does

1. `Install Nginx` — installs the `nginx` package.
2. `Clone the Lighthouse repo` — clones
   `https://github.com/VKCOM/lighthouse.git` (`master`) into
   `lighthouse_folder`. `force: true` is used, so any local changes in that
   directory will be overwritten.
3. `Set web root permissions` — sets ownership to `root:root` and permissions to
   `u=rwX,go=rX` recursively (directories `755`, files `644`) so that nginx
   (running as `www-data`) can read the files.
4. `Create nginx configuration file` — renders `templates/nginx.conf.j2` into
   `lighthouse_nginx_config_path` and notifies the handler.

## Handlers

| Handler                | Description |
|------------------------|-------------|
| `Restart nginx server` | `ansible.builtin.service: name=nginx state=restarted`. |

## How it works

Lighthouse is a single-page application made of static files; it needs no
server-side backend — nginx simply serves `index.html` and the assets.
Connections to ClickHouse are made **by the user's browser** over the ClickHouse
HTTP interface (default `http://localhost:8123`). This means:

- ClickHouse must be reachable from the machine where the browser runs;
- if the UI is opened from another host, port `8123` has to be forwarded and
  `access-control-allow-origin` has to be configured on the ClickHouse side.

Post-install check:

```bash
curl -I http://localhost:85/
# HTTP/1.1 200 OK
```

## Known limitations

- The repository is served from the web root as-is, including `.git` — acceptable
  for a lab environment, but for production copy only the static assets.
- `force: true` discards any local modifications in `lighthouse_folder`.
- The role does not configure TLS: HTTP only.

## License

MIT

## Author Information

Mikhail Nikitin
