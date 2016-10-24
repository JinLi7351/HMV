..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode


===================================================
Heat API microversions support in python-heatclient
===================================================

Blueprint: microversions

This spec proposes to add microversions support to Heat API and to call
out the specific behavior between heat and the heat client.

Problem description
===================

The framework of API Microversions allows user to explicitly ask a service
to treat a request with a specified API version.  This functionality allows
backward incompatible API changes without breaking users who do not ask for
the changes.  Backward compatibility of OpenStack APIs is defined by the
guidelines outlined here:
http://specs.openstack.org/openstack/api-wg/guidelines/evaluating_api_changes.html.

This can be achieved by passing in an extra HTTP header,
``X-OpenStack-API-Version``, which is a strictly monotone sequence of
version numbers.  If a version number is not specified with the request,
a default will be defined in ``heat/common/wsgi.py``.


Use Cases
---------

A poc has been created for Heat server to demostrate microversion in the server side.
https://gitlab.com/syjulian/poc-heat-mversion
The poc adds a new REST api named random_facts whose controller defines multiple 
implementations for a resource. To call a specific version of random_facts, a header 
attribute named Openstack-API-Version need to be added. If this header attribute is not
specified in the http request for random_facts,Heat will call the major API version 
without microversion by default. If the OpenStack-API-Version attribute specifies a 
version that is not supported by Heat, ``406 Not acceptable`` should be returned.

when the python-heatclient calls the Heat server, it will need to provide the 
OpenStack-API-Version header attribute. If Heatclient does not provide this header
attribute, Heat server will call the latest major version by default. Or if Heatclient
provides version that is not supported by Heat server, Heat server will return 
``406 Not acceptable``

.. code-block:: python

	class RandomFactsController(versioned_object.VersionedObject):
		def __init__(self, options):
			self.options = options

		@versioned_object.VersionedObject.api_version("1.0")
		def random_facts(self, req):
			fact = {'fact': ['Microversioning may or may not be working...']}
			return fact

		@versioned_object.VersionedObject.api_version("1.1")  # noqa
		def random_facts(self, req):
			fact = {'fact': ['Microversioning is sort of working...']}
			return fact

		@versioned_object.VersionedObject.api_version("1.2", "1.4")
		def facts(self, req):
			fact = {'fact': ['Racecar is racecar backwards...']}
			return fact

		@versioned_object.VersionedObject.api_version("1.5")  # noqa
		def facts(self, req):
			fact = {'fact': ['QA stands for quality assurance...']}
			return fact

example for requesting a specific version of random_facts in devstack

.. code-block:: python
	
	curl -H "X-Auth-Token: $keystone-Token"\
	-H "Openstack-API-Version: orchestration 1.0"\
	http://10.0.2.15:8004/v1/random_facts
	



Proposed change
===============

Microversions are implemented by supporting the standard OpenStack version
HTTP header ``X-OpenStack-API-Version`` with a service type of `heat`.  For
example:

.. code-block:: bash

   'X-OpenStack-API-Version': heat 1.4

This approach is outlined at
https://specs.openstack.org/openstack/api-wg/guidelines/microversion_specification.html.


Specific changes
----------------

Versioning
~~~~~~~~~~

The heat orchestration API should be extended to include a major and minor
parts of version.  It should be one of the following:

* ``X.Y``: where ``X`` is the major version, and ``Y`` is the minor version.
  Both should accept numeric values.
* ``X.latest``: where ``X`` is the major version and accepts numeric values.
  The client uses the latest supported microversion by the server and client.
* ``latest``: the client uses the latest major version known and the latest
  supported microversion by the server and client.

If the client chose to include the ``X-OpenStack-API-Version`` header to
request specific version, heat would response either by accepting the
request or returning ``406 Not acceptable`` if the version is invalid.

If the client chose to not use the header ``X-OpenStack-API-Version``,
heat would assume the API is using version << fill in blank >>, the latest
version before minimum version that supports microversions.

python-heatclient as CLI tool
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Microversions should be specified with a major API version, using a new client
option ``--os-heat-api-version``.  A user can also use
``--os-heat-api-version="None"`` to indicate the client should use the default
major API version without microversion.

Help messages should be displayed for all commands, sub-commands and related
options information about supported versions.

python-healtclient as a python lib
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Module ``heatclient.client`` is used as entry point to python-heatclient inside
other python libraries.  The interface of this module should not be changed
to support backward compatibility.

``heatclient.client.Client`` function should now accept a string value or
an instance of an ``APIVersion`` object as the first argument.  In the former
case, the format of the version should be checked.

If a microversion is specified, the client should add HTTP header
``X-OpenStack-API-Version`` to each call and validate that the response
header includes the same header, meaning the API side also supports
microversions.

"latest" microversion
~~~~~~~~~~~~~~~~~~~~~

The ``latest`` microversion is the maximum version.  Despite the fact heat-api
accepts the value of ``latest``, the client does not.  The client discovers
the ``latest`` microversion supported by both the API and the client.
The client should cache all discovered versions to minimize the number of
calls.

``latest`` microversion discovery should proceed as follows:

* The client makes a call to heat api listing all versions
* The client determines the current version by comparing the API response and
  the endpoint URL
* The client checks the current version supports microversions by checking
  the values of ``min_version`` and the current ``version``.  If the current
  version does not support microversion, the client uses the default version.
* The client chooses the latest microversion supported by both the heatclient
  and the heat-api server.

Developer impacts from python-heatclient
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Each "versioned" method of ``ResourceManager`` should be labeled with specific
decorator.  Each decorator should accept a ``start_version`` and an optional
``end_version`` parameter.  For example:

.. code-block:: python

   class ResourceManager(stacks.StackChildManager):

      @api_version(start_version='1.0', end_version='2.4')
      def show(self, stack_id, **kwargs):
         pass

      @api_versions.wrap(start_version='1.1')
      def show(self, stack_id, **kwargs):
         pass

"Versioned" arguments should be used as follow:

.. code-block:: python

   from heatclient.common import utils

   @utils.arg('-o',
              '--template-object',
              metavar='<URL>',
              help=_('URL to retrieve template object (e.g. from swift).'),
              start_version="1.4")

This will also need to be applied to the OpenStackClient.

Alternatives
------------

One alternative is to make only additive changes, and only make backward
incompatible changes at major API version release, e.g. going from v2 to v3.
A major drawback of this approach is the long release cycle of major version.
The need to support multiple major versions results in maintaince overhead.

Implementation
==============

Assignee(s)
-----------

Primary assignee(s):

* Julian Sy -
* Jin Li -


Milestones
----------

Target Milestone for completion:
  ocata-3

Work Items
----------

* Implement handling of ``X-OpenStack-API-Version`` header in
  python-heatclient.
* Implement unit tests.
* Update all appropriate documentation and CLI commands and
  sub-commands help text.

Dependencies
============

* Heat API Microversions BP: << Need URL for this BP >>

References
==========

* Nova Microversion BP: http://specs.openstack.org/openstack/nova-specs/specs/kilo/implemented/api-microversions.html
