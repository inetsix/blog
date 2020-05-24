# VirtualEnv Wrapper

VirtualEnvWrapper allows user to create python `virtual-environment` and use easy command to create / use / modify / delete

## Installation

Use `python-pip` to install module:

```shell
$ pip install virtualenvwrapper
```

To test in your shell:

- Create folder to store virtual-environments:

```shell
$ mkdir ~/.virtual-envs
$ mkdir ~/Scripting/
```

- Load vars to your shell

```shell
$ export WORKON_HOME=~/.virtual-envs
$ export PROJECT_HOME=~/Scripting/
$ source /usr/local/bin/virtualenvwrapper.sh
```

To make a permanent configuration:

```shell
$ vim ~/.zshrc

export WORKON_HOME=~/.virtual-envs
export PROJECT_HOME=~/Scripting/
source /usr/local/bin/virtualenvwrapper.sh
```

## Commands

1. Create virtual environment

```shell
$ mkvirtualenv my_venv
```

2. List existing environments

```shell
$ lsvirtualenv 
```

3. Load existing environment

```shell
$ workon my_env
```

4. Create project with a new venv

```shell
$ mkproject myproject
```
> Project is created under `PROJECT_HOME` with a dedicated folder and a venv with project's name

5. Leave a venv

```shell
deactivate
```

6. Delete a venv

```shell
$ rmvirtualenv my_venv
```


## Resources

- [List of commands](http://virtualenvwrapper.readthedocs.org/en/latest/command_ref.html)
- [Hooks and customisation](https://virtualenvwrapper.readthedocs.io/en/latest/hooks.html)