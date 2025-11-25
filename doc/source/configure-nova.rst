=================================================
Configuring the Compute (nova) service (optional)
=================================================

The Compute service (nova) handles the creation of virtual machines within an
OpenStack environment. Many of the default options used by OpenStack-Ansible
are found within ``defaults/main.yml`` within the nova role.

Availability zones
~~~~~~~~~~~~~~~~~~

Deployers with multiple availability zones can set the
``nova_nova_conf_overrides.DEFAULT.default_schedule_zone`` Ansible
variable to specify an availability zone for new requests. This is useful
in environments with different types of hypervisors, where builds are sent
to certain hardware types based on their resource requirements.

For example, if you have servers running on two racks without sharing the PDU.
These two racks can be grouped into two availability zones.
When one rack loses power, the other one still works. By spreading
your containers onto the two racks (availability zones), you will
improve your service availability.

Block device tuning for Ceph (RBD)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Enabling Ceph and defining ``nova_libvirt_images_rbd_pool`` changes two
libvirt configurations by default:

* hw_disk_discard: ``unmap``
* disk_cachemodes: ``network=writeback``

Setting ``hw_disk_discard`` to ``unmap`` in libvirt enables
discard (sometimes called TRIM) support for the underlying block device. This
allows reclaiming of unused blocks on the underlying disks.

Setting ``disk_cachemodes`` to ``network=writeback`` allows data to be written
into a cache on each change, but those changes are flushed to disk at a regular
interval. This can increase write performance on Ceph block devices.

You have the option to customize these settings using two Ansible
variables (defaults shown here):

.. code-block:: yaml

    nova_libvirt_hw_disk_discard: 'unmap'
    nova_libvirt_disk_cachemodes: 'network=writeback'

You can disable discard by setting ``nova_libvirt_hw_disk_discard`` to
``ignore``.  The ``nova_libvirt_disk_cachemodes`` can be set to an empty
string to disable ``network=writeback``.

The following minimal example configuration sets nova to use the
``ephemeral-vms`` Ceph pool. The following example uses cephx authentication,
and requires an existing ``cinder`` account for the ``ephemeral-vms`` pool:

.. code-block:: console

    nova_libvirt_images_rbd_pool: ephemeral-vms
    ceph_mons:
      - 172.29.244.151
      - 172.29.244.152
      - 172.29.244.153


If you have a different Ceph username for the pool, use it as:

.. code-block:: console

   cinder_ceph_client: <ceph-username>

* The `Ceph documentation for OpenStack`_ has additional information about
  these settings.
* `OpenStack-Ansible and Ceph Working Example`_


.. _Ceph documentation for OpenStack: http://docs.ceph.com/docs/master/rbd/rbd-openstack/
.. _OpenStack-Ansible and Ceph Working Example: https://www.openstackfaq.com/openstack-ansible-ceph/



Config drive
~~~~~~~~~~~~

By default, OpenStack-Ansible does not configure nova to force config drives
to be provisioned with every instance that nova builds. The metadata service
provides configuration information that is used by ``cloud-init`` inside the
instance. Config drives are only necessary when an instance does not have
``cloud-init`` installed or does not have support for handling metadata.

A deployer can set an Ansible variable to force config drives to be deployed
with every virtual machine:

.. code-block:: yaml

    nova_nova_conf_overrides:
      DEFAULT:
        force_config_drive: True

Certain formats of config drives can prevent instances from migrating properly
between hypervisors. If you need forced config drives and the ability
to migrate instances, set the config drive format to ``vfat`` using
the ``nova_nova_conf_overrides`` variable:

.. code-block:: yaml

    nova_nova_conf_overrides:
      DEFAULT:
        config_drive_format: vfat
        force_config_drive: True

Libvirtd connectivity and authentication
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

By default, OpenStack-Ansible configures the libvirt daemon in the following
way:

* TLS connections are enabled
* TCP plaintext connections are disabled
* Authentication over TCP connections uses SASL

You can customize these settings using the following Ansible variables:

.. code-block:: yaml

    # Enable libvirtd's TLS listener
    nova_libvirtd_listen_tls: 1

    # Disable libvirtd's plaintext TCP listener
    nova_libvirtd_listen_tcp: 0

    # Use SASL for authentication
    nova_libvirtd_auth_tcp: sasl

Multipath
~~~~~~~~~

Nova supports multipath for iSCSI-based storage. Enable multipath support in
nova through a configuration override:

.. code-block:: yaml

    nova_nova_conf_overrides:
      libvirt:
          iscsi_use_multipath: true

Shared storage and synchronized UID/GID
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Specify a custom UID for the nova user and GID for the nova group
to ensure they are identical on each host. This is helpful when using shared
storage on Compute nodes because it allows instances to migrate without
filesystem ownership failures.

By default, Ansible creates the nova user and group without specifying the
UID or GID. To specify custom values for the UID or GID, set the following
Ansible variables:

.. warning::

   Setting this value after deploying an environment with
   OpenStack-Ansible can cause failures, errors, and general instability. These
   values should only be set once before deploying an OpenStack environment
   and then never changed.

.. code-block:: yaml

    nova_system_user_uid = <specify a UID>
    nova_system_group_gid = <specify a GID>


Enabling Huge Pages
~~~~~~~~~~~~~~~~~~~

In order to enable Huge Pages for your kernel, as series of actions must be
done:

.. note::

  It is suggested to leverage ``group_vars``, as usually you need to enable
  Huge Pages only on specific set of hosts.
  For example, you can use `/etc/openstack_deploy/group_vars/compute_hosts.yml`
  file for defining variables discussed below.

#. In variables define a default size of Huge Pages.

    .. code-block:: yaml

      custom_huge_pages_size_mb: 2

#. Define custom GRUB records to enable Huge Pages

    .. code-block:: yaml

      openstack_host_custom_grub_options:
        - key: hugepagesz
          value: "{{ custom_huge_pages_size_mb }}M"
        - key: hugepages
          value: "{{ (ansible_facts['memtotal_mb'] - nova_reserved_host_memory_mb) // custom_huge_pages_size_mb }}"
        - key: transparent_hugepage
          value: never

#. Define override for the default ``dev-hugepages.mount``:

    .. code-block:: yaml

      openstack_hosts_systemd_mounts:
        - what: dev-hugepages
          mount_overrides_only: true
          type: hugetlbfs
          escape_name: false
          config_overrides:
            Mount:
              Options: "pagesize={{ custom_huge_pages_size_mb }}M"

#. Rollout changes to hosts

    .. code-block:: shell

      # openstack-ansible openstack.osa.openstack_hosts_setup --limit compute_hosts

#. Define flavors with Huge Pages support

    .. code-block:: yaml

      openstack_user_compute:
        flavors:
          - specs:
              - name: m1.small
                vcpus: 2
                ram: 4096
                disk: 0
              - name: m1.large
                vcpus: 8
                ram: 16384
                disk: 0
            extra_specs:
              hw:mem_page_size: large
              hw:cpu_policy: dedicated

#. Create defined flavors in region

    .. code-block:: shell

      # openstack-ansible openstack.osa.openstack_resources

Enabling post-copy for live migrations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

One of methodologies to ensure successful migration of high-loaded instances
is to use ``post-copy`` feature of Libvirt/QEMU. When this method is enabled,
when instance fails to migrate with provided timeout the migration is
forcefully completed, while remaining (untransferred memory) from the original
hypervisor are transferred once being an access attempt happens from the
instance.

For post-copy to work, it is important so satisfy multiple conditions:

* Nova configured to send post-copy to Libvirt while asking for migration:
  * ``live_migration_timeout_action`` should be set to ``force_complete``
  instead of default ``abort``
  * ``live_migration_permit_post_copy`` should be enabled
  * It is recommended to tune ``live_migration_completion_timeout``, as
  the default of 800 seconds might take too long before deciding that
  post-copy must be initiated. Please note, that the value of timeout
  is multiplied by amount of RAM and disk for the instance.
  So instance with 16Gb RAM and 50Gb disk may take over 14 hours before
  enforcing post-copy mechanism.
* Hypervisor needs to have ``vm.unprivileged_userfaultfd`` enabled to
  allow post-copy to happen
* Ensure Open vSwitch is not using ``mlockall`` for startup


Below you can find an example configuration for OpenStack-Ansible to
enable post-copy migration for your instances. For that, place the
following content into ``/etc/openstack_deploy/group_vars/nova_compute.yml``:

.. code:: yaml

  nova_nova_conf_overrides:
    libvirt:
      live_migration_permit_post_copy: true
      live_migration_timeout_action: force_complete
      live_migration_completion_timeout: 30

  openstack_user_kernel_options:
    - key: vm.unprivileged_userfaultfd
      value: 1

Once configuration is in place, you need to run following playbooks to
apply changes:

.. code-block:: console

  # openstack-ansible openstack.osa.openstack_hosts_setup --tags openstack_hosts-install --limit nova_compute
  # openstack-ansible openstack.osa.nova --tags post-install --limit nova_compute
