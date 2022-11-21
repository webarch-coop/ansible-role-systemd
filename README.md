# Webarchitects systemd Ansible role

[![pipeline status](https://git.coop/webarch/systemd/badges/main/pipeline.svg)](https://git.coop/webarch/systemd/-/commits/main)

An Ansible role for configuring systemd services on Debian, this role has been designed to be as generic as possible in order to enable to it be used to configure any Systemd service.

## Role variables

See the [defaults/main.yml](defaults/main.yml) file for the default variables.

### systemd

Set the `systemd` variable to `false` to prevent any tasks in this role being run, it defaults to `true`.

### systemd_timesyncd_reboot

Set the `systemd_timesyncd_reboot` variable to `true` for servers which have incorrect clocks to be rebooted by this role in order to correct their clocks, this variable defaults to false.

### systemd_units

A list of System units to configure, for example:

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
        state: templated
    pkgs:
      - systemd-timesyncd
    state: present
    unit_state: started
```

The only required variables is `name`, see the [meta/argument_specs.yml](meta/argument_specs.yml) for the variable types.

For each service required `.deb` packages, the state of the service and the files to be created / amended and their content in YAML can be specified.

Files are read using the [JC ini parser](https://kellyjonbrazil.github.io/jc/docs/parsers/ini) and only updated if the `conf` is to be changed.

Files can have one of three states set:

* `absent` - file deleted.
* `edited` - edit an existing file.
* `templated` - create a file if one does not exist or replace an existing one.

The `edited` option uses the [Ansible ini module](https://docs.ansible.com/ansible/latest/collections/community/general/ini_file_module.html) to change or add specified variables, howveer it can't remove variables, unlike the `templated` option it preserves existing comments the file.

The `templated` option generates the systemd file using the [templates/unit.j2](templates/unit.j2) template.

When files are updated or deleted backups are created based on the existing file name but prefixed with a leading `.` and suffixed with a timestamp in ISO8601 format and the file extension `.bak`.

## Read existing Systemd files using JC

You can read existing systemd files as YAML on the command line using [JC](https://github.com/kellyjonbrazil/jc), for example:

```bash
cat /etc/systemd/timesyncd.conf | jc --ini -py
---
Time:
  NTP: 0.pool.ntp.org 1.pool.ntp.org 3.pool.ntp.org 2.pool.ntp.org
```

## Dependencies

This role requires Ansible 2.11 or newer, [JC](https://pypi.org/project/jc/) and [JMESPath](https://pypi.org/project/jmespath/) to be installed using `pip3` on the Ansible controller.

## Repository

The primary URL of this repo is [`https://git.coop/webarch/systemd`](https://git.coop/webarch/systemd) however it is also [mirrored to GitHub](https://github.com/webarch-coop/ansible-role-systemd) and [available via Ansible Galaxy](https://galaxy.ansible.com/chriscroome/systemd).

If you use this role please use a tagged release, see [the release notes](https://git.coop/webarch/systemd/-/releases).

## License

This role is released under the same terms as Ansible itself, the [GNU GENERAL PUBLIC LICENSE, Version 3](LICENSE).

## Author

[Chris Croome](https://git.coop/chris).
