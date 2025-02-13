.. _dynamic_configuration:

==============================
Dynamic Configuration Settings
==============================

Dynamic configuration is stored in the DCS (Distributed Configuration Store) and applied on all cluster nodes.

In order to change the dynamic configuration you can use either :ref:`patronictl_edit_config` tool or Patroni :ref:`REST API <rest_api>`.

-  **loop\_wait**: the number of seconds the loop will sleep. Default value: 10, minimum possible value: 1
-  **ttl**: the TTL to acquire the leader lock (in seconds). Think of it as the length of time before initiation of the automatic failover process. Default value: 30, minimum possible value: 20
-  **retry\_timeout**: timeout for DCS and PostgreSQL operation retries (in seconds). DCS or network issues shorter than this will not cause Patroni to demote the leader. Default value: 10, minimum possible value: 3

.. warning::
    when changing values of **loop_wait**, **retry_timeout**, or **ttl** you have to follow the rule:

    .. code-block:: python

        loop_wait + 2 * retry_timeout <= ttl


-  **maximum\_lag\_on\_failover**: the maximum bytes a follower may lag to be able to participate in leader election.
-  **maximum\_lag\_on\_syncnode**: the maximum bytes a synchronous follower may lag before it is considered as an unhealthy candidate and swapped by healthy asynchronous follower. Patroni utilize the max replica lsn if there is more than one follower, otherwise it will use leader's current wal lsn. Default is -1, Patroni will not take action to swap synchronous unhealthy follower when the value is set to 0 or below. Please set the value high enough so Patroni won't swap synchrounous follower frequently during high transaction volume.
-  **max\_timelines\_history**: maximum number of timeline history items kept in DCS.  Default value: 0. When set to 0, it keeps the full history in DCS.
-  **primary\_start\_timeout**: the amount of time a primary is allowed to recover from failures before failover is triggered (in seconds). Default is 300 seconds. When set to 0 failover is done immediately after a crash is detected if possible. When using asynchronous replication a failover can cause lost transactions. Worst case failover time for primary failure is: loop\_wait + primary\_start\_timeout + loop\_wait, unless primary\_start\_timeout is zero, in which case it's just loop\_wait. Set the value according to your durability/availability tradeoff.
-  **primary\_stop\_timeout**: The number of seconds Patroni is allowed to wait when stopping Postgres and effective only when synchronous_mode is enabled. When set to > 0 and the synchronous_mode is enabled, Patroni sends SIGKILL to the postmaster if the stop operation is running for more than the value set by primary\_stop\_timeout. Set the value according to your durability/availability tradeoff. If the parameter is not set or set <= 0, primary\_stop\_timeout does not apply.
-  **synchronous\_mode**: turns on synchronous replication mode. Possible values: ``off``, ``on``, ``quorum``. In this mode the leader takes care of management of ``synchronous_standby_names``, and only the last known leader, or one of synchronous replicas, are allowed to participate in leader race. Synchronous mode makes sure that successfully committed transactions will not be lost at failover, at the cost of losing availability for writes when Patroni cannot ensure transaction durability. See :ref:`replication modes documentation <replication_modes>` for details.
-  **synchronous\_mode\_strict**: prevents disabling synchronous replication if no synchronous replicas are available, blocking all client writes to the primary. See :ref:`replication modes documentation <replication_modes>` for details.
-  **synchronous\_node\_count**: if ``synchronous_mode`` is enabled, this parameter is used by Patroni to manage the precise number of synchronous standby instances and adjusts the state in DCS and the ``synchronous_standby_names`` parameter in PostgreSQL as members join and leave. If the parameter is set to a value higher than the number of eligible nodes, it will be automatically adjusted. Defaults to ``1``.
-  **failsafe\_mode**: Enables :ref:`DCS Failsafe Mode <dcs_failsafe_mode>`. Defaults to `false`.
-  **postgresql**:

   -  **use\_pg\_rewind**: whether or not to use pg_rewind. Defaults to `false`. Note that either the cluster must be initialized with ``data page checksums`` (``--data-checksums`` option for ``initdb``) and/or ``wal_log_hints`` must be set to ``on``, or ``pg_rewind`` will not work.
   -  **use\_slots**: whether or not to use replication slots. Defaults to `true` on PostgreSQL 9.4+.
   -  **recovery\_conf**: additional configuration settings written to recovery.conf when configuring follower. There is no recovery.conf anymore in PostgreSQL 12, but you may continue using this section, because Patroni handles it transparently.
   -  **parameters**: configuration parameters (GUCs) for Postgres in format ``{max_connections: 100, wal_level: "replica", max_wal_senders: 10, wal_log_hints: "on"}``. Many of these are required for replication to work.

   -  **pg\_hba**: list of lines that Patroni will use to generate ``pg_hba.conf``. Patroni ignores this parameter if ``hba_file`` PostgreSQL parameter is set to a non-default value.

      -  **- host all all 0.0.0.0/0 md5**
      -  **- host replication replicator 127.0.0.1/32 md5**: A line like this is required for replication.

   -  **pg\_ident**: list of lines that Patroni will use to generate ``pg_ident.conf``. Patroni ignores this parameter if ``ident_file`` PostgreSQL parameter is set to a non-default value.

      -  **- mapname1 systemname1 pguser1**
      -  **- mapname1 systemname2 pguser2**

-  **standby\_cluster**: if this section is defined, we want to bootstrap a standby cluster.

   -  **host**: an address of remote node
   -  **port**: a port of remote node
   -  **primary\_slot\_name**: which slot on the remote node to use for replication. This parameter is optional, the default value is derived from the instance name (see function `slot_name_from_member_name`).
   -  **create\_replica\_methods**: an ordered list of methods that can be used to bootstrap standby leader from the remote primary, can be different from the list defined in :ref:`postgresql_settings`
   -  **restore\_command**: command to restore WAL records from the remote primary to nodes in a standby cluster, can be different from the list defined in :ref:`postgresql_settings`
   -  **archive\_cleanup\_command**: cleanup command for standby leader
   -  **recovery\_min\_apply\_delay**: how long to wait before actually apply WAL records on a standby leader

-  **member_slots_ttl**: retention time of physical replication slots for replicas when they are shut down. Default value: `30min`. Set it to `0` if you want to keep the old behavior (when the member key expires from DCS, the slot is immediately removed). The feature works only starting from PostgreSQL 11.
-  **slots**: define permanent replication slots. These slots will be preserved during switchover/failover. Permanent slots that don't exist will be created by Patroni. With PostgreSQL 11 onwards permanent physical slots are created on all nodes and their position is advanced every **loop_wait** seconds. For PostgreSQL versions older than 11 permanent physical replication slots are maintained only on the current primary. The logical slots are copied from the primary to a standby with restart, and after that their position advanced every **loop_wait** seconds (if necessary). Copying logical slot files performed via ``libpq`` connection and using either rewind or superuser credentials (see **postgresql.authentication** section). There is always a chance that the logical slot position on the replica is a bit behind the former primary, therefore application should be prepared that some messages could be received the second time after the failover. The easiest way of doing so - tracking ``confirmed_flush_lsn``. Enabling permanent replication slots requires **postgresql.use_slots** to be set to ``true``. If there are permanent logical replication slots defined Patroni will automatically enable the ``hot_standby_feedback``. Since the failover of logical replication slots is unsafe on PostgreSQL 9.6 and older and PostgreSQL version 10 is missing some important functions, the feature only works with PostgreSQL 11+.

   -  **my\_slot\_name**: the name of the permanent replication slot. If the permanent slot name matches with the name of the current node it will not be created on this node. If you add a permanent physical replication slot which name matches the name of a Patroni member, Patroni will ensure that the slot that was created is not removed even if the corresponding member becomes unresponsive, situation which would normally result in the slot's removal by Patroni. Although this can be useful in some situations, such as when you want replication slots used by members to persist during temporary failures or when importing existing members to a new Patroni cluster (see :ref:`Convert a Standalone to a Patroni Cluster <existing_data>` for details), caution should be exercised by the operator that these clashes in names are not persisted in the DCS, when the slot is no longer required, due to its effect on normal functioning of Patroni.

      -  **type**: slot type. Could be ``physical`` or ``logical``. If the slot is logical, you have to additionally define ``database`` and ``plugin``. If the slot is physical, you can optionally define ``cluster_type``.
      -  **database**: the database name where logical slots should be created.
      -  **plugin**: the plugin name for the logical slot.
      -  **cluster_type**: the type of cluster (``primary`` or ``standby``) the slot should only be created on, otherwise it will not be created or an already existing slot will be dropped.

-  **ignore\_slots**: list of sets of replication slot properties for which Patroni should ignore matching slots. This configuration/feature/etc. is useful when some replication slots are managed outside of Patroni. Any subset of matching properties will cause a slot to be ignored.

   -  **name**: the name of the replication slot.
   -  **type**: slot type. Can be ``physical`` or ``logical``. If the slot is logical, you may additionally define ``database`` and/or ``plugin``.
   -  **database**: the database name (when matching a ``logical`` slot).
   -  **plugin**: the logical decoding plugin (when matching a ``logical`` slot).

Note: **slots** is a hashmap while **ignore_slots** is an array. For example:

.. code:: YAML

        slots:
          permanent_logical_slot_name:
            type: logical
            database: my_db
            plugin: test_decoding
          permanent_physical_slot_name:
            type: physical
          ...
        ignore_slots:
          - name: ignored_logical_slot_name
            type: logical
            database: my_db
            plugin: test_decoding
          - name: ignored_physical_slot_name
            type: physical
          ...

Note: When running PostgreSQL v11 or newer Patroni maintains physical replication slots on all nodes that could potentially become a leader, so that replica nodes keep WAL segments reserved if they are potentially required by other nodes. In case the node is absent and its member key in DCS gets expired, the corresponding replication slot is dropped after ``member_slots_ttl`` (default value is `30min`). You can increase or decrease retention based on your needs. Alternatively, if your cluster topology is static (fixed number of nodes that never change their names) you can configure permanent physical replication slots with names corresponding to the names of the nodes to avoid slots removal and recycling of WAL files while replica is temporarily down:

.. code:: YAML

        slots:
          node_name1:
            type: physical
          node_name2:
            type: physical
          node_name3:
            type: physical
          ...


.. warning::
   Permanent replication slots are synchronized only from the ``primary``/``standby_leader`` to replica nodes. That means, applications are supposed to be using them only from the leader node. Using them on replica nodes will cause indefinite growth of ``pg_wal`` on all other nodes in the cluster.
   An exception to that rule are physical slots that match the Patroni member names (created and maintained by Patroni). Those will be synchronized among all nodes as they are used for replication among them.


.. warning::
   Setting ``nostream`` tag on standby disables copying and synchronization of permanent logical replication slots on the node itself and all its cascading replicas if any.
