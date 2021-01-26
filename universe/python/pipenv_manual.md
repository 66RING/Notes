---
title: pip venv tools usage
date: 2019-1-3
tags: 
- python
- pipenv
- tools
---

# Virtualenvwrapper

## Install

``` 
pip install virtualenvwrapper
```

Some environment variable

``` bash
export WORKON_HOME=~/Envs  # specify home of venv

export VIRTUALENVWRAPPER_PYTHON=/usr/local/bin/python3  # specify defaul python version 

source /usr/local/bin/virtualenvwrapper.sh  # active virtualenvwrapper when open terminal
```


## Usage

- New venv
    * `mkvirtualenv your_venv_name --python=python_version`
- Active/Switch between venvs
    * `workon your_venv_name`
- Quit
    * `deactivate`
- Delete
    * `rmvirtualenv your_venv_name`
- List All
    * `lsvirtualenv `
- Hook
    * This hook is sourced after this virtualenv is activated
    * Specify in `your_venv_home/your_venv_name/bin/postactive`


# Pipenv Usage

## Install 

``` 
pip install pipenv
```


## Usage 

Quick Start

``` 
cd     myproject
pipenv install  # create virtual environment
pipenv install request  #  or install module
pipenv --venv  # show location of this virtual environmenfuzzy findert
pipenv --py    # show location of this python interpreter
pipenv shell  # activate
```

Create a new project using Python 3.7, specifically

```
pipenv --python 3.7
```

Remove project virtualenv (inferred from current directory):

```
pipenv --rm
```

Install all dependencies for a project (including dev):

```
pipenv install --dev
```

Create a lockfile containing pre-releases:

```
pipenv lock --pre
```

Show a graph of your installed dependencies:

```
pipenv graph
```

Check your installed dependencies for security vulnerabilities:

```
pipenv check
```

Install a local setup.py into your virtual environment/Pipfile:

```
pipenv install -e .
```

Use a lower-level pip command:

```
pipenv run pip freeze
```


## Activate

``` 
pipenv shell
python --version
```

## Use pypi-mirror

``` 
pipenv install --pypi-mirror https://mirrors.aliyun.com/pypi/simple
```

OR edit Pipfile

``` 
[[source]]

url = "https://mirrors.aliyun.com/pypi/simple"
verify_ssl = true
name = "pypi"
```




