---
title: "Installing Azure CLI to Chromebook"
date: 2017-09-29T23:57:45Z
description: "It's possible to install Azure's CLI to a Chromebook if
you're willing to follow some manual steps."
categories: [ "work" ]
tags: [ "azure", "az", "chromebook", "termux", "python", "virtualenv" ]
---

I was asked to contribute to a white-paper for Azure container
services and as part of that task I naturally wanted to install the
CLI. No problem to do so on Linux and Windows, but I encountered some
problems when doing so for Termux and Chromebook. This post details
how to resolve the issue on a Chromebook or other locked down
environment.
<!--more-->

During installation the Azure CLI script assumes it can
install dependencies to a location forbidden by Android (and
Termux). On a hunch I guessed this could be resolved by a user folder
virtual environment, instead of a system install, and with a little
investigation of exceptions, I was able to get az cli working just
fine.

## 1. Install the python packages into a virtual environment

These steps are essentially what the standard Azure CLI install does, but this
method installs `ffi` and other dependencies into a virtualenv
to avoid problems with Android file permissions.

```bash
$ pkg install openssl-dev libffi-dev python-dev python
$ pip install --user virtualenv
$ virtualenv /home/lib/azure-cli
$ /home/lib/azure-cli/bin/pip install cffi
$ /home/lib/azure-cli/bin/pip install azure-cli
$ /home/lib/azure-cli/bin/pip install --upgrade --force-reinstall azure-nspkg azure-mgmt-nspkg
```

## 2. Add shell script for command completion

Create `az.completion` in `/home/lib/azure-cli`, or wherever you
installed Azure CLI files to have command completion.

{{< gist memes ebac7e28a7981e67b73b4080e1f45d27 "az.completion" >}}

Note that IFS in line 2 is a Ctrl-K; download the raw file from gist
if needed.

## 3. Enable bash command completion

Edit `~/.bashrc` to include completion support for Azure CLI; in my
example below I add Azure CLI completion on the last line.

{{< gist memes ebac7e28a7981e67b73b4080e1f45d27 ".bashrc" >}}

## 4. Create a shell script to invoke Azure CLI

Create the script `az` and add it to path; mine is in `/home/bin/az`

{{< gist memes ebac7e28a7981e67b73b4080e1f45d27 "az" >}}
