---
layout: post
title: "Python 3.4 from scratch, in an isolated environment"
date: 2014-03-31 04:48:35 -0400
comments: true
categories: [python]
---
Do you want to experiment with the new features of [Python 3.4](http://docs.python.org/dev/whatsnew/3.4.html) without installing it system-wide? There are multiple reasons you might want to do this:

- Your operating system doesn't have a supported package for it yet, and installing an experimental one might cause problems later
- You're concerned about accidentally running into Python version conflicts when you have multiple installed versions of Python
- You don't have root access, or you'd prefer not to use it
- You do everything in a virtualenv anyway, so what's the point of having another wrong place for packages to go?

Assuming you're on a reasonable Unix system (particularly Linux or Mac OS), you can accomplish this by building Python from the source code.

The new features of Python 3.4 make it very easy to install and start using it in an isolated way that never touches your `/usr` directory. Unlike Python 3.3 and earlier, you'll be able to quickly get started using `venv` and `pip`, and you won't be stuck in the purgatory of missing packages that `venv` would leave you in on 3.3.

I've tested these steps on Mac OS 10.9 and Ubuntu 13.10 Saucy Salamander.

## Step 1: C dependencies

This is a step where it is helpful to be root, if there are dependencies that you need but don't have yet. However, it's not strictly essential; you could install these dependencies under a custom prefix. To keep it simple, though, I'm going to assume you can install the dependencies system-wide.

Given that this is the CPython interpreter, you're going to need a C compiler (particularly `gcc`) to build it. On Mac OS, installing Xcode from the App Store gives everything that you need. On Ubuntu, you can get it with

```sh
sudo apt-get install build-essential
```

Python links to many external libraries to implement parts of its standard library, such as `sqlite3` and `readline`. I've found that if you're missing these libraries, it *will* compile, but of course you won't be able to use those libraries. The `ipython` experience in particular will be terrible without those libraries.

On my Mac, I found I had nearly everything that I needed already installed. This might be because I've already installed Python with Homebrew before, though. I do highly recommend Homebrew as a way of setting up development on a Mac. I was missing `ossaudiodev` support, which I don't plan to use anyway.

On Ubuntu, this command will install all of the library dependencies:

```sh
sudo apt-get install libc6-dev libreadline-dev libz-dev libncursesw5-dev \
     libssl-dev libgdbm-dev libsqlite3-dev libbz2-dev liblzma-dev tk-dev
```

## Step 2: Extract and compile Python

You're done with the hard part. The rest of the steps are things that should be as smooth as butter in Python 3.4.

Get the current source download from https://www.python.org/downloads/ . Right now, that'll give you a file called `Python-3.4.0.tar.xz`. Save it into a directory that you'd be happy running Python from, and change to that directory at the command line.

Then run:
```sh
tar xvf Python-3.4.0.tar.xz
cd Python-3.4.0
./configure && make
```

Sit back and relax for a few minutes.

At the end, if you were missing any optional libraries, Python will warn you about them. If anything shows up that you would sorely miss, go install the appropriate library, and `make` again.

## Step 3: Make your Python environment

We didn't run the traditional last step, `sudo make install`, because we don't need to! You've got everything you need to build a local Python environment right here, using Python 3's new `venv`.

```sh
mkdir -p ~/.virtualenvs
./python -m venv ~/.virtualenvs/py34
```

To activate this environment, now or in the future, run:

```
source ~/.virtualenvs/py34/bin/activate
```

You could put it in a different directory. I chose this location because it's compatible with Doug Hellmann's [virtualenvwrapper](http://virtualenvwrapper.readthedocs.org/en/latest/). If you have virtualenvwrapper installed, even from a previous version of Python, you can just run `workon py34` instead. It's totally fine with the fact that this environment was built with `venv`, not `virtualenv`.

**You are now using Python 3.4**. Type `python` and play around a bit.

But, of course, it's not really your Python environment until you've got packages installed. Fortunately, `pip` is already set up for this new environment!

Just to be sure, type `which pip`. It should show you a path in your `py34/bin` directory. If it says something like `/usr/bin/pip`, either you've forgotten to activate your environment or something has gone terribly wrong.

Now you can use this copy of `pip` to install your favorite packages:

```sh
pip install requests
```

Let's install IPython while we're at it:
```sh
pip install ipython
ipython3
```

IPython will install itself as `ipython3`, a compromise for the benefit of less-fortunate users who don't have virtualenvs set up. You could symlink `py34/bin/ipython` to `py34/bin/ipython3`, because you won't be needing ipython2 in this environment.


## Deactivating Python 3.4

When the time comes that you need to work on old code instead of living gloriously in the future, all you need to do is:

```sh
deactivate
```

The rest of your system is exactly as you left it until you activate the environment again.


## The not-so-isolated, really easy, bonus version

If you *do* have Python 3.4 installed system-wide -- maybe you're on Ubuntu 14.04 already -- then you can skip most of these steps and go straight to making the venv:

```sh
mkdir -p ~/.virtualenvs
python3.4 -m venv ~/.virtualenvs/py34
source ~/.virtualenvs/py34/bin/activate
pip install ipython
ipython3
```

