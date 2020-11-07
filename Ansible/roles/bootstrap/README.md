Ansible Role: Bootstrap
=========

Set up the common components on a K8s cluster (The components relevant to both the Controller and the Worker nodes, ex: Container Runtime)


Example Playbook
----------------
```yml
- hosts: servers
  roles:
      - bootstrap
```

License
-------

GPL
