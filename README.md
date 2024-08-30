# Webarchitects systemd Ansible role

[![pipeline status](https://git.coop/webarch/systemd/badges/main/pipeline.svg)](https://git.coop/webarch/systemd/-/commits/main)

`systemd` is a [System and Service Manager](https://systemd.io/) that _"runs as PID 1 and starts the rest of the system"_.

This repo contains an Ansible role for configuring [systemd on Debian](https://manpages.debian.org/systemd/systemd.1.en.html), this role has been designed to be as generic as possible in order to enable to it be used to configure any systemd service.

On Debian [Buster](https://packages.debian.org/buster-backports/systemd) and [Bullseye](https://packages.debian.org/bullseye-backports/systemd) when the [backports repo](https://backports.debian.org/Instructions/) is enabled this role will install systemd from backports, the [Webarchitects apt Ansible role](https://git.coop/webarch/apt) can be used to enable backports.

## Role variables

See the [defaults/main.yml](defaults/main.yml) file for the default variables and [meta/argument_specs.yml](meta/argument_specs.yml) for the variable specification.

### systemd

Set the `systemd` variable to `false` to prevent any tasks in this role being run.

The `systemd` variable defaults to `true`.

### systemd_delete_broken_symlinks

The `systemd_delete_broken_symlinks` variable is a boolean, when `true` is results in this role deleting broken symlinks found in the `/etc/systemd` directory, it defaults to `false`.

### systemd_delete_devnull_symlinks

The `systemd_delete_devnull_symlinks` variable is a boolean, when `true` is results in this role deleting symlinks that point to `/dev/null` in the `/etc/systemd` directory, it defaults to `false`.

### systemd_timesyncd_reboot

The `systemd_timesyncd_reboot` variable is a boolean, when `true` servers which have incorrect clocks will be rebooted by this role in an attempt to correct their clocks, it defaults to `false`.

This variable is only used if there is a item in the `systemd_units` list with the `name` `systemd-timesyncd`.

### systemd_tz

The `systemd_tz` variable is a string for the time zone to be used when configuring `systemd-timesyncd`, (the hardware clock is set to UTC), it defaults to `Etc/UTC`, which is fine for a server however for a desktop or laptop you will want to use a value like `Europe/London`.

This variable is only used if there is a item in the `systemd_units` list with the `name` `systemd-timesyncd`.

### systemd_units

The `systemd_units` array is a list of systemd units to configure, for example:

```yaml
systemd_units:
  - name: systemd-timesyncd
    files:
      - path: /etc/systemd/timesyncd.conf.d/timesyncd.conf
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
    state: enabled
    verify: systemd-timesyncd.service
  - name: systemd-networkd
    files:
      - path: /etc/systemd/network/eth0.network
        state: templated
        comment: Additional IP address added to the eth0 interface.
        conf:
          Match:
            Name: eth0
            Type: ether
          Network:
            Address:
              - 192.168.0.2/24
              - 192.168.0.3/24
            Gateway: 192.168.0.1
    verify: networking.service
```

The `systemd_units` list variables follow:

#### name

The `name` for the `systemd_units` list item is the `name` of the systemd unit to configure.

It is used as the `name` for the [Ansible systemd module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/systemd_module.html):

> This parameter takes the name of exactly one unit to work with.
>
> When no extension is given, it is implied to a `.service` as systemd.
>
> When using in a chroot environment you always need to specify the name of the unit with the extension. For example, `crond.service`.

You can generate a YAML list of all the unit names:

```bash
systemctl list-unit-files | jc --systemctl-luf | jp [].unit_file | yq -P
```

See also the [systemd unit configuration documentation](https://manpages.debian.org/systemd/systemd.unit.5.en.html).

#### files

The `files` variable for the `systemd_units` list item is an array of files to be configured for the unit, items in the `files` list can use the following variables:

##### path

The `path` for the item in the `files` list is used for the full filesystem path to the file to configure, this variables is required.

##### comment

The `comment` for the item in the `files` list can be used for adding commented text to the top of the file.

##### conf

The `conf` dictionary for the item in the `files` list defines the systemd file contents in the form of a YAML dictionary.

The depth of the dictionary defines the type of file that will be configured, fir example to enerate a systemd environment file (`conf` depth zero):

```yaml
conf:
  Time:
```

The dictionary is converted into a file containing an empty section:

```ini
[Time]
```

To generate a systemd environment file (`conf` depth one):

```yaml
conf:
  FOO: bar
```

The dictionary is converted into a environmental variables file:

```ini
FOO=bar
```

See the [systemd environment configuration documentation](https://manpages.debian.org/systemd/systemd.exec.5.en.html).

To generate generate a systemd unit file without any duplicated entries (`conf` depth two):

```yaml
conf:
  Unit:
    Description: Docker Compose container starter
```

The dictionary is converted into a file:

```ini
[Unit]
Description=Docker Compose container starter
```

When duplicate entries are allowed a YAML list is used (`conf` depth three):

```yaml
conf:
  Network:
    Address:
      - 192.168.0.2/24
      - 192.168.0.3/24
```

The list items are are converted into duplicated keys:

```ini
[Network]
Address=192.168.0.2/24
Address=192.168.0.3/24
```

Note that the documentation for the [general syntax of systemd configuration files](https://manpages.debian.org/systemd/systemd.syntax.7.en.html) includes:

> Boolean arguments used in configuration files can be written in various formats. For positive settings the strings `1`, `yes`, `true` and `on` are equivalent. For negative settings, the strings `0`, `no`, `false` and `off` are equivalent.

If you want a value that is used as a boolean in a systemd unit file to be treated as a string you need to quote it, for example `"yes"`, if you don't quote it then Ansible will consider numbers to be integers in the case of `1` or `0` and unquoted `true` and `false` will be booleans, the unit template, [templates/unit.j2](templates/unit.j2) is set to lower case booleans to avoid them becoming `True` and `False`.

Also note that duplicated sections are allowed by systemd however this role doesn't support duplicated sections in the same file.

When files are updated or deleted backups are created based on the existing file name but prefixed with a leading `.` and suffixed with a timestamp in ISO8601 format and the file extension `.bak`.

##### files state

The `state` of the item in the `files` list can optionally be set to one of four states when the `systemd_units` item is `enabled` or `stopped`, if the `systemd_units` item is set to `absent` all the items in the `files` list will be deleted, if the `state` of the item in the `files` list is not set then it defaults to `present`, the the four states are:

* `absent` - the file will be deleted.
* `edited` - if the file exists it will be edited using the [Ansible ini module](https://docs.ansible.com/ansible/latest/collections/community/general/ini_file_module.html), as long as there are no duplicates, if there are duplicates or the file doesn't exist it cannot be edited.  The `edited` option cannot remove variables however unlike the `templated` option, it preserves existing comments.
* `present` - if the file exists it will be edited using the [Ansible ini module](https://docs.ansible.com/ansible/latest/collections/community/general/ini_file_module.html), as long as there are no duplicates, if there are duplicates or it doesn't exist it will be created using the [templates/unit.j2](templates/unit.j2) template, `present` is the default state.
* `templated` - the file will be created if it does not exist or updated if it already exists using the [templates/unit.j2](templates/unit.j2) template.

Don't confuse the `state` of the items in the `files` list  with the `state` of the `systemd_unit`.

#### pkgs

The optional `pkgs` list of `.deb` packages for the `systemd_units` list item will be installed when the `state` is present and removed when `absent`.

#### systemd_units state

The `systemd_units` list item `state` can be optionally set to one of three states, if it is not set it defaults to `enabled`:

* `absent`, will result in the systemd unit being uninstalled, this means that the service will be stopped, `.deb` packages listed in the `pkgs` list will be removed and any files listed in the `files` array will be deleted.
* `enabled`, will result in the systemd unit being being installed, `enabled` and `started`. If any unit files are changed when the role is run the systemd unit will be  `restarted`.
* `stopped`, will result in the systemd unit being installed, but it will be `stopped` and not `enabled`.

#### verify

A optional systemd service name to be passed to `systemd-analyze verify`, the file extension is required.

A list of services that can be verified can be generated using:

```bash
systemctl list-unit-files | jc --systemctl-luf -py
```

If `verify` is not defined and there is an existing service that has a name that matches the `name`, or the `name` with `.service` appended then tis service will be verified.

## Usage example

This role can be included in another role along these lines (this has been based on [this gist](https://gist.github.com/Luzifer/7c54c8b0b61da450d10258f0abd3c917)):

```yaml
- name: Include systemd role
  ansible.builtin.include_role:
    name: systemd
    tasks_from: unit_present.yml
  loop: "{{ docker_systemd_units }}"
  loop_control:
    loop_var: systemd_unit
    label: "{{ systemd_unit.name }}"
  vars:
    docker_systemd_units:
      - name: docker-compose
        state: enabled
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
                EnvironmentFile: /etc/systemd/docker-compose.conf
                WorkingDirectory: /opt/mailcow-dockerized
                Type: oneshot
                RemainAfterExit: "yes"
                ExecStart: docker compose up -d
                ExecStop: docker compose down
              Install:
                WantedBy: multi-user.target
```

## Notes

You can read existing systemd files as YAML on the command line using [jc](https://github.com/kellyjonbrazil/jc), for example:

```bash
cat /etc/systemd/timesyncd.conf | jc --ini -py
```

```yaml
---
Time:
  NTP: 0.pool.ntp.org 1.pool.ntp.org 3.pool.ntp.org 2.pool.ntp.org
```

Note that the [jc --ini parser](https://kellyjonbrazil.github.io/jc/docs/parsers/ini) doesn't support duplicate keys, however the `--ini-dup` parset that is due to be released in version `1.22.5` does, however it returms all values as a list, for example:

```bash
cat /etc/systemd/timesyncd.conf | jc --ini-dup -py
```

```yaml

---
Time:
  NTP:
  - 0.pool.ntp.org 1.pool.ntp.org 2.pool.ntp.org 3.pool.ntp.org
```

### Network interface names

For upgrading older Debian servers see the ["Predictable Names" Migration HOWTO](https://wiki.debian.org/NetworkInterfaceNames#migration).

Check the existing name:

```bash
ls /sys/class/net/ | grep -v ^lo$
eth0
```

For each result ask `udevadm` what `NET_ID`s it knows:

```bash
udevadm test-builtin net_id /sys/class/net/eth0 2>/dev/null
ID_NET_NAMING_SCHEME=v253
ID_NET_NAME_MAC=enx00163ee9dd05
ID_OUI_FROM_DATABASE=Xensource, Inc.
ID_NET_NAME_SLOT=enX0
```

Remove `/etc/systemd/network/99-default.link` and rebuild `initd`:

```bash
mv /etc/systemd/network/99-default.link /etc/systemd/network/.99-default.link.bak
update-initramfs -u
```

Run this role with a config like this:

```yaml
systemd_units:
  - name: systemd-networkd
    files:
      - path: /etc/systemd/network/enX0.link
        conf:
          Match:
            MACAddress: "00:16:3e:e9:dd:05"
          Link:
            Name: enX0
      - path: /etc/systemd/network/enX0.network
        conf:
          Match:
            Name: enX0
            Type: ether
          Network:
            Address:
              - 81.95.52.71/25
            Gateway: 81.95.52.3
```

## Dependencies

This role requires Ansible `2.13` or newer, [JC](https://pypi.org/project/jc/) and [JMESPath](https://pypi.org/project/jmespath/) to be installed using `pip3` on the Ansible controller.

## Repository

The primary URL of this repo is [`https://git.coop/webarch/systemd`](https://git.coop/webarch/systemd) however it is also [mirrored to GitHub](https://github.com/webarch-coop/ansible-role-systemd) and [available via Ansible Galaxy](https://galaxy.ansible.com/chriscroome/systemd).

If you use this role please use a tagged release, see [the release notes](https://git.coop/webarch/systemd/-/releases).

## Copyright

Copyright 2022-2024 Chris Croome, &lt;[chris@webarchitects.co.uk](mailto:chris@webarchitects.co.uk)&gt;.

This role is released under [the same terms as Ansible itself](https://github.com/ansible/ansible/blob/devel/COPYING), the [GNU GPLv3](LICENSE).
