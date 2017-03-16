..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============
Ensure share
============

https://blueprints.launchpad.net/manila/+spec/ensure-share

It's not reasonable to update all shares on every share manager when we
restart manila share service. This driver interface is currently being used
wrong and needs a rewrite. This spec adds the ability to solve the potential
problem of slow start up, and deal with non-user-initiated state changes to
shares.

Problem description
===================

Manila-share service has share driver interface called "ensure_share",
that, depending on share driver, performs some actions on existing shares.
This method is called on each manila-share service start and none of RPC
requests are handled while this method does not finish processing existing
shares. It makes life of cloud administrator painful, because each
manila-share service restart takes more and more time with growth of shares
amount. Also, in most cases, this processing is redundant.

Use cases
=========

Consider the following reasonable use cases:

* Manila-share node going to be shut down for maintenance and then enabled
  back when we want to upgrade software or something else. In this case
  admin expects fast start of service, no matter how many shares do exist
  and managed by manila, and no matter they change any config options or not.

* Allows admins to update shares automatically when the admin restarts the
  service and they changed some config options or updates storage somehow
  (such as: we have an IPv4-only controller and we want to add an IPv6 address
  (any export location)).

In the future we will solve following use cases:

* Allows admins to automatically tag some shares or shares belonging to
  particular backends as needing 'ensure'.

* Allows admins to update other resouces(such as: snapshot, share group,
  etc) for share services when the admin restarts the service.

* Allows admins to update shares and don't need to restart manila share
  services when the admin changes something on array (such as: export
  location IP). (depends on other spec)

Proposed change
===============

To accomplish this change, we will do the following:

When the share manager starts up, it will compute a hash of several values and
compare the hash to the previously computed value. If the value hasn't changed
then the ensure_share logic will be skipped, otherwise the ensure_share logic
will execute as normal but the new hash value will be stored in the DB.

The values computed in the hash will include:

1. The ID of the most recent DB migration. Including this value guarantees
   that ensure_share will be called after DB schema changes.
2. All of the keys and values in the oslo config object, sorted
   lexicographically by key. Including these values guarantees that
   ensure_share is called whenever the config file changes in a way that
   impacts the driver.
3. All of the keys and values returned by the get_backend_config() driver
   method. This allows drivers to contact the actual storage controller and
   detect any important configuration values which might have changed.

The get_backend_config() driver method is a new driver method which will be
called after driver initialization but before ensure_share. It allows
drivers to return a dictionary of values containing either hardcoded values or
value retrieved from the storage controller (or some combination).

The reason for hashing the values is because Manila doesn't care what the old
values were, only that something changed. Storing a hash of the values is
simpler from a DB schema perspective. In order to avoid accidental hash
collisions SHA1 will be used to generate 160 bits worth of data. Python
objects in dictionaries will be converted to strings, then UTF-8 encoded into
bytes, and fed into the hash algorithm.



There are workflow examples:

1. An existing share with a set of characteristics, and then some change in
   config or the array.
2. When we restart manila share service, we will get driver info from DB, then
   the ensure_check_update method should be invoked.
3. Driver will check whether the driver info has been changed, and
   whether driver needs to update shares.
4. We will store the new driver info in DB.
5. If the check_needs_update method returns true, the ensure_shares method
   should be invoked. We will update share information after we receive the
   value of the driver ensure_shares method.

- We'd need to store driver information(such as: export location ip)
  in the database(Add the ``manila.db.sqlalchemy.models.DriverConfigInfo``
  model).
  Only run ensure share if the driver says yes. It means we will query the
  storage controller once at startup to determine if anything has changed.

- Add the driver's ``get_driver_info`` method, it should read the config
  options (so changes in manila.conf are read through this) and also array
  information.
- Add driver interface to check if driver needs to update shares::

    def check_needs_update(self, context, src_driver_info, now_driver_info):
        :param src_driver_info: a dict containing original driver-specific info
                                in the database
               Example::

               {
                'port': '80',
                'logicalportip': '1.1.1.2',
                 ...
               }
        :param now_driver_info: a dict containing driver-specific info
                                provided by config file, array information
                                or capabilities. It is up to driver.
               Example::

               {
                'port': '80',
                'logicalportip': '1.1.1.1',
                 ...
               }
        :return: A bool value indicating if the driver need to update shares.

- Change the ``ensure_share`` driver interface from one share update
  to a bulk update, and rename it as ``ensure_shares``. There is no order for
  processing shares. If shares have to be processed in any particular order,
  it would be up to the share driver to do so.

  def ensure_shares(self, context, shares, share_server=None):
        """Invoked to ensure that shares are exported.

        Driver can use this method to update the list of export locations of
        the shares if it changes. To do that, a dictionary of shares should be
        returned.
        :shares: None or a list of all shares for updates.
        :return None or a dictionary of updates in the format::

            {
                '09960614-8574-4e03-89cf-7cf267b0bd08': {
                    'export_locations': [{...}, {...}],
                    'status': 'error',
                },

                '28f6eabb-4342-486a-a7f4-45688f0c0295': {
                    'export_locations': [{...}, {...}],
                    'status': 'available',
                },

            }

        """
        raise NotImplementedError()


- The status transition for this:
  * There are no changes regarding the share status and share instance status
    before the response is received by driver ``ensure_shares`` method.

  * If the share status is not ``available``, we will not go to driver
    interface, and just keep the original status(such as: extending,
    migrating, etc).
  * Accept ``status`` attribute to be set by driver, allowing for shares in
    transitional status to be updated by the driver. If the driver does not
    return a status for each share after the ensure_shares method be invoked,
    the shares status will transition back to ``available``. Perform these
    actions by acquiring a lock to read the current status, and releasing it
    at the end of the transaction.
  * The share information will be provided by the driver when ensuring a
    share.


Alternatives
------------

None

Data model impact
-----------------

None

REST API impact
---------------

None

Driver impact
-------------

Add driver interfaces::

    def ensure_shares(self, context, shares, share_server=None):
        """Invoked to ensure that available shares are exported.

        Driver can use this method to update the list of export locations of
        the shares if it changes. To do that, you should return dictionary of
        shares.
        :shares: None or a list of all shares for updates.
        Example::

            [
                {
                'id': 'd487b88d-e428-4230-a465-a800c2cce5f8',
                'share_id': 'f0e4bb5e-65f0-11e5-9d70-feff819cdc9f',
                'replica_state': 'in_sync',
                    ...
                'share_server_id': '4ce78e7b-0ef6-4730-ac2a-fd2defefbd05',
                'share_server': <models.ShareServer> or None,
                },
                {
                'id': '10e49c3e-aca9-483b-8c2d-1c337b38d6af',
                'share_id': 'f0e4bb5e-65f0-11e5-9d70-feff819cdc9f',
                'replica_state': 'active',
                    ...
                'share_server_id': 'f63629b3-e126-4448-bec2-03f788f76094',
                'share_server': <models.ShareServer> or None,
                },
                {
                'id': 'e82ff8b6-65f0-11e5-9d70-feff819cdc9f',
                'share_id': 'f0e4bb5e-65f0-11e5-9d70-feff819cdc9f',
                'replica_state': 'in_sync',
                    ...
                'share_server_id': '07574742-67ea-4dfd-9844-9fbd8ada3d87',
                'share_server': <models.ShareServer> or None,
                },
                ...
            ]
        :return None or a dictionary of updates in the format::

            {
                '09960614-8574-4e03-89cf-7cf267b0bd08': {
                    'export_locations': [{...}, {...}],
                    'status': 'error',
                },

                '28f6eabb-4342-486a-a7f4-45688f0c0295': {
                    'export_locations': [{...}, {...}],
                    'status': 'active',
                },
            }

        """
        raise NotImplementedError()

    def get_driver_info(self, context):
        """Get driver and array configuration parameters and return for
           assessment.
        :return None or a dictionary containing driver-specific infos::
             {
              'port': '80',
              'logicalportip': '1.1.1.1',
               ...
             }
        raise NotImplementedError()

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance impact
------------------

The effect of this change would be to make the manila-share service start up
faster in cases where nothing important has changed (the common case). The
effect will be small when ensure_share doesn't do anything or when the number
of shares is small, but could be very large for backends with a large number
of shares and expensive ensure_share operations, reducing a O(n) startup time
to O(1).

Other deployer impact
---------------------

None

Developer impact
----------------

This will require a change on the driver interface. To support this feature,
drivers will need to implement the new methods.
We will use "ensure_shares" instead of "ensure_share" method in manager, and
move the existing "ensure_share" code to "ensure_shares" in driver. If driver
can not support the "get_driver_info" methods, we will catch the
NotImplementedError exception raised by "get_driver_info" method, and invoke
"ensure_shares". It allows the existing code to work if driver can not support
the new methods.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

* zhongjun(jun.zhongjun2@gmail.com)

Work items
----------

* Implement the core feature with functional tempest and scenario test
  coverage
* Implement ensure in one of the first-party drivers
* Add documentation of this feature

Dependencies
============

None

Testing
=======

Due to the difficulty of restarting services during functional tests, it's
only practical to test this change with unit tests.

Documentation impact
====================

Documentation of this feature will be added to the admin guide and
developer reference.

References
==========

None