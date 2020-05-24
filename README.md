[![Publication status](https://gitlab.com/titom73/inetsix-docs/badges/master/pipeline.svg)](https://gitlab.com/titom73/inetsix-docs/commits/master)

# Inetsix Documentation website builder

This repository provides all the material required to build a [`mkdocs`]() website.


## Development phase

Install mkdocs:

```shell
$ pip install mkdocs
```

Repository structure:

```shell
inetsix-docs/
    mkdocs.yml
    docs/
    	CNAME
    	index.md
    	<content to publish>

inetsix.github.io/
```

To load develpment server:

```shell
$ cd inetsix-docs
$ mkdocs serve
```

## Publication process

```shell
$ cd inetsix.github.io/
$ mkdocs gh-deploy --config-file ../inetsix-docs/mkdocs.yml --remote-branch master
INFO    -  Cleaning site directory
INFO    -  Building documentation to directory: ../inetsix-docs/site
WARNING -  Version check skipped: No version specificed in previous deployment.
INFO    -  Copying '../inetsix-docs/site' to 'master' branch and pushing to GitHub.
INFO    -  Based on your CNAME file, your documentation should be available shortly at: http://www.website.net
INFO    -  NOTE: Your DNS records must be configured appropriately for your CNAME URL to work.
```