.. _upgrading:

Upgrading Citus
$$$$$$$$$$$$$$$

.. _upgrading_citus:

Upgrading Citus Versions
########################

Citus adheres to `semantic versioning <http://semver.org/>`_ with patch-, minor-, and major-versions. The upgrade process differs for each, requiring more effort for bigger version jumps.

Upgrading the Citus version requires first obtaining the new Citus extension and then installing it in each of your database instances. Citus uses separate packages for each minor version to ensure that running a default package upgrade will provide bug fixes but never break anything. Let's start by examining patch upgrades, the easiest kind.

Patch Version Upgrade
---------------------

To upgrade a Citus version to its latest patch, issue a standard upgrade command for your package manager. Assuming version 8.3 is currently installed on Postgres 11:

**Ubuntu or Debian**

.. code-block:: bash

  sudo apt-get update
  sudo apt-get install --only-upgrade postgresql-11-citus-8.3
  sudo service postgresql restart

**Fedora, CentOS, or Red Hat**

.. code-block:: bash

  sudo yum update citus82_11
  sudo service postgresql-11.0 restart

.. _major_minor_upgrade:

Major and Minor Version Upgrades
--------------------------------

Major and minor version upgrades follow the same steps, but be careful: major upgrades can make backward-incompatible changes in the Citus API. It is best to review the Citus `changelog <https://github.com/citusdata/citus/blob/master/CHANGELOG.md>`_ before a major upgrade and look for any changes which may cause problems for your application.

.. note::

   Starting at version 8.1, new Citus nodes expect and require encrypted inter-node communication by default, whereas nodes upgraded to 8.1 from an earlier version preserve their earlier SSL settings. Be careful when adding a new Citus 8.1 (or newer) node to an upgraded cluster that does not yet use SSL. The :ref:`adding a worker <adding_worker_node>` section covers that situation.

Each major and minor version of Citus is published as a package with a separate name. Installing a newer package will automatically remove the older version.

Step 1. Update Citus Package
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When doing a **major** version upgrade instead, be sure to upgrade the Citus extension first, and the PostgreSQL version second (see :ref:`upgrading_postgres`). Here is how to do a **minor** upgrade from 8.2 to 8.3:

**Ubuntu or Debian**

.. code-block:: bash

  sudo apt-get update
  sudo apt-get install postgresql-11-citus-8.3
  sudo service postgresql restart

**Fedora, CentOS, or Red Hat**

.. code-block:: bash

  # Fedora, CentOS, or Red Hat
  sudo yum swap citus82_11 citus83_11
  sudo service postgresql-11 restart

Step 2. Apply Update in DB
~~~~~~~~~~~~~~~~~~~~~~~~~~

After installing the new package and restarting the database, run the extension upgrade script.

.. code-block:: bash

  # you must restart PostgreSQL before running this
  psql -c 'ALTER EXTENSION citus UPDATE;'

  # you should see the newer Citus version in the list
  psql -c '\dx'


.. note::

  During a major version upgrade, from the moment of yum installing a new
  version, Citus will refuse to run distributed queries until the server is restarted and
  ALTER EXTENSION is executed. This is to protect your data, as Citus object and
  function definitions are specific to a version. After a yum install you
  should (a) restart and (b) run alter extension. In rare cases if you
  experience an error with upgrades, you can disable this check via the
  :ref:`citus.enable_version_checks <enable_version_checks>` configuration
  parameter. You can also `contact us <https://www.citusdata.com/about/contact_us>`_
  providing information about the error, so we can help debug the issue.

.. _upgrading_postgres:

Upgrading PostgreSQL version from 11 to 12
##########################################

.. note::

   Do not attempt to upgrade *both* Citus and Postgres versions at once. If both upgrades are desired, upgrade Citus first.

Record the following paths before you start (your actual paths may be different than those below):

Existing data directory (e.g. /opt/pgsql/10/data)
  :code:`export OLD_PG_DATA=/opt/pgsql/11/data`

Existing PostgreSQL installation path (e.g. /usr/pgsql-10)
  :code:`export OLD_PG_PATH=/usr/pgsql-11`

New data directory after upgrade
  :code:`export NEW_PG_DATA=/opt/pgsql/12/data`

New PostgreSQL installation path
  :code:`export NEW_PG_PATH=/usr/pgsql-12`

For Every Node
--------------

1. Back up Citus metadata in the old coordinator node.

  .. code-block:: postgres

    -- this step for the coordinator node only, not workers

    SELECT citus_prepare_pg_upgrade();

2. Configure the new database instance to use Citus.

  * Include Citus as a shared preload library in postgresql.conf:

    .. code-block:: ini

      shared_preload_libraries = 'citus'

  * **DO NOT CREATE** Citus extension

  * **DO NOT** start the new server

3. Stop the old server.

4. Check upgrade compatibility.

   .. code-block:: bash

     $NEW_PG_PATH/bin/pg_upgrade -b $OLD_PG_PATH/bin/ -B $NEW_PG_PATH/bin/ \
                                 -d $OLD_PG_DATA -D $NEW_PG_DATA --check

   You should see a "Clusters are compatible" message. If you do not, fix any errors before proceeding. Please ensure that

  * :code:`NEW_PG_DATA` contains an empty database initialized by new PostgreSQL version
  * The Citus extension **IS NOT** created

5. Perform the upgrade (like before but without the :code:`--check` option).

  .. code-block:: bash

    $NEW_PG_PATH/bin/pg_upgrade -b $OLD_PG_PATH/bin/ -B $NEW_PG_PATH/bin/ \
                                -d $OLD_PG_DATA -D $NEW_PG_DATA

6. Start the new server.

  * **DO NOT** run any query before running the queries given in the next step

7. Restore metadata on new coordinator node.

  .. code-block:: postgres

    -- this step for the coordinator node only, not workers

    SELECT citus_finish_pg_upgrade();
