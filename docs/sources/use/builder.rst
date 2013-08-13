:title: Dockerfiles for Images
:description: Dockerfiles use a simple DSL which allows you to automate the steps you would normally manually take to create an image.
:keywords: builder, docker, Dockerfile, automation, image creation

.. _dockerbuilder:

==================
Dockerfile Builder
==================

**Docker can act as a builder** and read instructions from a text
Dockerfile to automate the steps you would otherwise make manually to
create an image. Executing ``docker build`` will run your steps and
commit them along the way, giving you a final image.

.. contents:: Table of Contents

1. Usage
========

To build an image from a source repository, create a description file
called ``Dockerfile`` at the root of your repository. This file will
describe the steps to assemble the image.

Then call ``docker build`` with the path of your source repository as
argument:

    ``docker build .``

You can specify a repository and tag at which to save the new image if the
build succeeds:

    ``docker build -t shykes/myapp .``

Docker will run your steps one-by-one, committing the result if necessary,
before finally outputting the ID of your new image.

2. Format
=========

The Dockerfile format is quite simple:

::

    # Comment
    INSTRUCTION arguments

The Instruction is not case-sensitive, however convention is for them to be
UPPERCASE in order to distinguish them from arguments more easily.

Docker evaluates the instructions in a Dockerfile in order. **The first
instruction must be `FROM`** in order to specify the base image from
which you are building.

Docker will ignore **comment lines** *beginning* with ``#``. A comment
marker anywhere in the rest of the line will be treated as an argument.

3. Instructions
===============

Here is the set of instructions you can use in a ``Dockerfile`` for
building images.

3.1 FROM
--------

    ``FROM <image>``

The ``FROM`` instruction sets the :ref:`base_image_def` for subsequent
instructions. As such, a valid Dockerfile must have ``FROM`` as its
first instruction.

``FROM`` must be the first non-comment instruction in the
``Dockerfile``.

``FROM`` can appear multiple times within a single Dockerfile in order
to create multiple images. Simply make a note of the last image id
output by the commit before each new ``FROM`` command.

3.2 MAINTAINER
--------------

    ``MAINTAINER <name>``

The ``MAINTAINER`` instruction allows you to set the *Author* field of
the generated images.

3.3 RUN
-------

    ``RUN <command>``

The ``RUN`` instruction will execute any commands on the current image
and commit the results. The resulting committed image will be used for
the next step in the Dockerfile.

Layering ``RUN`` instructions and generating commits conforms to the
core concepts of Docker where commits are cheap and containers can be
created from any point in an image's history, much like source
control.

3.4 CMD
-------

    ``CMD <command>``

The ``CMD`` instruction sets the command to be executed when running
the image.  This is functionally equivalent to running ``docker commit
-run '{"Cmd": <command>}'`` outside the builder.

.. note::
    Don't confuse ``RUN`` with ``CMD``. ``RUN`` actually runs a
    command and commits the result; ``CMD`` does not execute anything at
    build time, but specifies the intended command for the image.

3.5 EXPOSE
----------

    ``EXPOSE <port> [<port>...]``

The ``EXPOSE`` instruction sets ports to be publicly exposed when
running the image. This is functionally equivalent to running ``docker
commit -run '{"PortSpecs": ["<port>", "<port2>"]}'`` outside the
builder.

3.6 ENV
-------

    ``ENV <key> <value>``

The ``ENV`` instruction sets the environment variable ``<key>`` to the
value ``<value>``. This value will be passed to all future ``RUN``
instructions. This is functionally equivalent to prefixing the command
with ``<key>=<value>``

.. note::
    The environment variables will persist when a container is run
    from the resulting image.

3.7 ADD
-------

    ``ADD <src> <dest>``

The ``ADD`` instruction will copy new files from <src> and add them to
the container's filesystem at path ``<dest>``.

``<src>`` must be the path to a file or directory relative to the
source directory being built (also called the *context* of the build) or
a remote file URL.

``<dest>`` is the path at which the source will be copied in the
destination container.

The copy obeys the following rules:

* If ``<src>`` is a URL and ``<dest>`` does not end with a trailing slash,
  then a file is downloaded from the URL and copied to ``<dest>``.
* If ``<src>`` is a URL and ``<dest>`` does end with a trailing slash,
  then the filename is inferred from the URL and the file is downloaded to
  ``<dest>/<filename>``. For instance, ``ADD http://example.com/foobar /``
  would create the file ``/foobar``. The URL must have a nontrivial path
  so that an appropriate filename can be discovered in this case
  (``http://example.com`` will not work).
* If ``<src>`` is a directory, the entire directory is copied,
  including filesystem metadata.
* If ``<src>``` is a tar archive in a recognized compression format
  (identity, gzip, bzip2 or xz), it is unpacked as a directory.

  When a directory is copied or unpacked, it has the same behavior as
  ``tar -x``: the result is the union of

  1. whatever existed at the destination path and
  2. the contents of the source tree,

  with conflicts resolved in favor of 2) on a file-by-file basis.

* If ``<src>`` is any other kind of file, it is copied individually
  along with its metadata. In this case, if ``<dst>`` ends with a
  trailing slash ``/``, it will be considered a directory and the
  contents of ``<src>`` will be written at ``<dst>/base(<src>)``.
* If ``<dst>`` does not end with a trailing slash, it will be
  considered a regular file and the contents of ``<src>`` will be
  written at ``<dst>``.
* If ``<dest>`` doesn't exist, it is created along with all missing
  directories in its path. All new files and directories are created
  with mode 0755, uid and gid 0.

3.8 ENTRYPOINT
--------------

    ``ENTRYPOINT ["/bin/echo"]``

The ``ENTRYPOINT`` instruction adds an entry command that will not be
overwritten when arguments are passed to docker run, unlike the
behavior of ``CMD``.  This allows arguments to be passed to the
entrypoint.  i.e. ``docker run <image> -d`` will pass the "-d" argument
to the entrypoint.

3.9 VOLUME
----------

    ``VOLUME ["/data"]``

The ``VOLUME`` instruction will add one or more new volumes to any
container created from the image.

4. Dockerfile Examples
======================

.. code-block:: bash

    # Nginx
    #
    # VERSION               0.0.1

    FROM      ubuntu
    MAINTAINER Guillaume J. Charmes "guillaume@dotcloud.com"

    # make sure the package repository is up to date
    RUN echo "deb http://archive.ubuntu.com/ubuntu precise main universe" > /etc/apt/sources.list
    RUN apt-get update

    RUN apt-get install -y inotify-tools nginx apache2 openssh-server

.. code-block:: bash

    # Firefox over VNC
    #
    # VERSION               0.3

    FROM ubuntu
    # make sure the package repository is up to date
    RUN echo "deb http://archive.ubuntu.com/ubuntu precise main universe" > /etc/apt/sources.list
    RUN apt-get update

    # Install vnc, xvfb in order to create a 'fake' display and firefox
    RUN apt-get install -y x11vnc xvfb firefox
    RUN mkdir /.vnc
    # Setup a password
    RUN x11vnc -storepasswd 1234 ~/.vnc/passwd
    # Autostart firefox (might not be the best way, but it does the trick)
    RUN bash -c 'echo "firefox" >> /.bashrc'

    EXPOSE 5900
    CMD    ["x11vnc", "-forever", "-usepw", "-create"]

.. code-block:: bash

    # Multiple images example
    #
    # VERSION               0.1

    FROM ubuntu
    RUN echo foo > bar
    # Will output something like ===> 907ad6c2736f

    FROM ubuntu
    RUN echo moo > oink
    # Will output something like ===> 695d7793cbe4

    # You'll now have two images, 907ad6c2736f with /bar, and 695d7793cbe4 with
    # /oink.
