.. _rados_pools:

=======
 Pools
=======
Pools are logical partitions that are used to store objects.

Pools provide:

- **Resilience**: It is possible to set the number of OSDs that are allowed to
  fail without any data being lost. If your cluster uses replicated pools, the
  number of OSDs that can fail without data loss is equal to the number of
  replicas.
  
  For example: a typical configuration stores an object and two replicas
  (copies) of each RADOS object (that is: ``size = 3``), but you can configure
  the number of replicas on a per-pool basis. For `erasure-coded pools
  <../erasure-code>`_, resilience is defined as the number of coding chunks
  (for example, ``m = 2`` in the default **erasure code profile**).

- **Placement Groups**: You can set the number of placement groups (PGs) for
  the pool. In a typical configuration, the target number of PGs is
  approximately one hundred PGs per OSD. This provides reasonable balancing
  without consuming excessive computing resources.  When setting up multiple
  pools, be careful to set an appropriate number of PGs for each pool and for
  the cluster as a whole. Each PG belongs to a specific pool: when multiple
  pools use the same OSDs, make sure that the **sum** of PG replicas per OSD is
  in the desired PG-per-OSD target range. To calculate an appropriate number of
  PGs for your pools, use the `pgcalc`_ tool.

- **CRUSH Rules**: When data is stored in a pool, the placement of the object
  and its replicas (or chunks, in the case of erasure-coded pools) in your
  cluster is governed by CRUSH rules. Custom CRUSH rules can be created for a
  pool if the default rule does not fit your use case.

- **Snapshots**: The command ``ceph osd pool mksnap`` creates a snapshot of a
  pool.

Pool Names
==========

Pool names beginning with ``.`` are reserved for use by Ceph's internal
operations. Do not create or manipulate pools with these names.


List Pools
==========

To list your cluster's pools, run the following command:

.. prompt:: bash $

   ceph osd lspools

.. _createpool:

Creating a Pool
===============

Before creating a pool, consult `Pool, PG and CRUSH Config Reference`_.  Your
Ceph configuration file contains a setting (namely, ``pg_num``) that determines
the number of PGs.  However, this setting's default value is NOT appropriate
for most systems.  In most cases, you should override this default value when
creating your pool.  For details on PG numbers, see `setting the number of
placement groups`_

For example:

.. prompt:: bash $

    osd_pool_default_pg_num = 128
    osd_pool_default_pgp_num = 128

.. note:: In Luminous and later releases, each pool must be associated with the
   application that will be using the pool. For more information, see
   `Associating a Pool to an Application`_ below.

To create a pool, run one of the following commands:

.. prompt:: bash $

    ceph osd pool create {pool-name} [{pg-num} [{pgp-num}]] [replicated] \
             [crush-rule-name] [expected-num-objects]

or:

.. prompt:: bash $

    ceph osd pool create {pool-name} [{pg-num} [{pgp-num}]]   erasure \
             [erasure-code-profile] [crush-rule-name] [expected_num_objects] [--autoscale-mode=<on,off,warn>]

For a brief description of the elements of the above commands, consult the
following:

.. describe:: {pool-name}

   The name of the pool. It must be unique.

   :Type: String
   :Required: Yes.

.. describe:: {pg-num}

   The total number of PGs in the pool. For details on calculating an
   appropriate number, see :ref:`placement groups`. The default value ``8`` is
   NOT suitable for most systems.

  :Type: Integer
  :Required: Yes.
  :Default: 8

.. describe:: {pgp-num}

   The total number of PGs for placement purposes. This **should be equal to
   the total number of PGs**, except briefly while ``pg_num`` is being
   increased or decreased. 

  :Type: Integer
  :Required: Yes. If no value has been specified in the command, then the default value is used (unless a different value has been set in Ceph configuration).
  :Default: 8

.. describe:: {replicated|erasure}

   The pool type. This can be either **replicated** (to recover from lost OSDs
   by keeping multiple copies of the objects) or **erasure** (to achieve a kind
   of `generalized parity RAID <../erasure-code>`_ capability).  The
   **replicated** pools require more raw storage but can implement all Ceph
   operations. The **erasure** pools require less raw storage but can perform
   only some Ceph tasks and may provide decreased performance.

  :Type: String
  :Required: No.
  :Default: replicated

.. describe:: [crush-rule-name]

   The name of the CRUSH rule to use for this pool. The specified rule must
   exist; otherwise the command will fail.

   :Type: String
   :Required: No.
   :Default: For **replicated** pools, it is the rule specified by the :confval:`osd_pool_default_crush_rule` configuration variable. This rule must exist.  For **erasure** pools, it is the ``erasure-code`` rule if the ``default`` `erasure code profile`_ is used or the ``{pool-name}`` rule  if not. This rule will be created implicitly if it doesn't already exist.

.. describe:: [erasure-code-profile=profile]

   For **erasure** pools only. Instructs Ceph to use the specified `erasure
   code profile`_. This profile must be an existing profile as defined by **osd
   erasure-code-profile set**.

  :Type: String
  :Required: No.

.. _erasure code profile: ../erasure-code-profile

.. describe:: --autoscale-mode=<on,off,warn>

   - ``on``: the Ceph cluster will autotune or recommend changes to the number of PGs in your pool based on actual usage.
   - ``warn``: the Ceph cluster will autotune or recommend changes to the number of PGs in your pool based on actual usage.
   - ``off``: refer to :ref:`placement groups` for more information.

  :Type: String
  :Required: No.
  :Default: The default behavior is determined by the :confval:`osd_pool_default_pg_autoscale_mode` option.

.. describe:: [expected-num-objects]

   The expected number of RADOS objects for this pool. By setting this value and
   assigning a negative value to **filestore merge threshold**, you arrange
   for the PG folder splitting to occur at the time of pool creation and
   avoid the latency impact that accompanies runtime folder splitting.

   :Type: Integer
   :Required: No.
   :Default: 0, no splitting at the time of pool creation.

.. _associate-pool-to-application:

Associating a Pool to an Application
====================================

Pools need to be associated with an application before use. Pools that will be
used with CephFS or pools that are automatically created by RGW are
automatically associated. Pools that are intended for use with RBD should be
initialized using the ``rbd`` tool (see `Block Device Commands`_ for more
information).

For other cases, you can manually associate a free-form application name to
a pool.:

.. prompt:: bash $

   ceph osd pool application enable {pool-name} {application-name}

.. note:: CephFS uses the application name ``cephfs``, RBD uses the
   application name ``rbd``, and RGW uses the application name ``rgw``.

Set Pool Quotas
===============

You can set pool quotas for the maximum number of bytes and/or the maximum
number of objects per pool:

.. prompt:: bash $

   ceph osd pool set-quota {pool-name} [max_objects {obj-count}] [max_bytes {bytes}]

For example:

.. prompt:: bash $

   ceph osd pool set-quota data max_objects 10000

To remove a quota, set its value to ``0``.


Delete a Pool
=============

To delete a pool, execute:

.. prompt:: bash $

   ceph osd pool delete {pool-name} [{pool-name} --yes-i-really-really-mean-it]


To remove a pool the mon_allow_pool_delete flag must be set to true in the Monitor's
configuration. Otherwise they will refuse to remove a pool.

See `Monitor Configuration`_ for more information.

.. _Monitor Configuration: ../../configuration/mon-config-ref

If you created your own rules for a pool you created, you should consider
removing them when you no longer need your pool:

.. prompt:: bash $

   ceph osd pool get {pool-name} crush_rule

If the rule was "123", for example, you can check the other pools like so:

.. prompt:: bash $

	ceph osd dump | grep "^pool" | grep "crush_rule 123"

If no other pools use that custom rule, then it's safe to delete that
rule from the cluster.

If you created users with permissions strictly for a pool that no longer
exists, you should consider deleting those users too:


.. prompt:: bash $

	ceph auth ls | grep -C 5 {pool-name}
	ceph auth del {user}


Rename a Pool
=============

To rename a pool, execute:

.. prompt:: bash $

   ceph osd pool rename {current-pool-name} {new-pool-name}

If you rename a pool and you have per-pool capabilities for an authenticated
user, you must update the user's capabilities (i.e., caps) with the new pool
name.

Show Pool Statistics
====================

To show a pool's utilization statistics, execute:

.. prompt:: bash $

   rados df

Additionally, to obtain I/O information for a specific pool or all, execute:

.. prompt:: bash $

   ceph osd pool stats [{pool-name}]


Make a Snapshot of a Pool
=========================

To make a snapshot of a pool, execute:

.. prompt:: bash $

   ceph osd pool mksnap {pool-name} {snap-name}

Remove a Snapshot of a Pool
===========================

To remove a snapshot of a pool, execute:

.. prompt:: bash $

   ceph osd pool rmsnap {pool-name} {snap-name}

.. _setpoolvalues:


Set Pool Values
===============

To set a value to a pool, execute the following:

.. prompt:: bash $

   ceph osd pool set {pool-name} {key} {value}

You may set values for the following keys:

.. _compression_algorithm:

.. describe:: compression_algorithm

   Sets inline compression algorithm to use for underlying BlueStore. This setting overrides the global setting
   :confval:`bluestore_compression_algorithm`.

   :Type: String
   :Valid Settings: ``lz4``, ``snappy``, ``zlib``, ``zstd``

.. describe:: compression_mode

   Sets the policy for the inline compression algorithm for underlying BlueStore. This setting overrides the
   global setting :confval:`bluestore_compression_mode`.

   :Type: String
   :Valid Settings: ``none``, ``passive``, ``aggressive``, ``force``

.. describe:: compression_min_blob_size

   Chunks smaller than this are never compressed. This setting overrides the global settings of
   :confval:`bluestore_compression_min_blob_size`, :confval:`bluestore_compression_min_blob_size_hdd` and
   :confval:`bluestore_compression_min_blob_size_ssd`

   :Type: Unsigned Integer

.. describe:: compression_max_blob_size

   Chunks larger than this are broken into smaller blobs sizing
   ``compression_max_blob_size`` before being compressed.

   :Type: Unsigned Integer

.. _size:

.. describe:: size

   Sets the number of replicas for objects in the pool.
   See `Set the Number of Object Replicas`_ for further details.
   Replicated pools only.

   :Type: Integer

.. _min_size:

.. describe:: min_size

   Sets the minimum number of replicas required for I/O.
   See `Set the Number of Object Replicas`_ for further details.
   In the case of Erasure Coded pools this should be set to a value
   greater than 'k' since if we allow IO at the value 'k' there is no
   redundancy and data will be lost in the event of a permanent OSD
   failure. For more information see `Erasure Code <../erasure-code>`_

   :Type: Integer
   :Version: ``0.54`` and above

.. _pg_num:

.. describe:: pg_num

   The effective number of placement groups to use when calculating
   data placement.

   :Type: Integer
   :Valid Range: Superior to ``pg_num`` current value.

.. _pgp_num:

.. describe:: pgp_num

   The effective number of placement groups for placement to use
   when calculating data placement.

   :Type: Integer
   :Valid Range: Equal to or less than ``pg_num``.

.. _crush_rule:

.. describe:: crush_rule

   The rule to use for mapping object placement in the cluster.

   :Type: String

.. _allow_ec_overwrites:

.. describe:: allow_ec_overwrites


   Whether writes to an erasure coded pool can update part
   of an object, so cephfs and rbd can use it. See
   `Erasure Coding with Overwrites`_ for more details.

   :Type: Boolean

   .. versionadded:: 12.2.0

.. _hashpspool:

.. describe:: hashpspool

   Set/Unset HASHPSPOOL flag on a given pool.

   :Type: Integer
   :Valid Range: 1 sets flag, 0 unsets flag

.. _nodelete:

.. describe:: nodelete

   Set/Unset NODELETE flag on a given pool.

   :Type: Integer
   :Valid Range: 1 sets flag, 0 unsets flag
   :Version: Version ``FIXME``

.. _nopgchange:

.. describe:: nopgchange

   :Description: Set/Unset NOPGCHANGE flag on a given pool.
   :Type: Integer
   :Valid Range: 1 sets flag, 0 unsets flag
   :Version: Version ``FIXME``

.. _nosizechange:

.. describe:: nosizechange

   Set/Unset NOSIZECHANGE flag on a given pool.

   :Type: Integer
   :Valid Range: 1 sets flag, 0 unsets flag
   :Version: Version ``FIXME``

.. _bulk:

.. describe:: bulk

   Set/Unset bulk flag on a given pool.

   :Type: Boolean
   :Valid Range: true/1 sets flag, false/0 unsets flag

.. _write_fadvise_dontneed:

.. describe:: write_fadvise_dontneed

   Set/Unset WRITE_FADVISE_DONTNEED flag on a given pool.

   :Type: Integer
   :Valid Range: 1 sets flag, 0 unsets flag

.. _noscrub:

.. describe:: noscrub

   Set/Unset NOSCRUB flag on a given pool.

   :Type: Integer
   :Valid Range: 1 sets flag, 0 unsets flag

.. _nodeep-scrub:

.. describe:: nodeep-scrub

   Set/Unset NODEEP_SCRUB flag on a given pool.

   :Type: Integer
   :Valid Range: 1 sets flag, 0 unsets flag

.. _hit_set_type:

.. describe:: hit_set_type

   Enables hit set tracking for cache pools.
   See `Bloom Filter`_ for additional information.

   :Type: String
   :Valid Settings: ``bloom``, ``explicit_hash``, ``explicit_object``
   :Default: ``bloom``. Other values are for testing.

.. _hit_set_count:

.. describe:: hit_set_count

   The number of hit sets to store for cache pools. The higher
   the number, the more RAM consumed by the ``ceph-osd`` daemon.

   :Type: Integer
   :Valid Range: ``1``. Agent doesn't handle > 1 yet.

.. _hit_set_period:

.. describe:: hit_set_period

   The duration of a hit set period in seconds for cache pools.
   The higher the number, the more RAM consumed by the
   ``ceph-osd`` daemon.

   :Type: Integer
   :Example: ``3600`` 1hr

.. _hit_set_fpp:

.. describe:: hit_set_fpp

   The false positive probability for the ``bloom`` hit set type.
   See `Bloom Filter`_ for additional information.

   :Type: Double
   :Valid Range: 0.0 - 1.0
   :Default: ``0.05``

.. _cache_target_dirty_ratio:

.. describe:: cache_target_dirty_ratio

   The percentage of the cache pool containing modified (dirty)
   objects before the cache tiering agent will flush them to the
   backing storage pool.

   :Type: Double
   :Default: ``.4``

.. _cache_target_dirty_high_ratio:

.. describe:: cache_target_dirty_high_ratio

   The percentage of the cache pool containing modified (dirty)
   objects before the cache tiering agent will flush them to the
   backing storage pool with a higher speed.

   :Type: Double
   :Default: ``.6``

.. _cache_target_full_ratio:

.. describe:: cache_target_full_ratio

   The percentage of the cache pool containing unmodified (clean)
   objects before the cache tiering agent will evict them from the
   cache pool.

   :Type: Double
   :Default: ``.8``

.. _target_max_bytes:

.. describe:: target_max_bytes

   Ceph will begin flushing or evicting objects when the
   ``max_bytes`` threshold is triggered.

   :Type: Integer
   :Example: ``1000000000000``  #1-TB

.. _target_max_objects:

.. describe:: target_max_objects

   Ceph will begin flushing or evicting objects when the
   ``max_objects`` threshold is triggered.

   :Type: Integer
   :Example: ``1000000`` #1M objects


.. describe:: hit_set_grade_decay_rate

   Temperature decay rate between two successive hit_sets

   :Type: Integer
   :Valid Range: 0 - 100
   :Default: ``20``

.. describe:: hit_set_search_last_n

   Count at most N appearance in hit_sets for temperature calculation

   :Type: Integer
   :Valid Range: 0 - hit_set_count
   :Default: ``1``

.. _cache_min_flush_age:

.. describe:: cache_min_flush_age

   The time (in seconds) before the cache tiering agent will flush
   an object from the cache pool to the storage pool.

   :Type: Integer
   :Example: ``600`` 10min

.. _cache_min_evict_age:

.. describe:: cache_min_evict_age

   The time (in seconds) before the cache tiering agent will evict
   an object from the cache pool.

   :Type: Integer
   :Example: ``1800`` 30min

.. _fast_read:

.. describe:: fast_read

   On Erasure Coding pool, if this flag is turned on, the read request
   would issue sub reads to all shards, and waits until it receives enough
   shards to decode to serve the client. In the case of jerasure and isa
   erasure plugins, once the first K replies return, client's request is
   served immediately using the data decoded from these replies. This
   helps to tradeoff some resources for better performance. Currently this
   flag is only supported for Erasure Coding pool.

   :Type: Boolean
   :Defaults: ``0``

.. _scrub_min_interval:

.. describe:: scrub_min_interval

   The minimum interval in seconds for pool scrubbing when
   load is low. If it is 0, the value osd_scrub_min_interval
   from config is used.

   :Type: Double
   :Default: ``0``

.. _scrub_max_interval:

.. describe:: scrub_max_interval

   The maximum interval in seconds for pool scrubbing
   irrespective of cluster load. If it is 0, the value
   osd_scrub_max_interval from config is used.

   :Type: Double
   :Default: ``0``

.. _deep_scrub_interval:

.. describe:: deep_scrub_interval

   The interval in seconds for pool “deep” scrubbing. If it
   is 0, the value osd_deep_scrub_interval from config is used.

   :Type: Double
   :Default: ``0``

.. _recovery_priority:

.. describe:: recovery_priority

   When a value is set it will increase or decrease the computed
   reservation priority. This value must be in the range -10 to
   10.  Use a negative priority for less important pools so they
   have lower priority than any new pools.

   :Type: Integer
   :Default: ``0``


.. _recovery_op_priority:

.. describe:: recovery_op_priority

   Specify the recovery operation priority for this pool instead of :confval:`osd_recovery_op_priority`.

   :Type: Integer
   :Default: ``0``


Get Pool Values
===============

To get a value from a pool, execute the following:

.. prompt:: bash $

   ceph osd pool get {pool-name} {key}

You may get values for the following keys:

``size``

:Description: see size_

:Type: Integer

``min_size``

:Description: see min_size_

:Type: Integer
:Version: ``0.54`` and above

``pg_num``

:Description: see pg_num_

:Type: Integer


``pgp_num``

:Description: see pgp_num_

:Type: Integer
:Valid Range: Equal to or less than ``pg_num``.


``crush_rule``

:Description: see crush_rule_


``hit_set_type``

:Description: see hit_set_type_

:Type: String
:Valid Settings: ``bloom``, ``explicit_hash``, ``explicit_object``

``hit_set_count``

:Description: see hit_set_count_

:Type: Integer


``hit_set_period``

:Description: see hit_set_period_

:Type: Integer


``hit_set_fpp``

:Description: see hit_set_fpp_

:Type: Double


``cache_target_dirty_ratio``

:Description: see cache_target_dirty_ratio_

:Type: Double


``cache_target_dirty_high_ratio``

:Description: see cache_target_dirty_high_ratio_

:Type: Double


``cache_target_full_ratio``

:Description: see cache_target_full_ratio_

:Type: Double


``target_max_bytes``

:Description: see target_max_bytes_

:Type: Integer


``target_max_objects``

:Description: see target_max_objects_

:Type: Integer


``cache_min_flush_age``

:Description: see cache_min_flush_age_

:Type: Integer


``cache_min_evict_age``

:Description: see cache_min_evict_age_

:Type: Integer


``fast_read``

:Description: see fast_read_

:Type: Boolean


``scrub_min_interval``

:Description: see scrub_min_interval_

:Type: Double


``scrub_max_interval``

:Description: see scrub_max_interval_

:Type: Double


``deep_scrub_interval``

:Description: see deep_scrub_interval_

:Type: Double


``allow_ec_overwrites``

:Description: see allow_ec_overwrites_

:Type: Boolean


``recovery_priority``

:Description: see recovery_priority_

:Type: Integer


``recovery_op_priority``

:Description: see recovery_op_priority_

:Type: Integer


Set the Number of Object Replicas
=================================

To set the number of object replicas on a replicated pool, execute the following:

.. prompt:: bash $

   ceph osd pool set {poolname} size {num-replicas}

.. important:: The ``{num-replicas}`` includes the object itself.
   If you want the object and two copies of the object for a total of
   three instances of the object, specify ``3``.

For example:

.. prompt:: bash $

   ceph osd pool set data size 3

You may execute this command for each pool. **Note:** An object might accept
I/Os in degraded mode with fewer than ``pool size`` replicas.  To set a minimum
number of required replicas for I/O, you should use the ``min_size`` setting.
For example:

.. prompt:: bash $

   ceph osd pool set data min_size 2

This ensures that no object in the data pool will receive I/O with fewer than
``min_size`` replicas.


Get the Number of Object Replicas
=================================

To get the number of object replicas, execute the following:

.. prompt:: bash $

   ceph osd dump | grep 'replicated size'

Ceph will list the pools, with the ``replicated size`` attribute highlighted.
By default, ceph creates two replicas of an object (a total of three copies, or
a size of 3).


.. _pgcalc: https://old.ceph.com/pgcalc/
.. _Pool, PG and CRUSH Config Reference: ../../configuration/pool-pg-config-ref
.. _Bloom Filter: https://en.wikipedia.org/wiki/Bloom_filter
.. _setting the number of placement groups: ../placement-groups#set-the-number-of-placement-groups
.. _Erasure Coding with Overwrites: ../erasure-code#erasure-coding-with-overwrites
.. _Block Device Commands: ../../../rbd/rados-rbd-cmds/#create-a-block-device-pool
