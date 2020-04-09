---
id: install
title: Installation
---

## System Requirements

Watchman is known to compile and pass its test suite on:

- Linux systems with `inotify`
- macOS (uses `FSEvents` on 10.7+, `kqueue(2)` on earlier versions)
- Windows x64 on Windows 7, Windows Server 2012 R2 and later is currently in
  **beta** status.

Watchman used to support the following systems, but no one is actively
maintaining them. The core of the code should be OK, but they likely don't
build. We'd love it if someone would step forward to maintain them:

- BSDish systems (FreeBSD 9.1, OpenBSD 5.2) that have the `kqueue(2)` facility
- Illumos and Solaris style systems that have `port_create(3C)`

Watchman relies on the operating system facilities for file notification,
which means that you will likely have very poor results using it on any kind
of remote or distributed filesystem.

Watchman does not currently support any other operating system not covered by
the list above.

## Download for Windows (Beta)

Watchman is considered to be in **beta** status for Windows but has a
reasonably sized group of users depending on it already, and we expect to
remove the beta label in the coming months.

Watchman was built to support Windows Server 2012 R2 and later, but has had
community-provided patches that enable support for Windows 7 and later.

At this time, we recommend running the latest master build of watchman on
Windows; see below for binary downloads!

If you encounter issues with the Windows version of watchman, please report
them via GitHub!
[You can find the list of known Windows issues here](https://github.com/facebook/watchman/issues?utf8=%E2%9C%93&q=is%3Aopen+Windows).

## Binary Downloads for Linux, macOS and Windows (Beta)

We use GitHub Actions to build binaries for the following platforms:

- macOS
- Ubuntu 18.04
- Windows

GitHub Actions doesn't provide a way to link directly to these builds at this
time, so you will need to follow these steps to download the binaries:

- [Navigate to recent CI builds](https://github.com/facebook/watchman/actions?query=is%3Asuccess+event%3Apush+branch%3Amaster)
- Click on the `CI` label

![](/watchman/img/ci-link.png)

- Look in the top right of the resulting page for the `Artifacts` drop down

![](/watchman/img/artifacts-download.png)

- Click it to reveal the download artifacts link
- Click that and extract the zip file
- For Windows users:
  - Copy `windows\bin` directory somewhere and add it to your `PATH`
    environment variable.

## Build/Install

### Installing on macOS or Linux via Homebrew

To build the most recent release currently tracked by
[Homebrew](http://brew.sh/):

```bash
$ brew update
$ brew install watchman
```

To install the latest build from github:

```bash
$ brew install --HEAD watchman
```

### Installing on macOS via macports

To install the package maintained by [MacPorts](https://www.macports.org):

```bash
$ sudo port install watchman
```

### Installing from source

You can use these steps below to get watchman built. You will need
`libssl-dev`, `autoconf`, `automake` and `libtool` (or `glibtool` on macOS).

You may optionally build watchman without `pcre` and `python` support (see
configuration options below).

For python support, you will need `setuptools` and may need to install a
`python-dev` or `python-devel` package. To build the C++ client library you
will need to install `libfolly`.

See below for some more information on options to configure your build.

```bash
$ # use the latest stable release
$ git clone https://github.com/facebook/watchman.git -b v4.9.0 --depth 1
$ cd watchman
$ ./autogen.sh
$ ./configure
$ make
$ sudo make install
```

### Compile Time Configuration Options

Our configure script accepts all the standard options, but there are a couple
that are specific to watchman that might be relevant to your needs:

```
--enable-conffile=PATH  Use PATH as the default configuration file name.
                        Default is /etc/watchman.json

--enable-statedir=PATH  Use PATH as the default for state, log files
                        and sockets instead of using your system tempdir

--enable-lenient  Turn off more pedantic levels of warnings
                  and compilation checks

--enable-stack-protector  Enable stack protection in the same
                          way that rpmbuild does on some systems.

--enable-cppclient  Enable build of the C++ client library. This is built by
                    default if Folly is available.

--with-buildinfo=TEXT   Include some extra build information that will
                        be reported in the version command output

--without-python        Disable python bindings
--with-python=PATH      Enable Python bindings. PATH is location of python.
                        Default is to look for python in your PATH

--without-pcre       Don't enable pcre support.
--with-pcre=PATH     Enable pcre support.  PATH is location of pcre-config.
                     Default is to enable and look for pcre-config in your
                     $PATH

--with-folly=PATH  Specify root for Folly (needed for the C++ client library)
                   https://github.com/facebook/folly
```

(Run `./configure --help` to get the list for the version you checked out)

### Continuous Integration

We use continuous integration to build out every revision and pull-request to
make sure that we don't accidentally break things. The current build status
is:

[![Build Status](https://travis-ci.org/facebook/watchman.svg?branch=master)](https://travis-ci.org/facebook/watchman)

[![Build status](https://ci.appveyor.com/api/projects/status/uvafoyc550kg438h/branch/master?svg=true) ](https://ci.appveyor.com/project/wez/watchman/branch/master)

## System Specific Preparation

### Linux inotify Limits

The `inotify(7)` subsystem has three important tunings that impact watchman.

- `/proc/sys/fs/inotify/max_user_instances` impacts how many different root
  dirs you can watch.
- `/proc/sys/fs/inotify/max_user_watches` impacts how many dirs you can watch
  across all watched roots.
- `/proc/sys/fs/inotify/max_queued_events` impacts how likely it is that your
  system will experience a notification overflow.

You obviously need to ensure that `max_user_instances` and `max_user_watches`
are set so that the system is capable of keeping track of your files.

`max_queued_events` is important to size correctly; if it is too small, the
kernel will drop events and watchman won't be able to report on them. Making
this value bigger reduces the risk of this happening.

Watchman has two simple strategies for mitigating an overflow of
`max_queued_events`:

- It uses a dedicated thread to consume kernel events as quickly as possible
- When the kernel reports an overflow, watchman will assume that all the files
  have been modified and will re-crawl the directory tree as though it had
  just started watching the dir.

This means that if an overflow does occur, you won't miss a legitimate change
notification, but instead will get spurious notifications for files that
haven't actually changed.

### Mac OS File Descriptor Limits

_Only applicable on macOS 10.6 and earlier_

The default per-process descriptor limit on macOS is extremely low (256!).

Watchman will attempt to raise its descriptor limit to match
`kern.maxfilesperproc` when it starts up, so you shouldn't need to mess with
`ulimit`; just raising the sysctl should do the trick.

The following will raise the limits to allow 10 million files total, with 1
million files per process until your next reboot.

```bash
$ sudo sysctl -w kern.maxfiles=10485760
$ sudo sysctl -w kern.maxfilesperproc=1048576
```

Putting the following into a file named `/etc/sysctl.conf` on macOS will cause
these values to persist across reboots:

```
kern.maxfiles=10485760
kern.maxfilesperproc=1048576
```