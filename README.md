# Webarchitects systemd Ansible role

[![pipeline status](https://git.coop/webarch/systemd/badges/main/pipeline.svg)](https://git.coop/webarch/systemd/-/commits/main)

An Ansible role for configuring systemd services on Debian, this role has been designed to be as generic as possible in order to enable to it be used to configure any systemd service, by default it configures `systemd-timesyncd`.

On Debian Buster [backports](https://backports.debian.org/Instructions/) is required to get the [latest version of systemd](https://packages.debian.org/buster-backports/systemd), the [Webarchitects apt role](https://git.coop/webarch/apt) can be used to enable backports.

## Role variables

See the [defaults/main.yml](defaults/main.yml) file for the default variables, these are described below.

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

A list of systemd units to configure, for example:

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

The only required variables is `name`, see the [meta/argument_specs.yml](meta/argument_specs.yml) for the variable types.

For each service required `.deb` packages, the state of the service and the files to be created / amended and their content as YAML can be specified.

Files are read using the [JC ini parser](https://kellyjonbrazil.github.io/jc/docs/parsers/ini) and only updated if the `conf` is to be changed.

Files can optionally have one of four optional states set:

* `absent` - the file will be deleted.
* `edited` - the existing file will be edited using the [Ansible ini module](https://docs.ansible.com/ansible/latest/collections/community/general/ini_file_module.html).
* `present` - if the file exists it will be edited using the [Ansible ini module](https://docs.ansible.com/ansible/latest/collections/community/general/ini_file_module.html), if not it will be created using the [templates/unit.j2](templates/unit.j2) template.
* `templated` - the file will be created if it does not exist or updated if it already exists using the [templates/unit.j2](templates/unit.j2) template.

If the `state` is not set it defaults to `present`.

The `edited` option can not remove variables and, unlike the `templated` option, it preserves existing comments.

When files are updated or deleted backups are created based on the existing file name but prefixed with a leading `.` and suffixed with a timestamp in ISO8601 format and the file extension `.bak`.

See the [defaults/main.yml](defaults/main.yml) file for the default `systemd_units` YAML dictionary.

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
