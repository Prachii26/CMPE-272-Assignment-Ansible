# SJSU Individual Assignment – Ansible with Vagrant VMs

## 1. Introduction
This assignment demonstrates the setup and configuration of two Ubuntu VMs using Vagrant and VirtualBox, and the automation of webserver deployment using Ansible from WSL (Ubuntu). Each VM is configured to host a web page accessible on port 8080, displaying a unique message.
## 2. Requirements
•	Windows 10/11 with WSL2 (Ubuntu installed)
•	VirtualBox 7.x
•	Vagrant
•	Ansible installed inside WSL (via apt)

## 3. VM Setup with Vagrant
1.	Created project directory: C:\Users\Prach\ansible-vms
2.	Added a Vagrantfile with configuration for two Ubuntu VMs (web1, web2).
3.	Ran `vagrant up` to start the VMs.
4.	Verified IPs assigned: 192.168.56.11 (web1), 192.168.56.12 (web2).

## 4. Ansible Configuration in WSL
1.	Installed Ansible inside WSL.
2.	Copied SSH keys from Vagrant into ~/.ssh/vagrant-keys/    
```cd /mnt/c/Users/Prach/ansible-vms
mkdir -p ~/.ssh/vagrant-keys
cp .vagrant/machines/web1/virtualbox/private_key ~/.ssh/vagrant-keys/web1
cp .vagrant/machines/web2/virtualbox/private_key ~/.ssh/vagrant-keys/web2
chmod 600 ~/.ssh/vagrant-keys/* ```
3.	Created `inventory.ini` pointing directly to private IPs of the VMs.
4.	Verified SSH and Ansible connectivity(ansible -i inventory.ini all -m ping & ansible -i inventory.ini web -a "uname -a")
### inventory.ini code:
```[web]
web1 ansible_host=192.168.56.11 ansible_user=vagrant ansible_ssh_private_key_file=~/.ssh/vagrant-keys/web1 sjsu_id=1
web2 ansible_host=192.168.56.12 ansible_user=vagrant ansible_ssh_private_key_file=~/.ssh/vagrant-keys/web2 sjsu_id=2
```
## 5. Playbook and Templates
Created the following files:
### templates/index.html.j2:
<!doctype html>
<html>
<head><meta charset="utf-8"><title>SJSU-{{ sjsu_id }}</title></head>
<body>
<h1>Hello World from SJSU-{{ sjsu_id }}</h1>
</body>
</html>
### templates/nginx_8080.conf.j2:
server {
    listen 8080;
    server_name _;
    root /var/www/html;
    index index.html;
    location / {
        try_files $uri $uri/ =404;
    }
}
### site.yml:
---
- name: Deploy webserver on port 8080
  hosts: web
  become: true
  tags: [deploy]
  handlers:
    - name: reload nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
  tasks:
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600
    - name: Install Nginx
      ansible.builtin.apt:
        name: nginx
        state: present

    - name: Place site config (listen 8080)
      ansible.builtin.template:}
        src: templates/nginx_8080.conf.j2
        dest: /etc/nginx/sites-available/sjsu.conf
        mode: '0644'
      notify: reload nginx

    - name: Disable default site if present
      ansible.builtin.file:
        path: /etc/nginx/sites-enabled/default
        state: absent
      notify: reload nginx

    - name: Enable our site
      ansible.builtin.file:
        src: /etc/nginx/sites-available/sjsu.conf
        dest: /etc/nginx/sites-enabled/sjsu.conf
        state: link
      notify: reload nginx

    - name: Deploy index.html with SJSU id
      ansible.builtin.template:
        src: templates/index.html.j2
        dest: /var/www/html/index.html
        mode: '0644'

    - name: Ensure Nginx is enabled and running
      ansible.builtin.service:
        name: nginx
        enabled: true
        state: started
- name: Un-deploy webserver and clean resources
  hosts: web
  become: true
  tags: [undeploy]
  tasks:
    - name: Stop Nginx if running
      ansible.builtin.service:
        name: nginx
        state: stopped
      ignore_errors: true
    - name: Remove our site symlink
      ansible.builtin.file:
        path: /etc/nginx/sites-enabled/sjsu.conf
        state: absent
      ignore_errors: true

    - name: Remove our site config
      ansible.builtin.file:
        path: /etc/nginx/sites-available/sjsu.conf
        state: absent
      ignore_errors: true

    - name: Remove index.html (optional)
      ansible.builtin.file:
        path: /var/www/html/index.html
        state: absent
      ignore_errors: true

    - name: Remove Nginx package (optional)
      ansible.builtin.apt:
        name: nginx
        state: absent
      ignore_errors: true

## 6. Deployment Process
1.	Ran `ansible-playbook -i inventory.ini site.yml --tags deploy`.
2.	Verify tasks executed successfully.
3.	Ran again to confirm idempotency (changed=0).
  
## 7. Verification
1.	Opened browser on Windows host.
2.	Navigate to:
   - http://192.168.56.11:8080 → Hello World from SJSU-1
    - http://192.168.56.12:8080 → Hello World from SJSU-2
 
## 8. Un-deployment
1.	Run `ansible-playbook -i inventory.ini site.yml --tags undeploy`.
2.	Verify that Nginx and web resources were removed.

## 9. Conclusion
This assignment successfully demonstrated the use of Ansible to configure two VMs with a webserver on port 8080, deploy a unique web page on each, prove idempotency, and clean up using an un-deploy play. All requirements of the assignment were satisfied.
