Virtual Machine Generation ID Device
====================================

..
   Copyright (C) 2016 Red Hat, Inc.
   Copyright (C) 2017 Skyport Systems, Inc.

   This work is licensed under the terms of the GNU GPL, version 2 or later.
   See the COPYING file in the top-level directory.

The VM generation ID (``vmgenid``) device is an emulated device which
exposes a 128-bit, cryptographically random, integer value identifier,
referred to as a Globally Unique Identifier, or GUID.

This allows management applications (e.g. libvirt) to notify the guest
operating system when the virtual machine is executed with a different
configuration (e.g. snapshot execution or creation from a template).  The
guest operating system notices the change, and is then able to react as
appropriate by marking its copies of distributed databases as dirty,
re-initializing its random number generator etc.


Requirements
------------

These requirements are extracted from the "How to implement virtual machine
generation ID support in a virtualization platform" section of
`the Microsoft Virtual Machine Generation ID specification
<http://go.microsoft.com/fwlink/?LinkId=260709>`_ dated August 1, 2012.

- **R1a** The generation ID shall live in an 8-byte aligned buffer.

- **R1b** The buffer holding the generation ID shall be in guest RAM,
  ROM, or device MMIO range.

- **R1c** The buffer holding the generation ID shall be kept separate from
  areas used by the operating system.

- **R1d** The buffer shall not be covered by an AddressRangeMemory or
  AddressRangeACPI entry in the E820 or UEFI memory map.

- **R1e** The generation ID shall not live in a page frame that could be
  mapped with caching disabled. (In other words, regardless of whether the
  generation ID lives in RAM, ROM or MMIO, it shall only be mapped as
  cacheable.)

- **R2** to **R5** [These AML requirements are isolated well enough in the
  Microsoft specification for us to simply refer to them here.]

- **R6** The hypervisor shall expose a _HID (hardware identifier) object
  in the VMGenId device's scope that is unique to the hypervisor vendor.


QEMU Implementation
-------------------

The above-mentioned specification does not dictate which ACPI descriptor table
will contain the VM Generation ID device.  Other implementations (Hyper-V and
Xen) put it in the main descriptor table (Differentiated System Description
Table or DSDT).  For ease of debugging and implementation, we have decided to
put it in its own Secondary System Description Table, or SSDT.

The following is a dump of the contents from a running system::

  # iasl -p ./SSDT -d /sys/firmware/acpi/tables/SSDT

  Intel ACPI Component Architecture
  ASL+ Optimizing Compiler version 20150717-64
  Copyright (c) 2000 - 2015 Intel Corporation

  Reading ACPI table from file /sys/firmware/acpi/tables/SSDT - Length
  00000198 (0x0000C6)
  ACPI: SSDT 0x0000000000000000 0000C6 (v01 BOCHS  VMGENID  00000001 BXPC 00000001)
  Acpi table [SSDT] successfully installed and loaded
  Pass 1 parse of [SSDT]
  Pass 2 parse of [SSDT]
  Parsing Deferred Opcodes (Methods/Buffers/Packages/Regions)

  Parsing completed
  Disassembly completed
  ASL Output:    ./SSDT.dsl - 1631 bytes
  # cat SSDT.dsl
  /*
   * Intel ACPI Component Architecture
   * AML/ASL+ Disassembler version 20150717-64
   * Copyright (c) 2000 - 2015 Intel Corporation
   *
   * Disassembling to symbolic ASL+ operators
   *
   * Disassembly of /sys/firmware/acpi/tables/SSDT, Sun Feb  5 00:19:37 2017
   *
   * Original Table Header:
   *     Signature        "SSDT"
   *     Length           0x000000CA (202)
   *     Revision         0x01
   *     Checksum         0x4B
   *     OEM ID           "BOCHS "
   *     OEM Table ID     "VMGENID"
   *     OEM Revision     0x00000001 (1)
   *     Compiler ID      "BXPC"
   *     Compiler Version 0x00000001 (1)
   */
  DefinitionBlock ("/sys/firmware/acpi/tables/SSDT.aml", "SSDT", 1, "BOCHS ", "VMGENID", 0x00000001)
  {
      Name (VGIA, 0x07FFF000)
      Scope (\_SB)
      {
          Device (VGEN)
          {
              Name (_HID, "QEMUVGID")  // _HID: Hardware ID
              Name (_CID, "VM_Gen_Counter")  // _CID: Compatible ID
              Name (_DDN, "VM_Gen_Counter")  // _DDN: DOS Device Name
              Method (_STA, 0, NotSerialized)  // _STA: Status
              {
                  Local0 = 0x0F
                  If ((VGIA == Zero))
                  {
                      Local0 = Zero
                  }

                  Return (Local0)
              }

              Method (ADDR, 0, NotSerialized)
              {
                  Local0 = Package (0x02) {}
                  Index (Local0, Zero) = (VGIA + 0x28)
                  Index (Local0, One) = Zero
                  Return (Local0)
              }
          }
      }

      Method (\_GPE._E05, 0, NotSerialized)  // _Exx: Edge-Triggered GPE
      {
          Notify (\_SB.VGEN, 0x80) // Status Change
      }
  }


Design Details:
---------------

Requirements R1a through R1e dictate that the memory holding the
VM Generation ID must be allocated and owned by the guest firmware,
in this case BIOS or UEFI.  However, to be useful, QEMU must be able to
change the contents of the memory at runtime, specifically when starting a
backed-up or snapshotted image.  In order to do this, QEMU must know the
address that has been allocated.

The mechanism chosen for this memory sharing is writable fw_cfg blobs.
These are data object that are visible to both QEMU and guests, and are
addressable as sequential files.

More information about fw_cfg can be found in :doc:`fw_cfg`.

Two fw_cfg blobs are used in this case:

``/etc/vmgenid_guid``

- contains the actual VM Generation ID GUID
- read-only to the guest

``/etc/vmgenid_addr``

- contains the address of the downloaded vmgenid blob
- writable by the guest


QEMU sends the following commands to the guest at startup:

1. Allocate memory for vmgenid_guid fw_cfg blob.
2. Write the address of vmgenid_guid into the SSDT (VGIA ACPI variable as
   shown above in the iasl dump).  Note that this change is not propagated
   back to QEMU.
3. Write the address of vmgenid_guid back to QEMU's copy of vmgenid_addr
   via the fw_cfg DMA interface.

After step 3, QEMU is able to update the contents of vmgenid_guid at will.

Since BIOS or UEFI does not necessarily run when we wish to change the GUID,
the value of VGIA is persisted via the VMState mechanism.

As spelled out in the specification, any change to the GUID executes an
ACPI notification.  The exact handler to use is not specified, so the vmgenid
device uses the first unused one:  ``\_GPE._E05``.


Endian-ness Considerations:
---------------------------

Although not specified in Microsoft's document, it is assumed that the
device is expected to use little-endian format.

All GUID passed in via command line or monitor are treated as big-endian.
GUID values displayed via monitor are shown in big-endian format.


GUID Storage Format:
--------------------

In order to implement an OVMF "SDT Header Probe Suppressor", the contents of
the vmgenid_guid fw_cfg blob are not simply a 128-bit GUID.  There is also
significant padding in order to align and fill a memory page, as shown in the
following diagram::

  +----------------------------------+
  | SSDT with OEM Table ID = VMGENID |
  +----------------------------------+
  | ...                              |       TOP OF PAGE
  | VGIA dword object ---------------|-----> +---------------------------+
  | ...                              |       | fw-allocated array for    |
  | _STA method referring to VGIA    |       | "etc/vmgenid_guid"        |
  | ...                              |       +---------------------------+
  | ADDR method referring to VGIA    |       |  0: OVMF SDT Header probe |
  | ...                              |       |     suppressor            |
  +----------------------------------+       | 36: padding for 8-byte    |
                                             |     alignment             |
                                             | 40: GUID                  |
                                             | 56: padding to page size  |
                                             +---------------------------+
                                             END OF PAGE


Device Usage:
-------------

The device has one property, which may be only be set using the command line:

``guid``
  sets the value of the GUID.  A special value ``auto`` instructs
  QEMU to generate a new random GUID.

For example::

  QEMU  -device vmgenid,guid="324e6eaf-d1d1-4bf6-bf41-b9bb6c91fb87"
  QEMU  -device vmgenid,guid=auto

The property may be queried via QMP/HMP::

  (QEMU) query-vm-generation-id
  {"return": {"guid": "324e6eaf-d1d1-4bf6-bf41-b9bb6c91fb87"}}

Setting of this parameter is intentionally left out from the QMP/HMP
interfaces.  There are no known use cases for changing the GUID once QEMU is
running, and adding this capability would greatly increase the complexity.
