.. _versioning:
.. highlight:: bash

Versioning
==========

This project uses bumpversion_ to manage the version strings related to
releases. The ``history.rst`` file should also be updated with the release
notes for each version.

To configure ``bumpversion`` edit the ``setup.cfg`` file. Running the
``bumpversion`` command will run a ``git commit`` and ``git tag`` by default.
Also by default, the version is updated in the project's ``__init__.py`` and
``history.rst`` files.

Before running ``bumpversion``, make sure all the changes are commited, and
run: ::

    $ bump2version [major|minor|patch]

Examples: ::

    $ cat setup.cfg | grep current_version
      current_version = 0.1.0

    $ bump2version patch

    $ cat setup.cfg | grep current_version
      current_version = 0.1.1

    $ bump2version minor

    $ cat setup.cfg | grep current_version
      current_version = 0.2.0

    $ bump2version minor

    $ cat setup.cfg | grep current_version
      current_version = 1.0.0

For more examples and configuration options see the bumpversion_ documentation.

.. _bumpversion: https://github.com/c4urself/bump2version/
