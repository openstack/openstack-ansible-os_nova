Windows Performance Optimization in OpenStack
=============================================

Running Windows instances in OpenStack often requires additional tuning
to achieve optimal performance. By default, OpenStack uses generic hardware
models that may not provide the best results for Windows workloads.

This guide outlines key image properties and configuration choices that
can significantly improve Windows instance performance.

Recommended Image Properties
----------------------------

When uploading or updating a Windows image, set the following properties:

.. code-block:: console

   openstack image create \
       --property os_type=windows \
       --property hw_machine_type=q35 \
       --property hw_disk_bus=scsi \
       --property hw_scsi_model=virtio-scsi
       <windows_image_name>

* ``os_type=windows`` enables Windows-specific handling in the compute driver
  For details on available enlightenments that can be exposed to the guest,
  see `Hyper-V Enlightenments <https://www.qemu.org/docs/master/system/i386/hyperv.html>`_
* ``hw_machine_type=q35`` provides a modern PCIe-based machine type, improving
  compatibility and performance for newer Windows versions
* ``hw_disk_bus=scsi`` attaches disks using a SCSI controller instead of IDE
  or default models
* ``hw_scsi_model=virtio-scsi`` enables a paravirtualized SCSI controller,
  improving performance and scalability compared to legacy disk models

.. note::

   * Ensure that VirtIO drivers are installed inside the Windows guest
     (especially for SCSI and network devices)
   * These properties must be applied before instance creation, changes
     require rebuilding or recreating the instance

Using optimized image properties such as ``q35``, ``virtio-scsi``, and
``scsi`` disk bus can significantly improve Windows instance performance
in OpenStack environments, especially for I/O intensive workloads.

In the 2026.1 release cycle, support for enabling I/O threads (iothreads)
in Nova has been introduced. This can further improve disk I/O performance,
especially for Windows workloads.
