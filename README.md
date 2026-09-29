# V7AI DevOps Assessment

## Complete Hands-On DevOps Implementation Guide

A practical, step-by-step implementation of a multi-server DevOps environment using **Docker, Docker Compose, Ansible, Nginx, TLS/HTTPS, reverse proxying, and load balancing**.

This README is written so that someone can reproduce the project **from scratch**, understand **why each step is required**, execute the commands, verify the result, and troubleshoot common problems.

---

# 1. Project Overview

The assessment creates three Linux server-like environments using Docker containers:

```text
                    Developer Machine
                           |
                           |
                    Docker Host
                           |
              +------------+------------+
              |       v7ai-net          |
              |                         |
        +-----+-----+             +-----+-----+
        |    vm1    |             |    vm2    |
        | Nginx     |             | Nginx     |
        | HTTPS     |             | App       |
        | Reverse   |             | Server    |
        | Proxy     |             |           |
        +-----+-----+             +-----------+
              |
              |
        +-----+-----+
        |    vm3    |
        | Nginx     |
        | App       |
        | Server    |
        +-----------+
```

## Server responsibilities

| Server |            IP | SSH Port | Responsibility               |
| ------ | ------------: | -------: | ---------------------------- |
| vm1    | `172.30.0.11` |   `2011` | Public reverse proxy + HTTPS |
| vm2    | `172.30.0.12` |   `2012` | Application backend          |
| vm3    | `172.30.0.13` |   `2013` | Application backend          |

The public entry point is:

```text
Client
  |
  v
vm1
Nginx + HTTPS
  |
  +---- /vm2/ ----> vm2
  |
  +---- /vm3/ ----> vm3
  |
  +---- /app/ ----> vm2/vm3 load-balanced backend
```

---

# 2. Technologies Used

* Linux / Ubuntu
* Docker
* Docker Compose
* Docker networking
* OpenSSH
* Ansible
* Ansible Roles
* Nginx
* Nginx Reverse Proxy
* TLS / HTTPS
* Self-signed certificate
* UFW firewall
* Bash
* YAML
* Jinja2 templates

---

# 3. Prerequisites

The host machine should have:

```bash
docker --version
docker compose version
ansible --version
ssh -V
openssl version
```

Example environment used during the assessment:

```text
Ubuntu 24.04
Docker 29.1.3
Ansible
Nginx 1.24.0
```

Versions may differ on another machine.

---

# 4. Project Structure

The final project follows this structure:

```text
v7ai-devops-assessment/
│
├── README.md
│
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
│
└── ansible/
    │
    ├── inventory.ini
    ├── site.yml
    │
    └── roles/
        │
        ├── common/
        │   └── tasks/
        │       └── main.yml
        │
        ├── ssh/
        │   └── tasks/
        │       └── main.yml
        │
        ├── firewall/
        │   └── tasks/
        │       └── main.yml
        │
        ├── nginx/
        │   ├── tasks/
        │   │   └── main.yml
        │   │
        │   ├── handlers/
        │   │   └── main.yml
        │   │
        │   ├── templates/
        │   │   ├── default.conf.j2
        │   │   ├── reverse-proxy.conf.j2
        │   │   └── index.html.j2
        │   │
        │   └── files/
        │       └── tls/
        │           ├── public.vm1.local.crt
        │           └── public.vm1.local.key
        │
        ├── app/
        │   ├── tasks/
        │   │   └── main.yml
        │   │
        │   └── templates/
        │       ├── index.html.j2
        │       └── app.conf.j2
        │
        └── docker/
            └── tasks/
                └── main.yml
```

---

# 5. Step 1 — Create the Project

Create the project directory:

```bash
mkdir -p ~/v7ai-devops-assessment
cd ~/v7ai-devops-assessment
```

Create the Docker directory:

```bash
mkdir -p docker
```

Create the Ansible structure:

```bash
mkdir -p ansible/roles/{common,ssh,firewall,nginx,app,docker}/{tasks,handlers,templates,files}
```

For TLS files:

```bash
mkdir -p ansible/roles/nginx/files/tls
```

---

# 6. Step 2 — Create the Docker Image

Create:

```text
docker/Dockerfile
```

The image is used as the base operating system for all three server containers.

The container needs:

* Ubuntu
* SSH server
* Python
* sudo
* basic utilities

The important requirement is that the container starts SSH in the foreground:

```bash
/usr/sbin/sshd -D
```

Why?

Docker considers the main process as the container's lifecycle process. Running `sshd` in the foreground keeps the container alive and allows Ansible to connect through SSH.

---

# 7. Step 3 — Build the Docker Image

From the project root:

```bash
docker build -t v7ai-ubuntu:1.0 -f docker/Dockerfile .
```

Verify:

```bash
docker images
```

Expected:

```text
v7ai-ubuntu
```

with tag:

```text
1.0
```

---

# 8. Step 4 — Create the Docker Network

The three containers need a private network.

Create it:

```bash
docker network create \
  --subnet 172.30.0.0/24 \
  v7ai-net
```

Verify:

```bash
docker network inspect v7ai-net
```

The network should use:

```text
172.30.0.0/24
```

The servers will use:

```text
vm1 -> 172.30.0.11
vm2 -> 172.30.0.12
vm3 -> 172.30.0.13
```

### Why fixed IP addresses?

Nginx on vm1 needs predictable backend addresses.

For example:

```nginx
server 172.30.0.12:80;
server 172.30.0.13:80;
```

---

# 9. Step 5 — Docker Compose

Create:

```text
docker/docker-compose.yml
```

```yaml
services:
  vm1:
    image: v7ai-ubuntu:1.0
    container_name: vm1
    hostname: vm1
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
    ports:
      - "2011:22"
      - "80:80"
      - "443:443"
    networks:
      v7ai-net:
        ipv4_address: 172.30.0.11

  vm2:
    image: v7ai-ubuntu:1.0
    container_name: vm2
    hostname: vm2
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
    ports:
      - "2012:22"
    networks:
      v7ai-net:
        ipv4_address: 172.30.0.12

  vm3:
    image: v7ai-ubuntu:1.0
    container_name: vm3
    hostname: vm3
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
    ports:
      - "2013:22"
    networks:
      v7ai-net:
        ipv4_address: 172.30.0.13

networks:
  v7ai-net:
    external: true
```

### Why expose ports 80 and 443 only on vm1?

vm1 is the public entry point.

The browser should reach:

```text
Host -> vm1 -> Nginx -> backend
```

The backend servers remain available through the internal Docker network.

---

# 10. Step 6 — Start the Environment

Run:

```bash
docker compose -f docker/docker-compose.yml up -d
```

Check:

```bash
docker ps
```

Expected:

```text
vm1
vm2
vm3
```

Check IP addresses:

```bash
docker inspect vm1
docker inspect vm2
docker inspect vm3
```

Or:

```bash
docker network inspect v7ai-net
```

---

# 11. Step 7 — Verify SSH

Check the published SSH ports:

```bash
docker ps
```

Expected:

```text
vm1 -> 2011
vm2 -> 2012
vm3 -> 2013
```

Test:

```bash
ssh -p 2011 ansible_user@127.0.0.1
ssh -p 2012 ansible_user@127.0.0.1
ssh -p 2013 ansible_user@127.0.0.1
```

The actual SSH username and authentication method must match the Docker image configuration.

---

# 12. Step 8 — Ansible Inventory

Create:

```text
ansible/inventory.ini
```

```ini
[servers]
vm1 ansible_host=127.0.0.1 ansible_port=2011
vm2 ansible_host=127.0.0.1 ansible_port=2012
vm3 ansible_host=127.0.0.1 ansible_port=2013

[servers:vars]
ansible_user=ansible_user
ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

### Why?

Ansible needs to know:

1. Which machines to manage
2. Their IP/hostname
3. SSH port
4. SSH user
5. Private key

Although all three servers use:

```text
127.0.0.1
```

they are differentiated by their SSH ports.

---

# 13. Step 9 — Test Ansible Connectivity

From the project root:

```bash
ansible servers -i ansible/inventory.ini -m ping
```

Expected:

```text
vm1 | SUCCESS
vm2 | SUCCESS
vm3 | SUCCESS
```

You can also run:

```bash
ansible servers -i ansible/inventory.ini -b -m ping
```

If all three return:

```text
"ping": "pong"
```

the Ansible connection is working.

---

# 14. Important SSH Troubleshooting

One problem encountered during implementation was:

```text
Permission denied (publickey,password)
```

Check the container:

```bash
docker exec vm1 id ansible_user
```

If you get:

```text
no such user
```

then the Docker image does not contain the expected Ansible user.

Check:

```bash
docker exec vm1 ls -la /home/
```

Also check SSH:

```bash
docker exec vm1 ps aux | grep '[s]shd'
```

Expected:

```text
/usr/sbin/sshd -D
```

The SSH user configured in the Ansible inventory must actually exist inside the container.

---

# 15. Step 10 — Ansible Playbook

The main playbook should include the roles:

```yaml
---
- name: Configure V7AI servers
  hosts: servers
  become: true

  roles:
    - common
    - ssh
    - firewall
    - nginx
    - app
    - docker
```

Save as:

```text
ansible/site.yml
```

---

# 16. Step 11 — Common Role

The common role handles baseline server configuration.

Typical tasks:

```yaml
- name: Update apt package cache
  ansible.builtin.apt:
    update_cache: true

- name: Install baseline packages
  ansible.builtin.apt:
    name:
      - curl
      - vim
      - git
      - openssl
    state: present
```

### Why?

Every server should have a predictable baseline before application configuration begins.

---

# 17. Step 12 — SSH Hardening

The SSH role disables:

```text
Password authentication
Root login
```

This improves SSH security.

After applying the role, verify:

```bash
sshd -T | grep -E 'passwordauthentication|permitrootlogin'
```

Expected values should reflect the hardened configuration.

---

# 18. Step 13 — Firewall

Install UFW:

```yaml
- name: Install UFW
  ansible.builtin.apt:
    name: ufw
    state: present
```

Allow SSH:

```yaml
- name: Allow SSH
  community.general.ufw:
    rule: allow
    port: "22"
    proto: tcp
```

Allow HTTP:

```yaml
- name: Allow HTTP
  community.general.ufw:
    rule: allow
    port: "80"
    proto: tcp
```

For vm1, HTTPS should also be allowed:

```text
443/tcp
```

Enable the firewall only after required access has been allowed.

---

# 19. Step 14 — Install Nginx

The Nginx role installs Nginx:

```yaml
- name: Install Nginx
  ansible.builtin.apt:
    name: nginx
    state: present
    update_cache: true
```

Verify:

```bash
nginx -v
```

---

# 20. Step 15 — Application Directory

The application is stored at:

```text
/opt/v7ai-app
```

Create it using Ansible:

```yaml
- name: Create application directory
  ansible.builtin.file:
    path: /opt/v7ai-app
    state: directory
    owner: root
    group: root
    mode: '0755'
```

---

# 21. Step 16 — Application Page

The application page is deployed with a Jinja2 template.

Example:

```html
<!DOCTYPE html>
<html>
<head>
    <title>V7AI Application</title>
</head>
<body>
    <h1>V7AI Application</h1>
    <p>Deployed using Ansible.</p>
    <p>Server: {{ ansible_hostname }}</p>
</body>
</html>
```

This is useful because the same template produces different output:

```text
Server: vm2
```

and:

```text
Server: vm3
```

That makes load-balancing verification easy.

---

# 22. Step 17 — Backend Nginx Configuration

On vm2 and vm3, Nginx serves:

```text
/opt/v7ai-app
```

Example:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name _;

    root /opt/v7ai-app;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

---

# 23. Step 18 — Verify Backend Servers

From vm1:

```bash
curl -i http://172.30.0.12/
```

and:

```bash
curl -i http://172.30.0.13/
```

Expected:

```text
HTTP/1.1 200 OK
```

and the page should identify the server:

```text
Server: vm2
```

or:

```text
Server: vm3
```

---

# 24. Step 19 — Generate TLS Certificate

For this assessment, a self-signed certificate can be used.

Generate:

```bash
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout public.vm1.local.key \
  -out public.vm1.local.crt \
  -subj "/CN=public.vm1.local"
```

Place the files in:

```text
ansible/roles/nginx/files/tls/
```

Expected:

```text
public.vm1.local.crt
public.vm1.local.key
```

---

# 25. Verify the Certificate

On vm1:

```bash
openssl x509 \
  -in /etc/nginx/tls/public.vm1.local.crt \
  -noout \
  -subject \
  -issuer \
  -dates
```

Example:

```text
subject=CN = public.vm1.local
issuer=CN = public.vm1.local
```

The issuer is the same as the subject because the certificate is self-signed.

---

# 26. Verify Certificate and Private Key Match

Run:

```bash
openssl x509 -noout -modulus \
  -in /etc/nginx/tls/public.vm1.local.crt | openssl md5

openssl rsa -noout -modulus \
  -in /etc/nginx/tls/public.vm1.local.key | openssl md5
```

Both hashes must match.

Example:

```text
MD5(stdin)= 4b62b5d0ad0e54985ee6ae17418192b3
MD5(stdin)= 4b62b5d0ad0e54985ee6ae17418192b3
```

---

# 27. Step 20 — Reverse Proxy on vm1

vm1 acts as the public Nginx reverse proxy.

Backend definition:

```nginx
upstream app_backend {
    server 172.30.0.12:80;
    server 172.30.0.13:80;
}
```

HTTP redirects to HTTPS:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name public.vm1.local;

    return 301 https://$host$request_uri;
}
```

HTTPS server:

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name public.vm1.local;

    ssl_certificate /etc/nginx/tls/public.vm1.local.crt;
    ssl_certificate_key /etc/nginx/tls/public.vm1.local.key;

    location /vm2/ {
        proxy_pass http://172.30.0.12/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /vm3/ {
        proxy_pass http://172.30.0.13/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /app/ {
        proxy_pass http://app_backend/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location / {
        root /opt/v7ai-app;
        index index.html;
        try_files $uri $uri/ =404;
    }
}
```

---

# 28. Why Use an Upstream?

This:

```nginx
upstream app_backend {
    server 172.30.0.12:80;
    server 172.30.0.13:80;
}
```

creates a backend pool.

Then:

```nginx
proxy_pass http://app_backend/;
```

allows Nginx to distribute requests between the backend servers.

---

# 29. Step 21 — Nginx Configuration Validation

Always validate before reloading:

```bash
nginx -t
```

Expected:

```text
syntax is ok
test is successful
```

Then reload:

```bash
nginx -s reload
```

or use the Ansible handler:

```yaml
- name: Reload Nginx
  ansible.builtin.service:
    name: nginx
    state: reloaded
```

---

# 30. Step 22 — Verify HTTPS

On vm1:

```bash
curl -k -I https://127.0.0.1/
```

Expected:

```text
HTTP/1.1 200 OK
```

The `-k` option is required because the certificate is self-signed and therefore not trusted by the normal system CA store.

---

# 31. Step 23 — Verify vm2 Routing

Run:

```bash
curl -k -s https://127.0.0.1/vm2/ | grep "Server:"
```

Expected:

```text
<p>Server: vm2</p>
```

---

# 32. Step 24 — Verify vm3 Routing

Run:

```bash
curl -k -s https://127.0.0.1/vm3/ | grep "Server:"
```

Expected:

```text
<p>Server: vm3</p>
```

This proves vm1 can reverse proxy to both backend servers.

---

# 33. Step 25 — Verify Load Balancing

Run repeatedly:

```bash
for i in {1..10}; do
  curl -k -s https://127.0.0.1/app/ | grep "Server:"
done
```

You may see:

```text
Server: vm2
Server: vm3
Server: vm2
Server: vm3
```

The exact order is not guaranteed.

The important point is that requests can reach both backend servers.

---

# 34. Step 26 — Check Listening Ports

On vm1:

```bash
ss -lntp | grep -E ':80 |:443 '
```

Expected:

```text
0.0.0.0:80
0.0.0.0:443
```

On vm2/vm3:

```bash
ss -lntp | grep ':80 '
```

Expected:

```text
0.0.0.0:80
```

---

# 35. Step 27 — Verify Docker Port Mapping

On the host:

```bash
docker ps
```

vm1 should expose:

```text
0.0.0.0:80->80/tcp
0.0.0.0:443->443/tcp
0.0.0.0:2011->22/tcp
```

vm2:

```text
0.0.0.0:2012->22/tcp
```

vm3:

```text
0.0.0.0:2013->22/tcp
```

---

# 36. Browser Access

The intended architecture exposes HTTP/HTTPS through vm1.

If the host machine can reach the published Docker ports, the browser can use:

```text
http://127.0.0.1
```

or:

```text
https://127.0.0.1
```

For the hostname:

```text
public.vm1.local
```

add:

```text
127.0.0.1 public.vm1.local
```

to the host's hosts file.

Linux:

```bash
sudo nano /etc/hosts
```

Windows:

```text
C:\Windows\System32\drivers\etc\hosts
```

Then:

```text
127.0.0.1 public.vm1.local
```

Because the certificate is self-signed, the browser will show a certificate warning.

---

# 37. Important Browser Troubleshooting

If:

```text
curl inside vm1 -> works
```

but:

```text
browser -> 127.0.0.1 -> timeout
```

check Docker port publishing:

```bash
docker ps
```

vm1 must show:

```text
0.0.0.0:80->80/tcp
0.0.0.0:443->443/tcp
```

Also check:

```bash
docker port vm1
```

Expected:

```text
80/tcp -> 0.0.0.0:80
443/tcp -> 0.0.0.0:443
```

Check from the host:

```bash
curl -I http://127.0.0.1
```

and:

```bash
curl -k -I https://127.0.0.1
```

If host access fails but access from inside vm1 works, the issue is outside Nginx itself and should be investigated at the Docker/host networking layer.

---

# 38. Ansible Idempotency

Run the playbook:

```bash
ansible-playbook -i ansible/inventory.ini ansible/site.yml
```

Run it again.

The second execution should ideally show mostly:

```text
ok
```

with:

```text
changed=0
```

This is important because Ansible is designed to describe the desired state rather than blindly execute commands every time.

Example:

```text
vm1 : changed=0
vm2 : changed=0
vm3 : changed=0
```

---

# 39. Common Problem — Nginx Config Valid but Not Running

It is possible for:

```bash
nginx -t
```

to succeed while Nginx is not actually running.

Check:

```bash
ps aux | grep '[n]ginx'
```

and:

```bash
ss -lntp | grep -E ':80 |:443 '
```

If nothing is listening, start Nginx:

```bash
nginx
```

Then verify again.

---

# 40. Common Problem — TLS Certificate Missing

Error:

```text
cannot load certificate
"/etc/nginx/tls/public.vm1.local.crt"
```

Check:

```bash
ls -l /etc/nginx/tls/
```

Expected:

```text
public.vm1.local.crt
public.vm1.local.key
```

On the Ansible controller:

```bash
ls -l ansible/roles/nginx/files/tls/
```

Make sure the files exist before running the playbook.

---

# 41. Common Problem — Ansible Verification Fails on vm1

If vm1 uses HTTPS with a self-signed certificate, this can fail:

```yaml
ansible.builtin.uri:
  url: http://127.0.0.1
```

because vm1's port 80 may redirect to HTTPS.

Use:

```yaml
- name: Verify application is responding
  ansible.builtin.uri:
    url: "{{ 'https://127.0.0.1' if inventory_hostname == 'vm1' else 'http://127.0.0.1' }}"
    status_code: 200
    return_content: true
    validate_certs: false
```

The important setting for the self-signed certificate is:

```yaml
validate_certs: false
```

This should be understood as an assessment/lab setting. In production, a properly trusted certificate should be used.

---

# 42. Common Problem — vm1 Becomes Unreachable

If:

```text
vm1 | UNREACHABLE
```

check:

```bash
docker ps
```

Then:

```bash
docker exec vm1 ps aux | grep '[s]shd'
```

Check the published SSH port:

```bash
docker port vm1
```

Then test:

```bash
ssh -p 2011 ansible_user@127.0.0.1
```

If the container was recreated, verify that the expected user and SSH configuration still exist.

---

# 43. Assessment Verification Checklist

## Docker

```bash
docker ps
docker network inspect v7ai-net
docker images
```

## Ansible

```bash
ansible servers -i ansible/inventory.ini -m ping
```

## Nginx

```bash
ansible servers -i ansible/inventory.ini -b -m shell -a "nginx -t"
```

## Backend vm2

```bash
ansible vm2 -b -m shell -a "curl -s http://127.0.0.1/"
```

## Backend vm3

```bash
ansible vm3 -b -m shell -a "curl -s http://127.0.0.1/"
```

## HTTPS

```bash
ansible vm1 -b -m shell -a "curl -k -I https://127.0.0.1/"
```

## vm2 reverse proxy

```bash
ansible vm1 -b -m shell -a \
"curl -k -s https://127.0.0.1/vm2/ | grep 'Server:'"
```

## vm3 reverse proxy

```bash
ansible vm1 -b -m shell -a \
"curl -k -s https://127.0.0.1/vm3/ | grep 'Server:'"
```

## Load balancing

```bash
ansible vm1 -b -m shell -a \
"for i in {1..10}; do curl -k -s https://127.0.0.1/app/ | grep 'Server:'; done"
```

## TLS

```bash
ansible vm1 -b -m shell -a \
"openssl x509 -in /etc/nginx/tls/public.vm1.local.crt -noout -subject -issuer -dates"
```

## Certificate/key match

```bash
ansible vm1 -b -m shell -a \
"openssl x509 -noout -modulus -in /etc/nginx/tls/public.vm1.local.crt | openssl md5; \
openssl rsa -noout -modulus -in /etc/nginx/tls/public.vm1.local.key | openssl md5"
```

---

# 44. Final Architecture

The completed environment works like this:

```text
                       Browser / Client
                              |
                              |
                         HTTPS :443
                              |
                              v
                    +-------------------+
                    |       vm1         |
                    |      Nginx        |
                    | Reverse Proxy     |
                    | TLS Termination   |
                    +---------+---------+
                              |
               +--------------+--------------+
               |                             |
          /vm2/ |                        /vm3/
               |                             |
               v                             v
       +---------------+             +---------------+
       |      vm2      |             |      vm3      |
       | Nginx :80     |             | Nginx :80     |
       | Application   |             | Application   |
       +---------------+             +---------------+
               ^                             ^
               |                             |
               +-------------+---------------+
                             |
                        /app/
                             |
                     Nginx upstream
                     load balancing
```

---

# 45. What This Project Demonstrates

This assessment demonstrates practical knowledge of:

### Docker

* Building custom images
* Running multiple containers
* Docker Compose
* Static container IPs
* Custom Docker networks
* Port mapping

### Linux

* SSH
* Processes
* Ports
* File permissions
* Services
* Networking
* UFW

### Ansible

* Inventory
* SSH-based automation
* `become`
* Roles
* Tasks
* Handlers
* Templates
* File deployment
* Idempotency

### Nginx

* Static web serving
* Reverse proxy
* Upstream backend groups
* HTTP → HTTPS redirect
* TLS termination
* Path-based routing
* Load balancing

### Security

* SSH hardening
* Firewall
* TLS
* Private key permissions
* Self-signed certificates for lab use

---

# 46. Recommended Demo Sequence

For an interview or assessment demonstration, show the project in this order:

### 1. Show architecture

```text
vm1 = reverse proxy
vm2 = backend
vm3 = backend
```

### 2. Show Docker

```bash
docker ps
docker network inspect v7ai-net
```

### 3. Show Ansible inventory

```bash
cat ansible/inventory.ini
```

### 4. Test connectivity

```bash
ansible servers -i ansible/inventory.ini -m ping
```

### 5. Run the playbook

```bash
ansible-playbook -i ansible/inventory.ini ansible/site.yml
```

### 6. Show idempotency

Run it again and demonstrate:

```text
changed=0
```

where applicable.

### 7. Show Nginx

```bash
ansible vm1 -b -m shell -a "nginx -t"
```

### 8. Show HTTPS

```bash
curl -k -I https://127.0.0.1
```

### 9. Show routing

```bash
curl -k https://127.0.0.1/vm2/
curl -k https://127.0.0.1/vm3/
```

### 10. Show load balancing

```bash
for i in {1..10}; do
  curl -k -s https://127.0.0.1/app/ | grep "Server:"
done
```

Explain that the response identifies the backend server.

---

# 47. Stopping the Lab

When finished working for the day, stop the containers:

```bash
docker compose -f docker/docker-compose.yml stop
```

This is preferable to deleting the containers when you simply want to pause the assessment.

Verify:

```bash
docker ps
```

To continue later:

```bash
docker compose -f docker/docker-compose.yml start
```

Then:

```bash
docker ps
```

---

# 48. Do Not Use `docker compose down` When You Only Want a Break

This:

```bash
docker compose stop
```

means:

```text
Stop containers
Keep containers
Keep configuration
Resume later
```

Whereas:

```bash
docker compose down
```

removes the Compose-created containers and can require recreating the environment.

For a temporary pause, use:

```bash
docker compose -f docker/docker-compose.yml stop
```

---

# 49. Final Success Criteria

The assessment can be considered functionally complete when:

* ✔  Three containers are running
* ✔  Containers have static private IPs
* ✔  SSH connectivity works
* ✔  Ansible can manage all servers
* ✔  Nginx is installed
* ✔  Application files are deployed
* ✔  vm2 serves the application
* ✔  vm3 serves the application
* ✔  vm1 acts as reverse proxy
* ✔  TLS certificate is installed
* ✔  TLS private key is installed securely
* ✔  Nginx configuration passes `nginx -t`
* ✔  HTTPS works on vm1
* ✔  `/vm2/` reaches vm2
* ✔  `/vm3/` reaches vm3
* ✔  `/app/` uses the backend pool
* ✔  Backend responses identify vm2/vm3
* ✔  Ansible playbook is repeatable
* [x] Firewall is configured
* [x] SSH is hardened
* [x] Docker and Ansible configuration are reproducible

---

# 50. Key DevOps Concepts Learned

The most important lesson from this assessment is that the individual technologies are connected:

```text
Docker
   ↓
Creates isolated servers
   ↓
Docker Network
   ↓
Provides private communication
   ↓
Ansible
   ↓
Automates server configuration
   ↓
Nginx
   ↓
Provides web serving + reverse proxy
   ↓
TLS
   ↓
Provides HTTPS
   ↓
Upstream
   ↓
Provides backend load balancing
```

The final result is a small but realistic infrastructure environment that demonstrates **containerization + configuration management + web infrastructure + security + automation**.
