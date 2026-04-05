Installing OS from ISO image in OpenStack
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

#. Upload the ISO image:

   .. code-block:: console

      openstack image create \
          --file input.iso \
          --disk-format iso \
          --container-format bare \
          --public \
          my-iso-image

#. Choose the disk bus depending on OS requirements:

   * VirtIO (recommended):

     .. code-block:: console

        --block-device source_type=blank,destination_type=volume,volume_size=20,disk_bus=virtio,boot_index=1

     * High performance
     * Requires driver support

   * IDE (compatibility, e.g. Kerio Control, earlier Windows versions):

     .. code-block:: console

        --block-device source_type=blank,destination_type=volume,volume_size=20,disk_bus=ide,boot_index=1

     * Lower performance
     * No additional drivers required

#. And create an instance with a CD-ROM and a target disk:

   .. code-block:: console

      openstack server create \
          --flavor my_flavor \
          --block-device source_type=blank,destination_type=volume,volume_size=20,boot_index=1 \
          --block-device source_type=image,uuid=<IMAGE_ID>,destination_type=volume,volume_size=2,disk_bus=ide,device_type=cdrom,boot_index=0 \
          --network <NETWORK_ID> \
          install-vm

   .. note::

      * ``boot_index=0`` = CD-ROM (boot first)
      * ``boot_index=1`` = installation disk
      * CD-ROM is created as a volume

#. After instance creation open the VNC console and complete OS installation
   onto the blank volume.

#. Reboot the instance.

   .. note::

      The CD-ROM is attached as a volume-backed root device, so it cannot be detached.
      In order to boot from OS we can create a snapshot of the volume where the OS is
      installed and boot a new instance from that snapshot.

#. Get the Volume ID (search for first volume in ``volumes`` property):

   .. code-block:: console

      openstack server show install-vm

#. Verify the Volume is bootable:

   .. code-block:: console

      openstack volume show <VOLUME_ID>

   Expected output:

   .. code-block:: console

     ---------------------------------------------------+
     | Field                          | Value           |
     +--------------------------------+-----------------+
     | bootable                       | true            |
     +--------------------------------+-----------------+

#. Shut off instance and create a volume snapshot:

   .. code-block:: console

      openstack volume snapshot create \
         --force --volume <VOLUME_ID> \
          installed-os-snapshot

#. Verify snapshot status:

   .. code-block:: console

      openstack volume snapshot show <SNAPSHOT_ID>

   Expected output:

   .. code-block:: console

     ---------------------------------------------------+
     | Field                          | Value           |
     +--------------------------------+-----------------+
     | status                         | available       |
     +--------------------------------+-----------------+

Boot Instance from Snapshot
---------------------------

#. Create a new instance from the snapshot:

   .. code-block:: console

      openstack server create \
          --flavor my_flavor \
          --block-device source_type=snapshot,uuid=<SNAPSHOT_ID>,destination_type=volume,volume_size=20,boot_index=0 \
          --network <NETWORK_ID> \
          vm-from-snapshot
