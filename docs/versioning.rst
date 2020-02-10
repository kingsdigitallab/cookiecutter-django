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

Git Commit Messages
----------------------

For the Git commit messages, it is recommend to use the Emoji-Log_ spec. Sample
``.gitconfig`` configuration::

  [alias]
    # https://opensource.com/article/19/2/emoji-log-git-commit-messages
    ac = "!f() { git add ${@:1:$(($# - 1))}; git commit -m \"${@:$#}\"; }; f"
    new = "!f() { git ac ${@:1:$(($# - 1))} \"📦 New: ${@:$#}\";  }; f"
    imp = "!f() { git ac ${@:1:$(($# - 1))} \"👌 Improve: ${@:$#}\";  }; f"
    fix = "!f() { git ac ${@:1:$(($# - 1))} \"🐛 Fix: ${@:$#}\";  }; f"
    rlz = "!f() { git ac ${@:1:$(($# - 1))} \"🚀 Release: ${@:$#}\";  }; f"
    doc = "!f() { git ac ${@:1:$(($# - 1))} \"📖 Doc: ${@:$#}\";  }; f"
    tst = "!f() { git ac ${@:1:$(($# - 1))} \"✅ Test: ${@:$#}\";  }; f"

.. _Emoji-Log: https://github.com/ahmadawais/Emoji-Log
