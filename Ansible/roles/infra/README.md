Ansible Role: Infra
=========

* Creates a VPC with 2 subnets (private and public)
* Creates 2 ec2 instanes in the private subnet, one as kubernetes controller and one as kubernetes worker and attaches the required security groups to each one.
* Create a NAT instance in the public subnet to provide internet access to machines in the private subnet, the route table is configured to use this machine for internet access.

Requirements
------------

boto3 pip package needs to be installed but if it isn't the role already installs it in the beginning

Example Playbook
----------------

```yml
- hosts: servers
  roles:
  - infra
```

License
-------

GPL
