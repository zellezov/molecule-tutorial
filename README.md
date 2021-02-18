# Testing Ansible roles with Molecule

This is an example of Molecule testing setup for Ansible role.  
Execution in following configurations will be covered:

- Running tests on a local docker container (using Testinfra or GOSS as testing framework)
- Running tests on a local k8s cluster
- Running tests on a remote k8s cluster

The Molecule framework is under active development and it gets updated so fast that all the tutorials I have followed were outdated to different extent, so I decided there's no reason not to add one more. =)

See the links section for some great tutorials that have been used, however in this tutorial I'll describe few small adjustments one need to consider in order to get this thing working (at least, as of beginning of 2021).

This setup has been developed and verified on macOS Catalina (10.15.7).

## Required software

To execute tests in local docker container and in local k8s cluster, make sure you have following software installed (use [brew](https://brew.sh/) to install missing components):

- ansible 2.10.5
- molecule 3.2.3
- docker 20.10.2 (build 2291f61)
- docker desktop 3.1.0 (51484)
- minikube v1.17.1
- pyenv 1.2.22
- python 3.9.1 (install Python 3.9.1 using [pyenv](https://realpython.com/intro-to-pyenv/#using-pyenv-to-install-python))
- pipenv 2020.11.15

_In this tutorial the latest version of each component has been used (at the time of writing), the exact version provided just for reference._

## Prepare the environment

In this tutorial I will use `pipenv` as virtual environment management tool. Feel free to choose your favourite tool instead.

Create virtual environment using `python 3.9.1` and activate created environment:

```bash
pipenv --python 3.9.1
pipenv shell
```

Following packages needs to be installed for test writing and execution (install parts that suits your needs, or install all packages):

1. For Ansible installation

```bash
pipenv install wheel ansible
```

2. For Molecule installation with Docker driver support

```bash
pipenv install "molecule[docker]"
```

3. For writing and linting tests using Python as verifier

```bash
pipenv install pytest-testinfra pylint
```

4. For writing tests using GOSS as verifier

```bash
pipenv install molecule-goss
```

5. For linting Ansible playbook and GOSS files

```bash
pipenv install "ansible-lint[yamllint]"
```

6. For running tests on k8s

```bash
pipenv install openshift
```

_If some package installation fails, make sure it is not a beta package, otherwise add `--pre` flag to allow beta package installation._

## Prepare Ansible role for testing

All examples will use the same Ansible role setup. The main difference would be the selection of testing framework and Molecule driver/verifier setup.  
This part would explain the typical Ansible role setup to be used.

### Create role template

Use Molecule to create a new Ansible role that will be used for testing:

```bash
molecule init role python-test -d docker
```

_Choose the name of your choice instead of `python-test`, we will begin with python tests example, so we'll use this name._  
_Specify driver with flag `-d docker`._

Go to the created directory:

```bash
cd python-test
```

Now you see following default file structure created automatically by Molecule:

```bash
/python-test
├── .travis.yml
├── .yamllint
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
molecule init scenario -r role-name
```

### Configure role

Now it is necessary to configure packages and services that would be the subject of tests.  
I have no idea why, but any centOS7 image other than [milcom/centos7-systemd](https://hub.docker.com/r/milcom/centos7-systemd) (as used in DigitalOcean tutorial) had the problems with `systemd` during test execution, so I'll use the same image in this tutorial as well.

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

#### Setup Molecule for test execution using TestInfra

Now we need to specify docker image, test executor and linters. To do that, modify the `molecule.yml` file:

```yaml
# ~/python-test/molecule/default/molecule.yml
---
dependency:
  name: galaxy # default dependency manager
driver:
  name: docker # to execute tests in docker container
platforms:
  - name: centos7
    image: milcom/centos7-systemd # specific docker image to be used
    privileged: true
provisioner:
  name: ansible # default provisioner
verifier:
  name: testinfra # tests would be invoked by specified verifier
lint: | # pipe needed here to provide list of linters to use
  yamllint . 
  ansible-lint .
  pylint molecule/default/tests
```

Refer to Molecule [documentation](https://molecule.readthedocs.io/en/latest/configuration.html) for detailed description of all available parameters.

#### Add some tests

Create new folder `tests` inside the `molecule` folder, and add two files: `__init__.py` and `test_default.py`:

```bash
molecule
└── default
    ├── converge.yml
    ├── molecule.yml
    ├── tests
    │   ├── __init__.py
    │   └── test_default.py
    └── verify.yml
```

Add some tests to `test_default.py` file to check that ansible roles tasks have been executed successfully:

- verify packages are installed
- verify services are started and enabled
- verify server return specified template

```python
import os
import pytest

import testinfra.utils.ansible_runner

testinfra_hosts = testinfra.utils.ansible_runner.AnsibleRunner(
    os.environ['MOLECULE_INVENTORY_FILE']).get_hosts('all')


@pytest.mark.parametrize('pkg', [
  'httpd',
  'firewalld'
])
def test_pkg(host, pkg):
    package = host.package(pkg)

    assert package.is_installed


@pytest.mark.parametrize('svc', [
  'httpd',
  'firewalld'
])
def test_svc(host, svc):
    service = host.service(svc)

    assert service.is_running
    assert service.is_enabled


@pytest.mark.parametrize('file, content', [
  ("/etc/firewalld/zones/public.xml", "<service name=\"http\"/>"),
  ("/var/www/html/index.html", "Managed by Ansible")
])
def test_files(host, file, content):
    file = host.file(file)

    assert file.exists
    assert file.contains(content)
```

#### Running tests

Now you can try to run tests:

```bash
molecule test
```

Most likely, the tests would fail for the first time. Check that all packages have been installed and none are missing, and fix linting errors (or turn linters off, which is an undesirable option). After that, try to run tests again. If you see `pytest` summary in output, this means everything works:

```bash
INFO     Running default > verify
INFO     Executing Testinfra tests found in /Users/zellezov/github/zellezov/molecule-tutorial/python-test/molecule/default/tests/...
============================= test session starts ==============================
platform darwin -- Python 3.9.1, pytest-6.2.2, py-1.10.0, pluggy-0.13.1
rootdir: /Users/zellezov
plugins: testinfra-6.1.0
collected 6 items

molecule/default/tests/test_default.py ......                            [100%]

============================== 6 passed in 10.24s ==============================
INFO     Verifier completed successfully.
```

### GOSS tests

#### What is GOSS?

> [Goss](https://github.com/aelsabbahy/goss) is a YAML based Serverspec alternative tool for validating a server’s configuration. It eases the process of writing tests by allowing the user to generate tests from the current system state. Once the test suite is written they can be executed, waited-on, or served as a health endpoint.

You do not have to install GOSS on your workstation (it has limited support on Win and macOS), but as we are going to run tests in container, just make sure to follow all steps during setup, and you won't be disappointed with the results.

#### Initiate new role 'goss-test'

Create new role template with name `goss-test` and configure it using steps described in the beginning. Tasks, templates and vars should be the same.

#### Setup Molecule for test execution using GOSS

Now we need to specify docker image, test executor and linters. To do that, modify the `molecule.yml` file:

```yaml
# ~/goss-test/molecule/default/molecule.yml
---
dependency:
  name: galaxy # default dependency manager
driver:
  name: docker # to execute tests in docker container
platforms:
  - name: centos7
    image: milcom/centos7-systemd # specific docker image to be used
    privileged: true
provisioner:
  name: ansible # default provisioner
  log: true # log output needed to see GOSS execution results in log
verifier:
  name: goss # tests would be invoked by specified verifier
lint:
  | # pipe needed here to provide list of linters to use, only yaml linters needed for goss
  yamllint .
  ansible-lint .
```

Refer to Molecule [documentation](https://molecule.readthedocs.io/en/latest/configuration.html) for detailed description of all available parameters.

#### Install sudo package

Package `sudo` needs to be installed in container to install GOSS.
Update `converge.yml` file:

```yml
# ~/goss-test/molecule/default/converge.yml
---
- name: Converge
  hosts: all
  tasks:
    - name: Install the package 'sudo'
      yum:
        name: sudo
        state: present
    - name: "Include goss-test"
      include_role:
        name: "goss-test"
```

#### Install GOSS

Specify GOSS installation steps in `verify.yml` file:

```yml
# ~/goss-test/molecule/default/verify.yml
- name: Verify
  hosts: all
  become: true
  vars:
    goss_version: v0.3.2
    goss_arch: amd64
    goss_dst: /usr/local/bin/goss
    goss_sha256sum: 2f6727375db2ea0f81bee36e2c5be78ab5ab8d5981f632f761b25e4003e190ec
    goss_url: "https://github.com/aelsabbahy/goss/releases/download/{{ goss_version }}/goss-linux-{{ goss_arch }}"
    goss_test_directory: /tmp
    goss_format: documentation
  tasks:
    - name: Download and install Goss
      get_url:
        url: "{{ goss_url }}"
        dest: "{{ goss_dst }}"
        sha256sum: "{{ goss_sha256sum }}"
        mode: 0755
      register: download_goss
      until: download_goss is succeeded
      retries: 3

    - name: Copy Goss tests to remote
      copy:
        src: "{{ item }}"
        dest: "{{ goss_test_directory }}/{{ item | basename }}"
      with_fileglob:
        - "{{ lookup('env', 'MOLECULE_VERIFIER_TEST_DIRECTORY') }}/test_*.yml"

    - name: Register test files
      shell: "ls {{ goss_test_directory }}/test_*.yml"
      register: test_files

    - name: Execute Goss tests
      command: "{{ goss_dst }} -g {{ item }} validate --format {{ goss_format }}"
      register: test_results
      with_items: "{{ test_files.stdout_lines }}"

    - name: Display details about the Goss results
      debug:
        msg: "{{ item.stdout_lines }}"
      with_items: "{{ test_results.results }}"

    - name: Fail when tests fail
      fail:
        msg: "Goss failed to validate"
      when: item.rc != 0
      with_items: "{{ test_results.results }}"
```

#### Add some tests

Create new folder `tests` inside the `molecule` folder, and add `test_default.yml`:

```yml
# ~/goss-test/molecule/tests/test_default.yml
---
file:
  "/etc/firewalld/zones/public.xml":
    exists: true
    contains: ['<service name="http" />']
  "/var/www/html/index.html":
    exists: true
    contains: ["Managed by Ansible"]
  "/var/www/html":
    exists: true
    filetype: directory

package:
  httpd:
    installed: true
  firewalld:
    installed: true

service:
  httpd:
    enabled: true
    running: true
  firewalld:
    enabled: true
    running: true
```

#### Running tests

## Running tests on a local k8s cluster

todo

## Running tests on a remote k8s cluster

todo

## Troubleshooting

todo

## Sources and links

- [How To Test Ansible Roles with Molecule on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-test-ansible-roles-with-molecule-on-ubuntu-18-04)
- [Molecule-Kubernetes](https://github.com/bleung/molecule-kubernetes)
