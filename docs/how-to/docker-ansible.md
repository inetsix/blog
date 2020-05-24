# Ansible using docker

This how-to explains how to configure docker and your shell environment to execute ansible within a container and not from your system. It is useful to use this approach to validate code with different `ansible` version.

- __Github repository__ : [`titom73/docker-ansible`](https://github.com/titom73/docker-ansible)
- __Docker Image__ : [`inetsix/docker-ansible`](https://cloud.docker.com/u/inetsix/repository/docker/inetsix/docker-ansible)


[Repository](https://cloud.docker.com/u/inetsix/repository/docker/inetsix/docker-ansible) we use in this example gives us option to:

* share your ssh agent with the ansible docker
* share your AWS CLI configuration
* Use a specifc version without installation
* Use AWS and Junos componants: `aws cli`, `junos-eznc`, `Juniper.junos` ansible module

Just add these aliases to your `~/.{bash|zsh|...}_aliases` in order to use ansible as it where installed on your computer.

```bash
export DOCKER_ANSIBLE_VERSION=2.5
base_ansible() {
        docker run -it --rm \
            --volume $SSH_AUTH_SOCK:/ssh-agent \
            --env SSH_AUTH_SOCK=/ssh-agent \
            -v ${PWD}:project \
            -v ${HOME}/.ssh/known_hosts:/root/.ssh/known_hosts \
            -w ${PWD} \
            inetsix/docker-ansible:${DOCKER_ANSIBLE_VERSION} $@
}
```

Once this bash function is available, you can build you aliases to mimic ansible:

```bash
alias ansible-update='docker pull inetsix/docker-ansible:${DOCKER_ANSIBLE_VERSION}'
alias ansible-shell='base_ansible bash'
alias ansible='base_ansible ansible'
alias ansible-playbook='base_ansible ansible-playbook'
alias ansible-vault='base_ansible ansible-vault'
alias ansible-galaxy='base_ansible ansible-galaxy'
```

The `export` statement gives option to specify which version of ansible we want to use. This version is equal to [tags](https://cloud.docker.com/u/inetsix/repository/docker/inetsix/docker-ansible/tags) defined during the build

This docker image is built with the following tags (12/2018) for both `alpine` and Debian `jessie` OS:

- `latest`
- `2.3`
- `2.4`
- `2.5`
- `2.6`
- `2.7`

## Usage

Simply use your aliases, and you can override ansible default version when required

```bash
$ DOCKER_ANSIBLE_VERSION=2.5 
$ docker-ansible --version 
ansible 2.5.2
  config file = /Users/titom73/Scripting/ansible.projects/ansible.training.phase2/ansible.cfg
  configured module search path = [u'/root/.ansible/plugins/modules',\
    u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/local/lib/python2.7/site-packages/ansible
  executable location = /usr/local/bin/ansible
  python version = 2.7.14 (default, Apr 27 2018, 09:37:08) [GCC 4.9.2]
```

## Optional parameters

### Export AWS CLI parameters

Assuming you have configured your AWS CLI, you may export your parameters with the following variables:

On your local shell:

```bash
export AWS_DEFAULT_REGION='eu-west-2'
export AWS_PROFILE='your_aws_profile'
```

Then, these variables are exposed to your docker instance by using `--env` keyword. If they are not set, then, nothing is export to your container.

```bash
export DOCKER_ANSIBLE_VERSION=2.5
base_ansible() {
        docker run -it --rm \
            --env ANSIBLE_REMOTE_USER=${USER} \
            --env AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION} \
            --env AWS_PROFILE=${AWS_PROFILE} \
            --volume $SSH_AUTH_SOCK:/ssh-agent \
            --env SSH_AUTH_SOCK=/ssh-agent \
            -v ${PWD}:project \
            -v ${HOME}/.aws/:/root/.aws/ \
            -v ${HOME}/.ssh/known_hosts:/root/.ssh/known_hosts \
            -w ${PWD} \
            inetsix/docker-ansible:${DOCKER_ANSIBLE_VERSION} $@
}
```

### Overwrite ansible commands 
If for some reasons, you need to use your installed ansible version, just use `\\` in front of any ansible command like below:

```bash
$ \ansible --version
ansible 2.5.1
  config file = /Users/titom73/Scripting/ansible.projects/ansible.training.phase2/ansible.cfg
  configured module search path = [u'/Users/titom73/.ansible/plugins/modules',\
     u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/local/lib/python2.7/site-packages/ansible
  executable location = /usr/local/bin/ansible
  python version = 2.7.14 (default, Mar 22 2018, 15:04:47)\
    [GCC 4.2.1 Compatible Apple LLVM 9.0.0 (clang-900.0.39.2)]

$ ansible --version
ansible 2.5.2
  config file = /Users/titom73/Scripting/ansible.projects/ansible.training.phase2/ansible.cfg
  configured module search path = [u'/root/.ansible/plugins/modules',\
     u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/local/lib/python2.7/site-packages/ansible
  executable location = /usr/local/bin/ansible
  python version = 2.7.14 (default, Apr 27 2018, 09:37:08) [GCC 4.9.2]
```