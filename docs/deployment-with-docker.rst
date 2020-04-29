Deployment with Docker
======================

.. index:: deployment, docker, docker-compose, compose


Prerequisites
-------------

* Docker 17.05+.
* Docker Compose 1.17+


Understanding the Docker Compose Setup
--------------------------------------

Before you begin, check out the ``production.yml`` file in the root of this
project. Keep note of how it provides configuration for the following services:

* ``django``: your application running behind ``Gunicorn``;
* ``postgres``: PostgreSQL database with the application's relational data;
* ``redis``: Redis instance for caching;
* ``traefik``: Traefik reverse proxy with HTTPS on by default.

Provided you have opted for Celery (via setting ``use_celery`` to ``y``) there
are three more services:

* ``celeryworker`` running a Celery worker process;
* ``celerybeat`` running a Celery beat process;
* ``flower`` running Flower_ (for more info, check out :ref:`CeleryFlower` instructions for local environment).

.. _`Flower`: https://github.com/mher/flower


Configuring the Stack
---------------------

The majority of services above are configured through the use of environment
variables. Just check out :ref:`envs` and you will know the drill.

To obtain logs and information about crashes in a production setup, make sure
that you have access to an external Sentry instance (e.g. by creating an
account with `sentry.io`_), and set the ``SENTRY_DSN`` variable. Logs of level
`logging.ERROR` are sent as Sentry events. Therefore, in order to send a Sentry
event use:

.. code-block:: python

    import logging
    logging.error("This event is sent to Sentry", extra={"<example_key>": "<example_value>"})

The `extra` parameter allows you to send additional information about the
context of this error.


You will probably also need to setup the Mail backend, for example by adding a
`Mailgun`_ API key and a `Mailgun`_ sender domain, otherwise, the account
creation view will crash and result in a 500 error when the backend attempts to
send an email to the account owner.

.. _sentry.io: https://sentry.io/welcome
.. _Mailgun: https://mailgun.com


.. warning::

    .. include:: mailgun.rst


Optional: Use AWS IAM Role for EC2 instance
-------------------------------------------

If you are deploying to AWS, you can use the IAM role to substitute AWS
credentials, after which it's safe to remove the ``AWS_ACCESS_KEY_ID`` and
``AWS_SECRET_ACCESS_KEY`` from ``.envs/.production/.django``. To do it, create
an `IAM role`_ and `attach`_ it to the existing EC2 instance or create a new
EC2 instance with that role. The role should assume, at minimum, the
``AmazonS3FullAccess`` permission.

.. _IAM role: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html
.. _attach: https://aws.amazon.com/blogs/security/easily-replace-or-attach-an-iam-role-to-an-existing-ec2-instance-by-using-the-ec2-console/


HTTPS is On by Default
----------------------

SSL (Secure Sockets Layer) is a standard security technology for establishing
an encrypted link between a server and a client, typically in this case, a web
server (website) and a browser. Not having HTTPS means that malicious network
users can sniff authentication credentials between your website and end users'
browser.

It is always better to deploy a site behind HTTPS and will become crucial as
the web services extend to the IoT (Internet of Things). For this reason, we
have set up a number of security defaults to help make your website secure:

* If you are not using a subdomain of the domain name set in the project, then remember to put your staging/production IP address in the ``DJANGO_ALLOWED_HOSTS`` environment variable (see :ref:`settings`) before you deploy your website. Failure to do this will mean you will not have access to your website through the HTTP protocol.

* Access to the Django admin is set up by default to require HTTPS in production or once *live*.

The Traefik reverse proxy used in the default configuration will get you a
valid certificate from Lets Encrypt and update it automatically. All you need
to do to enable this is to make sure that your DNS records are pointing to the
server Traefik runs on.

You can read more about this feature and how to configure it, at
`Automatic HTTPS`_ in the Traefik docs.

.. _Automatic HTTPS: https://docs.traefik.io/configuration/acme/


(Optional) Postgres Data Volume Modifications
---------------------------------------------

Postgres is saving its database files to the ``production_postgres_data``
volume by default. Change that if you want something else and make sure to make
backups since this is not done automatically.


Building & Running Production Stack
-----------------------------------

You will need to build the stack first. To do that, run::

    docker-compose -f production.yml build

Once this is ready, you can run it with::

    docker-compose -f production.yml up

To run the stack and detach the containers, run::

    docker-compose -f production.yml up -d

To run a migration, open up a second terminal and run::

   docker-compose -f production.yml run --rm django python manage.py migrate

To create a superuser, run::

   docker-compose -f production.yml run --rm django python manage.py createsuperuser

If you need a shell, run::

   docker-compose -f production.yml run --rm django python manage.py shell

To check the logs out, run::

   docker-compose -f production.yml logs

If you want to scale your application, run::

   docker-compose -f production.yml scale django=4
   docker-compose -f production.yml scale celeryworker=2

.. warning:: don't try to scale ``postgres``, ``celerybeat``, or ``traefik``.

To see how your containers are doing run::

    docker-compose -f production.yml ps


Example: Supervisor
-------------------

Once you are ready with your initial setup, you want to make sure that your
application is run by a process manager to survive reboots and auto restarts
in case of an error. You can use the process manager you are most familiar
with. All it needs to do is to run ``docker-compose -f production.yml up`` in
your projects root directory.

If you are using ``supervisor``, you can use this file as a starting point::

    [program:{{cookiecutter.project_slug}}]
    command=docker-compose -f production.yml up
    directory=/path/to/{{cookiecutter.project_slug}}
    redirect_stderr=true
    autostart=true
    autorestart=true
    priority=10

Move it to ``/etc/supervisor/conf.d/{{cookiecutter.project_slug}}.conf`` and
run::

    supervisorctl reread
    supervisorctl update
    supervisorctl start {{cookiecutter.project_slug}}

For status check, run::

    supervisorctl status


Docker Security
---------------

This section contains a list of security issues identified by the
`Docker Bench for Security`_ tool, after a deployment in an `Ubuntu 16.04`
machine using the instructions in `Building & Running Production Stack`_,
and possible fixes.

.. warning::

    After applying some of the fixes you might need to rebuild the stack,
    otherwise the issues might still be reported when re-running
    `Docker Bench for Security`_.

Issues
~~~~~~

The numbers in the headings correspond to the `Docker Bench for Security`_ test
number.

1.2.1 - Ensure a separate partition for containers has been created
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^


1.2.3 - Ensure auditing is configured for the Docker daemon and files/directories
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Install auditd_::

    $ sudo apt-get install auditd

Edit the auditing system rules::

    $ sudo vim /etc/audit/audit.rules

These rules instruct auditd to watch (-w) the specified file or directory and
log any writes or attribute changes (-p wa) to those files::

    -w /etc/default/docker -p wa
    -w /etc/docker -p wa
    -w /etc/docker/daemon.json -p wa
    -w /etc/sysconfig/docker -p wa
    -w /lib/systemd/system/docker.service -p wa
    -w /lib/systemd/system/docker.socket -p wa
    -w /project/docker -p wa
    -w /usr/bin/docker -p wa
    -w /usr/bin/containerd -p wa
    -w /usr/bin/runc -p wa
    -w /var/lib/docker -p wa

Restart autditd::

    $ sudo systemctl restart auditd

2 - Docker daemon configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: JSON
    :caption: /etc/docker/daemon.json
    :name: daemon.json

    {
        "data-root": "/project/docker",
        "icc": false,
        "live-restore": true,
        "log-driver": "syslog",
        "no-new-privileges": true,
        "userland-proxy": false,
        "userns-remap": "default"
    }

For more information on how to configure the Docker daemon see the official
`Docker daemon`_ documentation. Below is a short explanation for each of the
configuration options in ``daemon.json``.

``data_root``
    Root directory of persistent Docker state (default "/var/lib/docker")
``icc``
    2.1 - Ensure network traffic is restricted between containers on the
    default bridge
``live-restore``
    2.13 - Ensure live restore is Enabled
``log-driver``
    2.12 - Ensure centralized and remote logging is configured
``no-new-privileges``
    2.18 - Ensure containers are restricted from acquiring new privileges
``userland-proxy``
    2.15 - Ensure Userland Proxy is Disabled
``userns-remap``
    2.8 - Enable user namespace support

4.5 - Ensure Content trust for Docker is Enabled
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To enable content trust for all users and sessions::

    $ echo "DOCKER_CONTENT_TRUST=1" | sudo tee -a /etc/environment

For more information see the `Docker content trust`_ documentation.

4.6 - Ensure that HEALTHCHECK instructions have been added to container images
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
This should also cover the issue with
`5.26 - Ensure that container health is checked at runtime`.

For example, to check every five minutes or so that a web-server is able to
serve the site’s main page within three seconds::

    HEALTHCHECK --interval=5m --timeout=3s \
    CMD curl -f http://localhost/ || exit 1

5.2 - Ensure that, if applicable, SELinux security options are set
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

5.7 - Ensure privileged ports are not mapped within containers
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Mapping http port 80 and https port 443 is necessary for traefik/webserver. All
the other ports in the stack are not privileged ports.

5.10 - Ensure that the memory usage for containers is limited
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

`Runtime options with Memory`_.

5.11 - Ensure CPU priority is set appropriately on the container
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

`Runtime options with CPUs`_.

5.12 - Ensure that the container's root filesystem is mounted as read only
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Mount host-sensitive directories as read-only. In the default cookiecutter
configuration no host-sensitive directories are shared with the containers.

5.13 - Ensure that incoming container traffic is bound to a specific host interface
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

5.14 - Ensure that the 'on-failure' container restart policy is set to '5'
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

`Restart policy`_.

5.25 - Ensure that the container is restricted from acquiring additional privileges
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Set in :ref:`daemon.json`.

5.27 - Ensure that Docker commands always make use of the latest version of their image
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

5.28 - Ensure that the PIDs cgroup limit is used
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Useful Resources
^^^^^^^^^^^^^^^^

- `Top 20 Docker Security Tips`_
- `10 Docker Image Security Best Practices`_
- `10+ top open-source tools for Docker security`_
- `How To Audit Docker Host Security with Docker Bench for Security on Ubuntu 16.04`_
- `Securing Docker Containers on AWS`_
- `Hardening Docker containers, images, and host - security toolkit`_
- `Building Docker Images using Docker Compose and Gitlab CI/CD`_

.. _Docker Bench for Security: https://github.com/docker/docker-bench-security
.. _auditd: https://linux.die.net/man/8/auditd
.. _Docker daemon: https://docs.docker.com/config/daemon/
.. _Docker content trust: https://docs.docker.com/engine/security/trust/content_trust/
.. _Runtime options with Memory: https://docs.docker.com/config/containers/resource_constraints/#memory
.. _Runtime options with CPUs: https://docs.docker.com/config/containers/resource_constraints/#cpu
.. _Restart policy: https://docs.docker.com/compose/compose-file/#restart_policy
.. _Top 20 Docker Security Tips: https://towardsdatascience.com/top-20-docker-security-tips-81c41dd06f57
.. _10 Docker Image Security Best Practices: https://snyk.io/blog/10-docker-image-security-best-practices/
.. _10+ top open-source tools for Docker security: https://techbeacon.com/security/10-top-open-source-tools-docker-security
.. _How To Audit Docker Host Security with Docker Bench for Security on Ubuntu 16.04: https://www.digitalocean.com/community/tutorials/how-to-audit-docker-host-security-with-docker-bench-for-security-on-ubuntu-16-04
.. _Securing Docker Containers on AWS: https://www.nearform.com/blog/securing-docker-containers-on-aws/
.. _Hardening Docker containers, images, and host - security toolkit: https://www.stackrox.com/post/2017/08/hardening-docker-containers-and-hosts-against-vulnerabilities-a-security-toolkit/
.. _Building Docker Images using Docker Compose and Gitlab CI/CD: https://vgarcia.dev/blog/2019-06-17-building-docker-images-using-docker-compose-and-gitlab/
