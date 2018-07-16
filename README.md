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
        # Use odd number of replicas to prevent split-brain situation
        replicas: 3
        cluster:
          - 192.168.169.10
          - 192.168.169.11
          - 192.168.169.12
        run_once: yes
        options:
          # Allow connection from client running on the localhost only
          auth.allow: localhost
          # Reduce ping timeout to detect host is down
          network.ping-timeout: "10"
          # Disable NFS
          nfs.disable: "on"
  roles:
    - lvm_extend
    - glusterfs_server
```


Role variables
--------------

```yaml
# GlusfterFS release
glusterfs_server_yumrepo_release: "{{
  '3.10'
    if (
      ansible_distribution == 'Ubuntu' and
      ansible_distribution_major_version | int < 16
    ) else
  '3.13'
    if (
      (
        ansible_os_family == 'RedHat' and
        ansible_distribution_major_version | int < 7
      ) or (
        ansible_distribution == 'Debian' and
        ansible_distribution_major_version | int < 9
      )
    ) else
  '4.1' }}"

# YUM repo URL
glusterfs_server_yumrepo_url: https://buildlogs.centos.org/centos/{{ ansible_distribution_major_version }}/storage/$basearch/gluster-{{ glusterfs_server_yumrepo_release }}/

# APT key
glusterfs_server_apt_key: "{{
  'https://download.gluster.org/pub/gluster/glusterfs/' ~ glusterfs_server_yumrepo_release ~ '/rsa.pub'
    if ansible_distribution == 'Debian'
    else
  '' }}"

# APT repo URL
glusterfs_server_apt_url: "{{
  'ppa:gluster/glusterfs-' ~ glusterfs_server_yumrepo_release
    if ansible_distribution == 'Ubuntu'
    else
  'deb https://download.gluster.org/pub/gluster/glusterfs/' ~ glusterfs_client_yumrepo_release ~ '/LATEST/Debian/' ~ ansible_distribution_release ~ '/amd64/apt ' ~ ansible_distribution_release ~  ' main' }}"

# Package to be installed (explicit version can be specified here)
glusterfs_server_pkg: glusterfs-server

# Additional Gluster packages (e.g. glusterfs-rdma)
glusterfs_server_pkg_add: []

# Location of the secure-access file
glusterfs_server_secure_access_file: /var/lib/glusterd/secure-access

# Whether to create the secure-access file or not
glusterfs_server_secure_access: no

# Location of the glusterd.info file (recovery only)
glusterfs_server_glusterd_info: /var/lib/glusterd/glusterd.info

# Server UUID (recovery only)
glusterfs_server_uuid: null

# Service name
glusterfs_server_service: "{{
  'glusterd'
    if (
      ansible_os_family == 'RedHat' or (
        ansible_distribution == 'Ubuntu' and
        ansible_distribution_major_version | int >= 16
      ) or
        ansible_distribution == 'Debian' and
        ansible_distribution_major_version | int >= 9
    ) else
  'glusterfs-server' }}"

# Brick directory attributes
glusterfs_server_brick_owner: root
glusterfs_server_brick_group: root
glusterfs_server_brick_mode: "0750"

# Volumes (see README for examples)
glusterfs_server_volumes: []
```


GlusterFS procedures
--------------------

### Replacing a Host Machine with the same hostname

If a GlusterFS host becomes unrecovable, it's necessary to rebuild it and
make it to join the cluster. This can be done like described in the [GlusterFS
documentation](https://access.redhat.com/documentation/en-us/red_hat_gluster_storage/3/html/administration_guide/sect-replacing_hosts#Replacing_a_Host_Machine_with_the_Same_Hostname)
and this role can help with that a little bit. The procedure requires to gather
two important information - the _server UUID_ of the failed server and the
_volume ID(s)_ of the volume(s).

First we build a new host with the same hostname and IP like the failed host.
Then we find the UUID of the failed host by running `gluster pool list` on any
healthy host of the cluster. Then we find the volume IDs of all volumes we need
to recover o the new host by running `getfattr -d -m trusted.glusterfs.volume-id
-ehex /mnt/bricks/b1` on any of the healthy hosts containing that volume. Then
we use these information during the `glusterfs_server` role installation by
adding these variables:

```
# This is the host UUID we got from the pool list
glusterfs_server_uuid: <uuid>

glusterfs_server_volumes:
  - name: test1
    ...
    # This is the volume ID we got by running getfattr
    volume_id: <volume_id>
```

Now we we run the `glusterfs_server` role on the new host which will install
GlusterFS, set the UUID and the volume ID. After that we are ready to start the
recovery process (all commands are run on the new host):

```
# Probe the healthy peers
gluster peer probe 192.168.234.11
gluster peer probe 192.168.234.12
# Sync volumes from one of the healthy hosts
gluster volume sync 192.168.234.11 all
# Restart the service
systemctl stop glusterd
ps -ef | grep glus | grep -v grep && killall glusterfsd glusterfs glusterd
systemctl start glusterd
# Check if the brick runs (the brick should have a port assigned)
gluster volume status
# Start healing of the volume
gluster volume heal <volume_name> full
# Wait for the cluster to finish healing
gluster volume heal <volume_name> info
```

Then remove or comment out the `glusterfs_server_uuid` variable and the
`volume_id` option from the `glusterfs_server_volumes` variable. Then you can
continue by running the `glusterfs_client` on the new host.


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
