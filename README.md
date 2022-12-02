# Webarchitects systemd Ansible role

[![pipeline status](https://git.coop/webarch/systemd/badges/main/pipeline.svg)](https://git.coop/webarch/systemd/-/commits/main)

`systemd` is a [System and Service Manager](https://systemd.io/) that _"runs as PID 1 and starts the rest of the system"_.

This repo contains an Ansible role for configuring systemd services on Debian, this role has been designed to be as generic as possible in order to enable to it be used to configure any systemd service, by default it configures `systemd-timesyncd`.

On Debian Buster [backports](https://backports.debian.org/Instructions/) is required to get the [latest version of systemd](https://packages.debian.org/buster-backports/systemd), the [Webarchitects apt role](https://git.coop/webarch/apt) can be used to enable backports.

## Role variables

See the [defaults/main.yml](defaults/main.yml) file for the default variables and [meta/argument_spacs.yml](meta/argument_specs.yml) for the variable specification.

### systemd

Set the `systemd` variable to `false` to prevent any tasks in this role being run.

The `systemd` variable defaults to `true`.

### systemd_timesyncd_reboot

When the `systemd_timesyncd_reboot` variable is set to `true` servers which have incorrect clocks will be rebooted by this role in order to correct their clocks.

The `systemd_timesyncd_reboot` variable defaults to `false`.

### systemd_tz

The time zone to be used (the hardware clock is set to UTC), for example `Europe/London`.

The `systemd_tz` variable defaults to `Etc/UTC`.

### systemd_units

`systemd_units` is a list of systemd units to configure, for example:

```yaml
systemd_units:
  - name: systemd-timesyncd
    files:
      - path: /etc/systemd/timesyncd.conf
        comment: |
          Entries in this file show the compile time defaults.
          You can change settings by editing this file.
          Defaults can be restored by simply deleting this file.
          See timesyncd.conf(5) for details.
        conf:
          Time:
            NTP: 0.pool.ntp.org 1.pool.ntp.org 3.pool.ntp.org 2.pool.ntp.org
        state: present
    pkgs:
      - systemd-timesyncd
    state: present
    unit_state: started
```

#### name

The only required variables is `name`, see the [meta/argument_specs.yml](meta/argument_specs.yml) for the variable types.

#### files

The `files` variable is a list of files, which can use the following variables:

##### path

The `path` variable is the path to the file to configure.

##### comment

The `comment` variable can be used for commented text at the top of the file.

##### conf

The `conf` dictionary defines the systemd `ini` file variables, as a dictionary. The depth of the dictionary defines the type of file generated, for example to generate a systemd environment file:

```yaml
conf:
  FOO: bar
```

And to generate generate a systemd unit file:

```yaml
conf:
  Unit:
    Description: Docker Compose container starter
```

When files are updated or deleted backups are created based on the existing file name but prefixed with a leading `.` and suffixed with a timestamp in ISO8601 format and the file extension `.bak`.

##### state

The `files` can optionally have one of four optional states set:

* `absent` - the file will be deleted.
* `edited` - the existing file will be edited using the [Ansible ini module](https://docs.ansible.com/ansible/latest/collections/community/general/ini_file_module.html).
* `present` - if the file exists it will be edited using the [Ansible ini module](https://docs.ansible.com/ansible/latest/collections/community/general/ini_file_module.html), if not it will be created using the [templates/unit.j2](templates/unit.j2) template.
* `templated` - the file will be created if it does not exist or updated if it already exists using the [templates/unit.j2](templates/unit.j2) template.

If the `files` `state` is not set it defaults to `present`. The `edited` option can not remove variables and, unlike the `templated` option, it preserves existing comments.

#### pkgs

The `pkgs` variable is a list of `.deb` packages which will be installed when the `state` is present and removed when `absent`.

#### state

The `systemd_units` list elemenst can have a `state` of `absent` or `present`.

#### unit_state

The `unit_state` variable is used for the [Ansible systemd module state](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/systemd_module.html#parameter-state) so it can be one of `reloaded`, `restarted`, `started` or `stopped`.

## Usage example

This role can be included in another role along these lines (this has been based on [this gist](https://gist.github.com/Luzifer/7c54c8b0b61da450d10258f0abd3c917)):

```yaml
- name: Include systemd role
  ansible.builtin.include_role:
    name: systemd
    tasks_from: unit_present.yml
  vars:
    systemd_unit:
      name: docker-compose
      files:
        - path: /etc/systemd/docker-compose.conf
          conf:
            DOCKER_COMPOSE_VERSION: native
        - path: /etc/systemd/service/docker-compose.service
          conf:
            Unit:
              Description: Docker Compose container starter
              After: docker.service network-online.target
              Requires: docker.service network-online.target
            Service:
              User: mailcow
              Group: mailcow
              EnvironmentFile: /etc/systemd/system/sharedfutures.conf
              WorkingDirectory: /opt/mailcow-dockerized
              Type: oneshot
              RemainAfterExit: "yes"
              ExecStart: docker compose up -d
              ExecStop: docker compose down
            Install:
              WantedBy: multi-user.target
```

## Read existing systemd files as YAML using JC

You can read existing systemd files as YAML on the command line using [JC](https://github.com/kellyjonbrazil/jc), for example:

```bash
cat /etc/systemd/timesyncd.conf | jc --ini -py
---
Time:
  NTP: 0.pool.ntp.org 1.pool.ntp.org 3.pool.ntp.org 2.pool.ntp.org
```

## Dependencies

This role requires Ansible `2.13` or newer, [JC](https://pypi.org/project/jc/) and [JMESPath](https://pypi.org/project/jmespath/) to be installed using `pip3` on the Ansible controller.

## Repository

The primary URL of this repo is [`https://git.coop/webarch/systemd`](https://git.coop/webarch/systemd) however it is also [mirrored to GitHub](https://github.com/webarch-coop/ansible-role-systemd) and [available via Ansible Galaxy](https://galaxy.ansible.com/chriscroome/systemd).

If you use this role please use a tagged release, see [the release notes](https://git.coop/webarch/systemd/-/releases).

## License

This role is released under [the same terms as Ansible itself](https://github.com/ansible/ansible/blob/devel/COPYING), the [GNU GPLv3](LICENSE).
