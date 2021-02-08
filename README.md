# Testing Ansible roles with Molecule

This is an example of Molecule testing setup for Ansible role.  
Execution in following configurations will be covered:  
- Running tests on a local docker container (using Testinfra or GOSS as testing framework)
- Running tests on a local k8s cluster
- Running tests on a remote k8s cluster

The Molecule framework is under active development and it gets updated so fast that all the tutorials I have followed were outdated to different extent, so I decided there's no reason not to add one more. =)  

See the links section for some great tutorials that have been used, however in this tutorial I'll describe few small adjustments one need to consider in order to get this thing working (at least, as of beginning of 2021).  

This setup has been developed and verified on macOS Catalina (10.15.7) and macOS Big Sur (11.1).

## Required software

To execute tests in local docker container and in local k8s cluster, make sure you have following software installed (use [brew](https://brew.sh/) to install missing components):  
- ansible 2.10.5  
- molecule 3.2.3  
- docker 20.10.2 (build 2291f61)  
- docker desktop 3.1.0 (51484)
- minikube v1.17.1
- pyenv 1.2.22
- python 3.9.1  (install Python 3.9.1 using [pyenv](https://realpython.com/intro-to-pyenv/#using-pyenv-to-install-python))
- pipenv 2020.11.15  

*In this tutorial the latest version of each component has been used (at the time of writing), exact version provided just for reference.*

## Prepare Ansible role for testing

All examples will use the same Ansible role setup. The main difference would be the selection of testing framework.  
This part would explain the Ansible role setup.

### Create role template

Use Molecule to create a new Ansible role that will be used for testing:
```bash
molecule init role python-test -d docker
```
Choose the name of your choice instead of `python-test`, we will begin with python tests example, so we'll use this name.  
Specify driver with flag `-d docker`.

Go to the created directory:
```bash
cd python-test
```

Now you see following file structure created automatically by Molecule:
```bash
├── README.md
├── defaults
├── files
├── handlers
├── meta
├── molecule
├── tasks
├── templates
├── tests
└── vars
```

Alternatively, you can initialize Molecule for existing Ansible role, running following command from within the role directory:
```bash
molecule init scenario -r role-name -d docker
```

### Configure role

Now it is necessary to configure packages and services that would be the subject of tests.  
I have no idea why, but any centOS7 image other than [milcom/centos7-systemd](https://hub.docker.com/r/milcom/centos7-systemd) (used in DigitalOcean tutorial) had the problems with `systemd`, so it will be used in this tutorial as well.

#### Add vars

Add package and service variables to the `main.yml` file in `vars` folder:
```yaml 
# ~/python-test/vars/main.yml
---
pkg_list: # packages to be installed
  - httpd
  - firewalld
svc_list: # services to be started and enabled
  - httpd
  - firewalld
```

#### Add tasks

Specify tasks for Ansible role in `main.yml` file inside `tasks` folder.  
These tasks would ensure all required packages and services are installed, configured and started:
```yaml
# ~/python-test/tasks/main.yml
---
- name: "Ensure required packages are present"
  yum:
    name: "{{ pkg_list }}"
    state: present

- name: "Ensure latest index.html is present"
  template:
    src: index.html.j2
    dest: /var/www/html/index.html

- name: "Ensure httpd service is started and enabled"
  service:
    name: "{{ item }}"
    state: started
    enabled: true
  with_items: "{{ svc_list }}"

- name: "Whitelist http in firewalld"
  firewalld:
    service: http
    state: enabled
    permanent: true
    immediate: true
```

#### Add templates
Create a template in `index.html.j2` file inside `templates` folder.  
This file would describe output to be returned by Apache server:
```html
<!-- ~/python-test/templates/index.html.j2 -->
<div style="text-align: center">
    <h2>Managed by Ansible</h2>
</div>
```

Now the role is created, and we can move on to configuring Molecule for testing.

## Running tests on a local docker container

### Python tests

#### What is TestInfra?

> With [Testinfra](https://testinfra.readthedocs.io/en/latest/) you can write unit tests in Python to test actual state of your servers configured by management tools like Salt, Ansible, Puppet, Chef and so on.  
> Testinfra aims to be a Serverspec equivalent in python and is written as a plugin to the powerful Pytest test engine.  

### GOSS tests

#### What is GOSS?

 > [Goss](https://github.com/aelsabbahy/goss) is a YAML based Serverspec alternative tool for validating a server’s configuration. It eases the process of writing tests by allowing the user to generate tests from the current system state. Once the test suite is written they can be executed, waited-on, or served as a health endpoint.  

You do not have to install GOSS on your workstation (it has limited support on Win and macOS), but as we are going to run tests in container, just make sure to follow all steps during setup, and you won't be disappointed with the results.

## Running tests on a local k8s cluster

todo

## Running tests on a remote k8s cluster

todo

## Troubleshooting

todo

## Sources and links
- [How To Test Ansible Roles with Molecule on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-test-ansible-roles-with-molecule-on-ubuntu-18-04)  
- [Molecule-Kubernetes](https://github.com/bleung/molecule-kubernetes)
