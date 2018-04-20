Tests are dona via [vagrantfile_config](https://github.com/jtyr/vagrantfile_config).

Usage:

```bash
export VAGRANT_CONFIG_YAML=vagrant-centos6.yaml
vagrant up
```

Used roles:

- [`glusterfs_server`](https://github.com/jtyr/ansible-glusterfs_server)
- [`glusterfs_client`](https://github.com/jtyr/ansible-glusterfs_client)
- [`vagrant_ubuntu_init`](https://github.com/jtyr/ansible-vagrant_ubuntu_init)
- [`lvm_extend`](https://github.com/jtyr/ansible-lvm_extend)
