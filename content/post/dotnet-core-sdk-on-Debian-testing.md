+++
title = ".NET Core SDK on Debian Testing"
date = "2017-06-06T13:46:46-07:00"
description = "Installing .NET Core SDK on Debian and resolving a segfault"
categories = [ "work" ]
tags = [ ".net", "linux", "debian", "segfault" ]
aliases = [ "/post/dotnet-core-sdk-on-debian-testing/" ]
+++

# Installing .NET Core SDK on Debian Testing

## tl;dr
* install missing packages libicu52, libssl1.0.0 and liblttng-ust0
* downgrade curl to ``stable`` release 7.38.0-4+deb8u5

>## Update 6/11/2017:-
> Accidentally released the hold on ``curl``/``libcurl3``, and ``dotnet``
> command still works. I think that ``libcurl3`` is only used when
> downloading framework and can be reset when done.

As part of a series of posts on deploying .NET Core applications to
GKE, I had to install .NET Core SDK to my Debian laptop and make it
work correctly. ``curl`` and ``gettext`` packages were already installed, so
only ``libunwind8`` was required.

```bash
$ sudo apt-get install libunwind8
$ mkdir -p ~/lib/.net
$ curl -sSL https://go.microsoft.com/fwlink/?linkid=848826 | tar xvzC ~/lib/.net/
```

After installation, I tried to run ``dotnet`` and it worked, but any
arguments to ``dotnet`` caused a problem.

```bash
$ ~/lib/.net/dotnet

Microsoft .NET Core Shared Framework Host

...

$ ~/lib/.net/dotnet -h
Failed to initialize CoreCLR, HRESULT: 0x80131500
```

I figured that this was because .NET Core SDK is supported on Debian
Stable, and I tend to run a mix of Testing/Sid packages. Quick check
for missing libraries showed a few candidates, and most were easily
resolved, with the exception of liblldb-3.6 for which a package could
not be found.

```bash
$ find ~/lib/.net/ -type f -print0 | xargs -0 ldd | grep 'not found' | sort | uniq
        libcrypto.so.1.0.0 => not found
		libicu18n.so.52 => not found
		libicuuc.so.52 => not found
		liblldb-3.6.so => not found
		liblttng-ust.so.0 => not found
		libssl.so.1.0.0 => not found
$ sudo apt-get install -y libssl1.0.0 libicu52 liblttng-ust0
$ find ~/lib/.net/ -type f -print0 | xargs -0 ldd | grep 'not found' | sort | uniq
        liblldb-3.6.so => not found
$ ~/lib/.net/dotnet -h
.NET Command Line Tools (1.0.4)
Usage: dotnet [host-options] [command] [arguments] [common-options]

...
```

The command works, but trying to create a project generates a segfault. A quick Googling found a bugreport; segfault due to curl/libcurl3 using a more recent version of libssl than .NET Core. Downgrading the package fixed the problem.

```bash
$ sudo apt-get install curl=7.38.0-4+deb8u5 libcurl3=7.38.0-4+deb8u5
$ sudo apt-mark hold curl libcurl3
```
