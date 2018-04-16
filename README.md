glusterfs_server
================

Ansible role which helps to install and configure GlusterFS server.

The configuration of the role is done in such way that it should not be
necessary to change the role for any kind of configuration. All can be
done either by changing role parameters or by declaring completely new
configuration as a variable. That makes this role absolutely
universal. See the examples below for more details.

Please report any issues or send PR.


Examples
--------

```yaml
---

- name: Example of how to only install GlusterFS server
  hosts: all
  roles:
    - glusterfs_server

- name: Example of how to install GlusterFS and manage volumes
  hosts: all
  vars:
    # Partitioning
    lvm_extend_default_fs_type: xfs
    lvm_extend_config:
      vg_data:
        disks:
          - sdb
        vols:
          - name: bricks
            mount: yes
    # GlusterFS volume creation
    glusterfs_server_volumes:
      - name: test1
        bricks: /mnt/bricks/b1
        replicas: 2
        cluster:
          - 192.168.169.10
          - 192.168.169.11
        run_once: yes
  roles:
    - lvm_extend
    - glusterfs_server
```


Role variables
--------------

```yaml
# GlusfterFS release
glusterfs_server_yumrepo_release: "{{
  '3.13'
    if ansible_distribution_major_version | int == 6
    else
  '4.0' }}"

# YUM repo URL
glusterfs_server_yumrepo_url: https://buildlogs.centos.org/centos/{{ ansible_distribution_major_version }}/storage/$basearch/gluster-{{ glusterfs_server_yumrepo_release }}/

# Package to be installed (explicit version can be specified here)
glusterfs_server_pkg: glusterfs-server

# Service name
glusterfs_server_service: glusterd

# Brick directory attributes
glusterfs_server_brick_owner: gluster
glusterfs_server_brick_group: gluster
glusterfs_server_brick_mode: "0750"

# Volumes (see README for examples)
glusterfs_server_volumes: []
```


Dependencies
------------

- [`glusterfs_client`](https://github.com/jtyr/ansible-glusterfs_client) (optional)
- [`lvm_extend`](https://github.com/jtyr/ansible-lvm_extend) (optional)


License
-------

MIT


Author
------

Jiri Tyr
