Instance Recovery Using Rescue Image
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Upload Rescue Image
-------------------

Upload the SystemRescue .iso image (example uses ``systemrescue-13.00``
you can download any other version on its `website <https://www.system-rescue.org/download/>`_.):

   .. code-block:: console

      openstack image create --disk-format iso \
          --container-format bare \
          --property hw_rescue_device=disk \
          --property hw_rescue_bus=virtio \
          --file systemrescue-13.00-amd64.iso \
          --public rescuecd13

   .. note::

      For volume-backed instances, additional image properties are required:

      * ``hw_rescue_device=disk`` ensures the rescue image is attached as a disk
        instead of a CD-ROM during rescue mode
      * ``hw_rescue_bus=virtio`` ensures the rescue disk uses the VirtIO bus,
        which is required for proper device detection in many environments
      * This allows proper access to the instance root volume for recovery

Enter Rescue Mode
-----------------

#. Put the instance into rescue mode:

   .. code-block:: console

      openstack server rescue --image rescuecd13 <server_id>

   .. note::

      This action can also be performed via the Horizon dashboard.

#. After the command completes, the instance will boot into the rescue
   environment, allowing recovery actions to be performed.

Exit Rescue Mode
----------------

Return the instance to normal operation:

   .. code-block:: console

      openstack server unrescue <server_id>
