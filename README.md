# Ansible + AWS


## Create a Security Group

Create a key pair

Name it `ansible` and download it in `.pem`. format. 

Create an access key. 

In account settings, choose My Security Credentials. 

Download the .csv file. 



Verify (or create) a default VPC (Virtual Private Cloud).



Add access key to PATH.
```
export AWS_ACCESS_KEY_ID='ABCDEFGHIJKLMNOP'
```
And:
```
export AWS_SECRET_ACCESS_KEY=`1234567890etc'
```

Create `ec2_playbook.yaml`:
```yml
---
-
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
  - name: Create a security group in AWS for SSH access and HTTP
    ec2_group:
      name: ansible
      description: Ansible Security Group
      region: us-east-1
      rules:
        - proto: tcp
            from_port: 80
            to_port: 80
            cidr_ip: 0.0.0.0/0
        - proto: tcp
            from_port: 22
            to_port: 22
            cidr_ip: 0.0.0.0/0
...
```

Install `boto` and `boto3`:
```
sudo pip3 install boto boto3
```

Run the playbook: 
```sh
ansible-playbook ec2_playbook.yaml
```

The output should be:
```sh
PLAY [localhost] ***************************************************************************

TASK [Create a security group in AWS for SSH access and HTTP] ******************************
ok: [localhost]

PLAY RECAP *********************************************************************************
localhost                  : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
```

## Provision a Set of Instances

Back in the AWS console, under Launch Instance, find the AMI for OS you would like to deploy. For example, the AMI ID for Ubuntu is:
```
ami-08c40ec9ead489470
```

Add the following tasks to `ec2_playbook.yaml`:
```yml
  - name: Provision a set of instances
      ec2:
         key_name: ansible-ec2
         group: ansible
         instance_type: t2.micro
         image: ami-08c40ec9ead489470
         region: us-east-1
         wait: true
         exact_count: 20
         count_tag:
            Name: AnsibleNginxWebservers
         instance_tags:
            Name: Ansible
      register: ec2
  
  - name: Add all instance public IPs to host group
    add_host:
      hostname: "{{ item.public_ip }}"
      groups: ansiblehosts
    with_items: "{{ ec2.instances }}"

  - name: Show group
    debug:
      var: groups.ansiblehosts
```

The first time you run this will be fine. But if you run it again, you'll get errors due to using up your instance limit on the free tier. 

We can prevent this error from stopping the playbook by adding the following line to `ec2_playbook.yaml`:
```
ignore_errors: true
```

## Dynamic Inventory


TODO UPDATE TO USE aws_ec2 plugin https://docs.ansible.com/ansible/latest/collections/amazon/aws/aws_ec2_inventory.html

The aws_ec2 plugin is a part of the `amazon.aws` collection. Check to see if it is installed:
```
ansible-galaxy collection list
```

It'll be right at the top of the list. 

If it's not installed:
```
ansible-galaxy collection install amazon.aws
```

See: https://aaronoellis.com/articles/replacing-ansibles-aws-ec2py-script-with-the-aws-ec2-plugin

---

Make a directory for the inventory: 
```
mkdir inventory
```

TODO
```
cd inventory
```

Download the Amazon dynaminc inventory:
```
wget TODO
```

Make the inventory executable:
```
chmod u+x ec2.py
```


TODO `ansible.cfg`:
```
[defaults]
inventory = inventory/ec2.py
host_key_checking = False
forks=20
ansible_managed = Managed by Ansible - file:{file} - host:{host} - uid:{uid}
```

TODO `group_vars`:
...


Ensure permissions are set correctly:
```
sudo chmod 600 ~/.ssh/ansible-ec2.pem
```

Check that the instances are pingable:
```
ansible tag_Name_Ansible -m ping -o
```


## Roles

Create a new role:
```sh
ansible-galaxy init nginx
```

In `/nginx/handlers/main.yml` add:
```yml

- name: Check HTTP Service
  uri:
    url: http://{{ ansible_default_ipv4.address }}
    status_code: 200 
```

In `/nginx/vars/main.yml` add:
```yml
target_dir: 
```

In `/nginx/tasks/main.yml` add:
```yml
- name: Install EPEL
  yum:
    name: epel-release
    update_cache: yes
    state: latest
  when: ansible_distribution == 'CentOS'
  tags:
    - install-epel

- name: Install Nginx
  package:
    name: nginx
    state: latest
  tags:
    - install-nginx

- name: Restart nginx
  service:
    name: nginx
    state: restarted
  notify: Check HTTP Service
  tags:
    - always
```


Create a role for `webapp`:
```sh
ansible-galaxy init webapp
```

TODO templates

TODO files


In /`webapp/vars/main.yml` add: 
```yml
target_dir: /var/www/html
```

In `/webapp/tasks/main.yml` add:
```yml
- name: Template index.html-easter_egg.j2 to index.html on target
  template:
    src: index.html-easter_egg.j2
    dest: "{{ target_dir }}/index.html"
    mode: 0644
  tags:
    - deploy-app

- name: Install unzip
  package:
    name: unzip
    state: present

- name: Unarchive playbook stacker game
  unarchive:
    src: playbook_stacker.zip
    dest: "{{ target_dir }}"
    mode: 0755
  tags:
    - deploy-app
```



In `webapp/meta/main.yml` add `nginx` as a dependency:
```yml
dependencies: []
  - nginx
```



Add the roles to `ec2_playbook.yml`:
```yml
  roles:
    - { role: webapp, target_dir: /usr/share/nginx/html }
```



Repace this:
```yml
  - name: Add all instance public IPs to host group
    add_host:
      hostname: "{{ item.public_ip }}"
      groups: ansiblehosts
    with_items: "{{ ec2.instances }}"

  - name: Show group
    debug:
      var: groups.ansiblehosts
```

with this:
```yml
    - name: Refresh inventory to ensure new instances exist in inventory
      meta: refresh_inventory

-

  # Target: where our play will run and options it will run with
  hosts: tag_Name_Ansible

  # Roles: list of roles to be imported into the play
  roles:
    - { role: webapp, target_dir: /usr/share/nginx/html }
```

And run:
```sh
ansible-playbook ec2_playbook.yaml 
```

## Remove Group and Terminate Instances

Add the following to pause:
```yml
  hosts: tag_Name_Ansible

  # Task: the list of tasks that will be executed within the play, this section
  # can also be used for pre and post tasks
  tasks:
    - debug:
        msg: "Check http://{{ ansible_host }}"

    - pause:
        prompt: "Verify service availability and continue to terminate"

    - name: Remove tagged EC2 instances from security group by setting an empty group
      ec2:
        state: running
        region: "{{ ec2_region }}"
        instance_ids: "{{ ec2_id }}"
        group_id: ""
      delegate_to: localhost

    - name: Terminate EC2 instances
      ec2:
        state: absent
        region: "{{ ec2_region }}"
        instance_ids: "{{ ec2_id }}"
        wait: true
      delegate_to: localhost

-

  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
  - name: Remove ansible security group
    ec2_group:
      name: ansible
      region: us-east-1
      state: absent

```









