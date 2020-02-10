.. _technical-overview:
.. highlight:: bash

Technical Overview
==================

Team
----

TODO

* Person name/role

Technologies and Processes
--------------------------

Development
^^^^^^^^^^^

For more information see `development`_ and `development with docker`_.

.. _development: https://cookiecutter-django-kingsdigitallab.readthedocs.io/en/latest/developing-locally.html
.. _development with docker: https://cookiecutter-django-kingsdigitallab.readthedocs.io/en/latest/developing-locally-docker.html

Data model
^^^^^^^^^^

TODO: Possible model types include glossary, conceptual model, encoding models.

To generate a graph of the Django models run the command
``compose/bin/manage.sh graph_models -o models.png`` if using Docker, if not
run the command ``python manage.py graph_models -o models.png``.

Workflows
^^^^^^^^^

TODO: data processing & editorial workflows

Architecture
^^^^^^^^^^^^
TODO: Extract high level description from PQ

To generate a graph of the Docker services/architecture run the command::

    docker run --rm -it --name dcv -v $(pwd):/input pmsipilot/docker-compose-viz render -m image local.yml

For more information see docker-compose-viz_.

.. _docker-compose-viz: https://github.com/ahmadawais/Emoji-Log

Design process
--------------

TODO: describe design process
