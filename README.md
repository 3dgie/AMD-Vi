# AMD-Vi

After tonnes of blood, tears and sweat....

I'm glad to have been working on Qemu AMD IOMMU emulation as part of GSoC and to have achieved most of what was planned, that is, both a working host translation implementation and a working interrupt remapping implementation. The code implementing this device is still under review and not yet merged into Qemu.

![](http://hotpepper.co.ke/content/images/2016/08/complete.png)

*So*, *what now??* - *nothing, really*

**IOMMU TOPOLOGY**

IOMMU is a device that sits between host bridge and peripherals allowing them to make I/O requests by-passing the need to request CPU for access permissions. This allows user-space application to take full control of devices a.k.a device pass-through and is hence an important feature for virtualization.

![](http://hotpepper.co.ke/content/images/2016/08/2016-08-02-180438_1366x768_scrot.png)

*1.2.6  Virtualizing the IOMMU*

 *The IOMMU has been designed so that it can be   emulated in  software by a VMM that wishes to present its VM guests the illusion that they have an IOMMU*   - *AMD IOMMU Specification*

This didn't turn out *so* easy.

## how come?!?

1. IOMMU masquerades as a PCI device by stealing PCI config space so that the Software driver is able to communicate with it as with     any other PCI device while it's not really a PCI device.

   _3.1 PCI Resources_ 

   _The IOMMU is implemented as an independent PCI Function. Any PCI Function containing an IOMMU capability block cannot be used for any purpose other than containing an IOMMU. A PCI Function containing an IOMMU capability block does not include PCI BAR registers. Configuration and status information for the IOMMU are mapped into PCI configuration space using a PCI capability block._"  - *AMD IOMMU Spec*

   (Someone actually pointed this out, for me ;-p)

   The above becomes evident since IOMMU reserves MMIO region without a BAR register. This feature was particularly hard to emulate in Qemu since it only supports either purely PCI devices or platform devices(system bus devices).

2. AMD IOMMU, while masquerading as a PCI device generates interrupts without being a BusMaster device(which is typical of PCI devices)

   ![](http://hotpepper.co.ke/content/images/2016/08/busmaster-3.png)

   IOMMU should generate an interrupt each time it logs an event which should include all hardware errors and page faults.

3. IOMMUs(including AMD IOMMU) use device source ID (which should be same as BDF) to index _device table_ and other informational data    structures in-order to make decisions on access rights and remap interrupts. Platform devices like HPET and IOAPIC don't make         DMArequests, at least as long as Qemu is concerned which means that the DMA part of IOMMU can be completed without any problem.       These devices are however major interrupt sources in a system which means the interrupt remapping work is a bit tricky. Interrupt     requests should be accompanied by information identifying the device and the nature of the interrupt requests. Included in this       information is the Requester ID which is basically the Bus Device Function(BDF) when PCI devices are concerned. Platform devices      which want to make interrupt requests should report the BDF that will accompany their interrupt requests. The problem comes in        because platform devices, in Qemu currently make interrupt requests using unspecified attributes meaning unspecified requester ID.

   _huh_ ? _sounds like AMD is hating on Qemu_

The working Qemu AMD IOMMU setup implements a composite PCI/Platform device similar to the stripped down device below.

```c
    #include "qemu/osdep.h"
    #include "hw/pci/pci.h"
    #include "hw/i386/x86-iommu.h"
    #include "hw/pci/msi.h"
    #include "hw/pci/pci_bus.h"
    #include "hw/sysbus.h"
    #include "qom/object.h"
    #include "hw/i386/pc.h"
    
    #define AMDVI_MMIO_SIZE        0x4000
    #define AMDVI_CAPAB_SIZE       0x18
    #define AMDVI_CAPAB_REG_SIZE   0x04
    #define AMDVI_CAPAB_ID_SEC     0xff

    #define TYPE_AMD_IOMMU_DEVICE "amd-iommu"
    #define AMD_IOMMU_DEVICE(obj)\
        OBJECT_CHECK(AMDVIState, (obj),TYPE_AMD_IOMMU_DEVICE)

    #define TYPE_AMD_IOMMU_PCI "AMDVI-PCI"
    #define AMD_IOMMU_PCI(obj)\
       OBJECT_CHECK(AMDVIPCIState, (obj), TYPE_AMD_IOMMU_PCI)

    typedef struct AMDVIPCIState {
       PCIDevice dev;               /* PCI specific properties      */
       uint32_t capab_offset;       /* capability offset pointer    */
    } AMDVIPCIState;

    typedef struct AMDVIState {
       X86IOMMUState iommu;        /* IOMMU bus device             */
       AMDVIPCIState *dev;         /* IOMMU PCI device             */

       uint8_t mmior[AMDVI_MMIO_SIZE];    /* read/write MMIO              */
       uint8_t w1cmask[AMDVI_MMIO_SIZE];  /* read/write 1 clear mask      */
       uint8_t romask[AMDVI_MMIO_SIZE];   /* MMIO read/only mask          */
    } AMDVIState;

    static void amdvi_init(AMDVIState *s)
    {
         /* reset PCI device */
         pci_config_set_vendor_id(s->dev->dev.config, PCI_VENDOR_ID_AMD);
         pci_config_set_prog_interface(s->dev->dev.config, 00);
         pci_config_set_class(s->dev->dev.config, 0x0806);
    }

    static void amdvi_reset(DeviceState *dev)
    {
        AMDVIState *s = AMD_IOMMU_DEVICE(dev);
        amdvi_init(s);
    }

    static void amdvi_realize(DeviceState *dev, Error **errp)
    {
         AMDVIState *s = AMD_IOMMU_DEVICE(dev);
         PCIBus *bus = PC_MACHINE(qdev_get_machine())->bus;

         /* This device should take care of IOMMU PCI properties */
         PCIDevice *createddev = pci_create_simple(bus, -1, TYPE_AMD_IOMMU_PCI);
         AMDVIPCIState *amdpcidevice = container_of(createddev, AMDVIPCIState, dev);
         s->dev = amdpcidevice;
    }

    static void amdvi_class_init(ObjectClass *klass, void* data)
    {
         DeviceClass *dc = DEVICE_CLASS(klass);
         X86IOMMUClass *dc_class = X86_IOMMU_CLASS(klass);

         dc->reset = amdvi_reset;
         dc_class->realize = amdvi_realize;
    }

     static const TypeInfo amdvi = {
         .name = TYPE_AMD_IOMMU_DEVICE,
         .parent = TYPE_X86_IOMMU_DEVICE,
         .instance_size = sizeof(AMDVIState),
         .class_init = amdvi_class_init
     };

     static void amdviPCI_realize(PCIDevice *dev, Error **errp)
    {
         AMDVIPCIState *s = container_of(dev, AMDVIPCIState, dev);

         /* we need to report certain PCI capabilities */
         s->capab_offset = pci_add_capability(&s->dev, AMDVI_CAPAB_ID_SEC, 0,
                                              AMDVI_CAPAB_SIZE);
         pci_add_capability(&s->dev, PCI_CAP_ID_MSI, 0, AMDVI_CAPAB_REG_SIZE);
         pci_add_capability(&s->dev, PCI_CAP_ID_HT, 0, AMDVI_CAPAB_REG_SIZE);
    } 

    static void amdviPCI_class_init(ObjectClass *klass, void* data)
    {
        PCIDeviceClass *k = PCI_DEVICE_CLASS(klass);
        k->realize = amdviPCI_realize;
    }

    static const TypeInfo amdviPCI = {
       .name = TYPE_AMD_IOMMU_PCI,
       .parent = TYPE_PCI_DEVICE,
       .instance_size = sizeof(AMDVIPCIState),
       .class_init = amdviPCI_class_init
    };

    static void amdviPCI_register_types(void)
    {
        type_register_static(&amdviPCI);
        type_register_static(&amdvi);
    }

    type_init(amdviPCI_register_types);
   
```

The above design solves two problems one being we're able to inherit from Qemu X86 IOMMU class (which is implemented as Platform device) and the other being we're able to reserve MMIO region for like any other Platform device hence avoiding the need for a BAR register.

```c
     static void amdvi_generate_msi_interrupt(AMDVIState *s)
     {
          MSIMessage msg;
          if (msi_enabled(&s->pci.dev)) {
              msg = msi_get_message(&s->pci.dev, 0);
              address_space_stl_le(&address_space_memory, msg.address, msg.data,
                         MEMTXATTRS_UNSPECIFIED, NULL);
        }
    }
```

The above code solves the lack of BusMaster capability on AMD IOMMU by writing interrupts directly to system address space

Lastly, the Platform device issue was solved by affiliating x86 MSI route, which was initially

```c

     struct MSIRouteEntry {
         PCIDevice *dev;             /* Device pointer */
         int vector;                 /* MSI/MSIX vector index */
         int virq;                   /* Virtual IRQ index */
        QLIST_ENTRY(MSIRouteEntry) list;
      };
```

with a MSI Requester ID to have
```c
     struct MSIRouteEntry {
         PCIDevice *dev;             /* Device pointer */
         uint16_t requester_id;      /* Requesting SID */
         int vector;                 /* MSI/MSIX vector index */
         int virq;                   /* Virtual IRQ index */
        QLIST_ENTRY(MSIRouteEntry) list;
      };
```
which ensures that IOAPIC triggering interrupts in split irqchip mode is able to tell the IOMMU it's SID and so would any other platform device.

##Why spend all this effort on a seemingly useless device ?

Good Question

Am impressive proposal for using Virtual IOMMU for inter-VM communication can be accessed here https://wiki.opnfv.org/display/kvm/Vm2vm+Mst

Interrupt remapping as an IOMMU feature enables interception, validation and delivery interrupts to specific CPUs. Interrupt remapping together with interrupt virtualization could bring performance improvements to guest VM but interrupt remapping alone is supposed to provide device isolation, compatibility with x2apic and could possibly be used to program a different route for delivery of an interrupt.

Device isolation allows the handing over of device to unprivileged users since with the validation of interrupts a device controlled by, say, a malicious user cannot be able to generate spurious interrupts (which is possible by just writing to MSI region) and direct them to arbitrary CPUs.This allows safe development of device drivers from userspace but could not be done with the IOMMU models with don't allow for interrupt remapping where one requires supervisor rights to passthrough devices which is not necessary with interrupt remapping.

It’s a requirement from intel that with x2apic enabled peripheral interrupts should be delivered through interrupt remapping and lastly It _could_ be possible to reroute interrupts without reconfiguring the interrupt source by modifying the Interrupt remapping table since it contains all the information relating to routing interrupts related to a certain device. This could avoid the messy process of rerouting interrupts.

_And may be something else I forgot ?_

## how can I test this device

Quickly grab the code at https://github.com/aslaq/qemu IR

    $qemu-system-x86_64 -M q35,kernel_irqchip={split|off} -device amd-iommu 
   
should start Qemu with the device emulated. 
