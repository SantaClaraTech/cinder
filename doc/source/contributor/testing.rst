.. _testing-cinder:

Testing
=======

Cinder contains a few different test suites in the cinder/tests/ directory. The
different test suites are Unit Tests, Functional Tests, and Tempest Tests.

Test Types
----------


Unit Tests
~~~~~~~~~~

Unit tests are tests for individual methods, with at most a small handful of
modules involved. Mock should be used to remove any external dependencies.

All significant code changes should have unit test coverage validating the code
happy path and any failure paths.

There's a tox environment defined that will run code coverage tests for you::

  tox -e cover

It will create an HTML code coverage report that you can use a web browser
to read locally from the location ``./cover/index.html`` (relative to the
location of your ``tox.ini`` file).  If you are reviewing someone else's
patch in Gerrit, we have ``cinder-code-coverage`` job that generates a
coverage report that you can read.  From the review page, follow:
"Zuul Summary" tab -> "cinder-code-coverage" link -> "Logs" tab.  The
"raw" link next to "cover" will take you to the index page of the report.

Any proposed code change will be automatically rejected by the OpenDev
Zuul project gating system [#f1]_ if the change causes unit test failures.

Functional Tests
~~~~~~~~~~~~~~~~

Functional tests validate a code path within Cinder. These tests should
validate the interaction of various modules within the project to verify the
code is logically correct.

Functional tests run with a database present and may start Cinder services to
accept requests. These tests should not need to access an other OpenStack
non-Cinder services.

Tempest Tests
~~~~~~~~~~~~~

The tempest tests in the Cinder tree validate the operational correctness
between Cinder and external components such as Nova, Glance, etc. These are
integration tests driven via public APIs to verify actual end user usage
scenarios.

Running the tests
-----------------

There are a number of ways to run tests currently, and there's a combination of
frameworks used depending on what commands you use. The preferred method is to
use tox, which calls ostestr via the tox.ini file.

Unit Tests
~~~~~~~~~~

To run all unit tests simply run::

    tox

This will create a virtual environment, load all the packages from
test-requirements.txt and run all unit tests as well as run flake8 and hacking
checks against the code.

You may run individual test targets, for example only py37 tests, by running::

    tox -e py37

Note that you can inspect the tox.ini file to get more details on the available
options and what the test run does by default.

Functional Tests
~~~~~~~~~~~~~~~~

To run all functional tests, run::

    tox -e functional

Tempest Tests
~~~~~~~~~~~~~

Tempest tests in the Cinder tree are "plugged in" to the normal tempest test
execution. To ensure the Cinder tests are picked up when running tempest, run::

    cd /opt/stack/tempest
    tox -e all-plugin

More information about tempest can be found in the `Tempest Documentation
<https://docs.openstack.org/tempest/latest/>`_.

Database Setup
~~~~~~~~~~~~~~~

Some unit and functional tests will use a local database. You can use
``tools/test-setup.sh`` to set up your local system the same way as
it's setup in the CI environment.

Running a subset of tests using tox
-----------------------------------
One common activity is to just run a single test, you can do this with tox
simply by specifying to just run py37 tests against a single test::

    tox -epy37 -- cinder.tests.unit.volume.test_availability_zone.AvailabilityZoneTestCase.test_list_availability_zones_cached

Or all tests in the test_volume.py file::

    tox -epy37 -- cinder.tests.unit.volume.test_volume

You may also use regular expressions to run any matching tests::

    tox -epy37 -- test_volume

For more information on these options and details about stestr, please see the
`stestr documentation <http://stestr.readthedocs.io/en/latest/MANUAL.html>`_.

Gotchas
-------

**Running Tests from Shared Folders**

If you are running the unit tests from a shared folder, you may see tests start
to fail or stop completely as a result of Python lockfile issues. You
can get around this by manually setting or updating the following line in
``cinder/tests/conf_fixture.py``::

    CONF['lock_path'].SetDefault('/tmp')

Note that you may use any location (not just ``/tmp``!) as long as it is not
a shared folder.

**Assertion types in unit tests**

In general, it is best to use the most specific assertion possible in a unit
test, to have the strongest validation of code behavior.

For example:

.. code-block:: python

    self.assertEqual("in-use", volume.status)

is preferred over

.. code-block:: python

    self.assertIsNotNone(volume.status)

or

Test methods that implement comparison checks are also generally preferred
over writing code into assertEqual() or assertTrue().

.. code-block:: python

   self.assertGreater(2, volume.size)

is preferred over

.. code-block:: python

   self.assertTrue(2 > volume.size)

However, assertFalse() behavior is not obvious in this regard.  Since
``None`` evaluates to ``False`` in Python, the following check will pass when
x is ``False`` or ``None``.

.. code-block:: python

   self.assertFalse(x)

Therefore, it is preferable to use:

.. code-block:: python

   self.assertEqual(x, False)


.. rubric:: Footnotes

.. [#f1] See :doc:`zuul`.


Debugging
---------

**Debugging unit tests**

It is possible to attach a debugger to unit tests.

First, modify the test you want to debug by adding the
following to the test code itself:

.. code-block:: python

   import pdb
   pdb.set_trace()

Then run the unit test with pdb enabled:

.. code-block:: bash

   source .tox/py36/bin/activate

   stestr run -n cinder.tests.unit.test_volume_utils

   # Or to get a list of tests to run

   stestr list test_volume_utils > tests_to_run.txt
   stestr run --load-list tests_to_run.txt
