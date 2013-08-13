:title: PostgreSQL service How-To
:description: Running and installing a PostgreSQL service
:keywords: docker, example, package installation, postgresql

.. _postgresql_service:

PostgreSQL Service
==================

.. note::

    A shorter version of `this blog post`_.

.. note::

    As of version 0.5.2, docker requires root privileges to run.
    You have to either manually adjust your system configuration (permissions on
    /var/run/docker.sock or sudo config), or prefix `docker` with `sudo`. Check
    `this thread`_ for details.

.. _this blog post: http://zaiste.net/2013/08/docker_postgresql_how_to/
.. _this thread: https://groups.google.com/forum/?fromgroups#!topic/docker-club/P3xDLqmLp0E

Installing PostgreSQL on Docker
-------------------------------

For clarity I won't be showing commands output.


Run an interactive shell in Docker container.

.. code-block:: bash

    docker run -i -t ubuntu /bin/bash

Update its dependencies.

.. code-block:: bash

    apt-get update

Install ``python-software-properies``.

.. code-block:: bash

    apt-get install python-software-properties
    apt-get install software-properties-common

Add Pitti's PostgreSQL repository. It contains the most recent stable release
of PostgreSQL i.e. ``9.2``.

.. code-block:: bash

    add-apt-repository ppa:pitti/postgresql
    apt-get update

Finally, install PostgreSQL 9.2

.. code-block:: bash

    apt-get -y install postgresql-9.2 postgresql-client-9.2 postgresql-contrib-9.2

Now, create a PostgreSQL superuser role that can create databases and other roles.
Following Vagrant's convention the role will be named `docker` with `docker`
password assigned to it.

.. code-block:: bash

    sudo -u postgres createuser -P -d -r -s docker

Create a test database also named ``docker`` owned by previously created ``docker``
role.

.. code-block:: bash

    sudo -u postgres createdb -O docker docker

Adjust PostgreSQL configuration so that remote connections to the database are
possible. Make sure that inside ``/etc/postgresql/9.2/main/pg_hba.conf`` you have
following line:

.. code-block:: bash

    host    all             all             0.0.0.0/0               md5

Additionaly, inside ``/etc/postgresql/9.2/main/postgresql.conf`` uncomment
``listen_address`` so it is as follows:

.. code-block:: bash

    listen_address='*'

*Note:* this PostgreSQL setup is for development only purposes. Refer to
PostgreSQL documentation how to fine-tune these settings so that it is enough
secure.

Create an image and assign it a name. ``<container_id>`` is in the Bash prompt;
you can also locate it using ``docker ps -a``.

.. code-block:: bash

    docker commit <container_id> <your username>/postgresql

Finally, run PostgreSQL server via ``docker``.

.. code-block:: bash

    CONTAINER=$(docker run -d -p 5432 \
      -t <your username>/postgresql \
      /bin/su postgres -c '/usr/lib/postgresql/9.2/bin/postgres \
        -D /var/lib/postgresql/9.2/main \
        -c config_file=/etc/postgresql/9.2/main/postgresql.conf')

Connect the PostgreSQL server using ``psql``.

.. code-block:: bash

    CONTAINER_IP=$(docker inspect $CONTAINER | grep IPAddress | awk '{ print $2 }' | tr -d ',"')
    psql -h $CONTAINER_IP -p 5432 -d docker -U docker -W

As before, create roles or databases if needed.

.. code-block:: bash

    psql (9.2.4)
    Type "help" for help.

    docker=# CREATE DATABASE foo OWNER=docker;
    CREATE DATABASE

Additionally, publish there your newly created image on Docker Index.

.. code-block:: bash

    docker login
    Username: <your username>
    [...]

.. code-block:: bash

    docker push <your username>/postgresql

PostgreSQL service auto-launch
------------------------------

Running our image seems complicated. We have to specify the whole command with
``docker run``. Let's simplify it so the service starts automatically when the
container starts.

.. code-block:: bash

    docker commit <container_id> <your username>/postgresql -run='{"Cmd": \
      ["/bin/su", "postgres", "-c", "/usr/lib/postgresql/9.2/bin/postgres -D \
      /var/lib/postgresql/9.2/main -c \
      config_file=/etc/postgresql/9.2/main/postgresql.conf"], PortSpecs": ["5432"]}

From now on, just type ``docker run <your username>/postgresql`` and PostgreSQL
should automatically start.
