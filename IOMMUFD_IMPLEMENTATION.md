# IOMMUFD Support Implementation for libvirt

This document describes the implementation of `iommufd` attribute support for the `<hostdev>` tag in libvirt domain XML.

## Overview

This implementation allows users to specify an `iommufd` parameter in the hostdev driver configuration, which will be passed through to the QEMU command line for vfio-pci devices.

## XML Syntax

```xml
<hostdev mode='subsystem' type='pci' managed='yes'>
  <driver name='vfio' iommufd='iommufd0'/>
  <source>
    <address domain='0x0000' bus='0x9a' slot='0x00' function='0x0'/>
  </source>
</hostdev>
```

## QEMU Command Line Output

The above XML will generate a QEMU command line similar to:

```
-object iommufd,id=iommufd0
-device vfio-pci,host=9a:00.0,bus=pci.5,iommufd=iommufd0
```

**Important:** The `-object iommufd,id=iommufd0` is automatically generated before the device that uses it. If multiple devices reference the same iommufd object, the object is only created once.

## Files Modified

### 1. src/conf/device_conf.h (lines 47-51)
**Purpose:** Added `iommufd` field to the driver info structure

```c
struct _virDeviceHostdevPCIDriverInfo {
    virDeviceHostdevPCIDriverName name;
    char *model;
    char *iommufd;  // NEW FIELD
};
```

### 2. src/conf/schemas/basictypes.rng (lines 660-683)
**Purpose:** Updated XML schema to allow the `iommufd` attribute

```xml
<define name="hostdevDriver">
  <element name="driver">
    <optional>
      <attribute name="name">
        <choice>
          <value>kvm</value>
          <value>vfio</value>
          <value>xen</value>
        </choice>
      </attribute>
    </optional>
    <optional>
      <attribute name="model">
        <ref name="genericName"/>
      </attribute>
    </optional>
    <optional>
      <attribute name="iommufd">  <!-- NEW ATTRIBUTE -->
        <ref name="genericName"/>
      </attribute>
    </optional>
    <empty/>
  </element>
</define>
```

### 3. src/conf/device_conf.c

**a) XML Parsing (lines 59-73):**
Added parsing for the `iommufd` attribute from XML:

```c
int
virDeviceHostdevPCIDriverInfoParseXML(xmlNodePtr node,
                                      virDeviceHostdevPCIDriverInfo *driver)
{
    if (virXMLPropEnum(node, "name",
                       virDeviceHostdevPCIDriverNameTypeFromString,
                       VIR_XML_PROP_NONZERO,
                       &driver->name) < 0) {
        return -1;
    }

    driver->model = virXMLPropString(node, "model");
    driver->iommufd = virXMLPropString(node, "iommufd");  // NEW LINE
    return 0;
}
```

**b) XML Formatting (lines 76-100):**
Added formatting of the `iommufd` attribute back to XML:

```c
int
virDeviceHostdevPCIDriverInfoFormat(virBuffer *buf,
                                    const virDeviceHostdevPCIDriverInfo *driver)
{
    g_auto(virBuffer) driverAttrBuf = VIR_BUFFER_INITIALIZER;

    if (driver->name != VIR_DEVICE_HOSTDEV_PCI_DRIVER_NAME_DEFAULT) {
        const char *driverName = virDeviceHostdevPCIDriverNameTypeToString(driver->name);

        if (!driverName) {
            virReportError(VIR_ERR_INTERNAL_ERROR,
                           _("unexpected pci hostdev driver name %1$d"),
                           driver->name);
            return -1;
        }

        virBufferAsprintf(&driverAttrBuf, " name='%s'", driverName);
    }

    virBufferEscapeString(&driverAttrBuf, " model='%s'", driver->model);
    virBufferEscapeString(&driverAttrBuf, " iommufd='%s'", driver->iommufd);  // NEW LINE

    virXMLFormatElement(buf, "driver", &driverAttrBuf, NULL);
    return 0;
}
```

**c) Memory Cleanup (lines 103-108):**
Added cleanup for the `iommufd` string:

```c
void
virDeviceHostdevPCIDriverInfoClear(virDeviceHostdevPCIDriverInfo *driver)
{
    VIR_FREE(driver->model);
    VIR_FREE(driver->iommufd);  // NEW LINE
}
```

### 4. src/qemu/qemu_domain.h (line 263)
**Purpose:** Added hash table to track iommufd objects and prevent duplicates

```c
GSList *threadContextAliases; /* List of IDs of thread-context objects */
GHashTable *iommufdObjects; /* Hash table of iommufd object IDs (key=id, value=boolean) */  // NEW LINE
```

### 5. src/qemu/qemu_domain.c

**a) Initialization (line 2005):**
Initialize the iommufd hash table when creating domain private data:

```c
priv->blockjobs = virHashNew(virObjectUnref);
priv->fds = virHashNew(g_object_unref);
priv->iommufdObjects = virHashNew(NULL);  // NEW LINE
```

**b) Cleanup (line 1977):**
Free the iommufd hash table when destroying domain private data:

```c
g_clear_pointer(&priv->blockjobs, g_hash_table_unref);
g_clear_pointer(&priv->fds, g_hash_table_unref);
g_clear_pointer(&priv->iommufdObjects, g_hash_table_unref);  // NEW LINE
```

### 6. src/qemu/qemu_command.c

**a) Device Property Addition (lines 4774-4784):**
Added `iommufd` property to QEMU device command line generation:

```c
if (virJSONValueObjectAdd(&props,
                          "s:driver", driver,
                          "s:host", host,
                          "s:id", dev->info->alias,
                          "p:bootindex", dev->info->effectiveBootIndex,
                          "S:failover_pair_id", failover_pair_id,
                          "S:display", qemuOnOffAuto(pcisrc->display),
                          "B:ramfb", ramfb,
                          "S:iommufd", pcisrc->driver.iommufd,  // NEW LINE
                          NULL) < 0)
    return NULL;
```

**b) IOMMUFD Object Generation (lines 5193-5237):**
New function to generate `-object iommufd` command line arguments:

```c
static int
qemuBuildIOMMUFDCommandLine(virCommand *cmd,
                            const virDomainDef *def,
                            virDomainObj *vm)
{
    qemuDomainObjPrivate *priv = QEMU_DOMAIN_PRIVATE(vm);
    size_t i;

    for (i = 0; i < def->nhostdevs; i++) {
        virDomainHostdevDef *hostdev = def->hostdevs[i];
        virDomainHostdevSubsys *subsys = &hostdev->source.subsys;
        const char *iommufd = NULL;
        g_autoptr(virJSONValue) props = NULL;

        if (hostdev->mode != VIR_DOMAIN_HOSTDEV_MODE_SUBSYS)
            continue;

        if (subsys->type != VIR_DOMAIN_HOSTDEV_SUBSYS_TYPE_PCI)
            continue;

        iommufd = subsys->u.pci.driver.iommufd;
        if (!iommufd)
            continue;

        /* Check if this iommufd object was already added */
        if (virHashHasEntry(priv->iommufdObjects, iommufd))
            continue;

        /* Create the iommufd object */
        if (virJSONValueObjectAdd(&props,
                                  "s:qom-type", "iommufd",
                                  "s:id", iommufd,
                                  NULL) < 0)
            return -1;

        if (qemuBuildObjectCommandlineFromJSON(cmd, props) < 0)
            return -1;

        /* Mark this iommufd as added */
        if (virHashAddEntry(priv->iommufdObjects, iommufd, (void *)0x1) < 0)
            return -1;
    }

    return 0;
}
```

**c) Function Call (line 10925):**
Call the iommufd object builder before building hostdev devices:

```c
if (qemuBuildRedirdevCommandLine(cmd, def, qemuCaps) < 0)
    return NULL;

if (qemuBuildIOMMUFDCommandLine(cmd, def, vm) < 0)  // NEW LINE
    return NULL;

if (qemuBuildHostdevCommandLine(cmd, def, qemuCaps) < 0)
    return NULL;
```

## Implementation Details

### Data Flow

1. **XML → Structure:**
   - XML attribute `iommufd="iommufd0"` is parsed by `virDeviceHostdevPCIDriverInfoParseXML()`
   - Value is stored in `virDeviceHostdevPCIDriverInfo.iommufd` field

2. **Structure → QEMU Objects:**
   - During QEMU command line generation, `qemuBuildIOMMUFDCommandLine()` is called
   - Function iterates through all hostdevs looking for PCI devices with iommufd specified
   - For each unique iommufd ID, creates a `-object iommufd,id=<id>` argument
   - Uses hash table to track which iommufd objects have been created to avoid duplicates

3. **Structure → QEMU Device:**
   - After objects are created, `qemuBuildPCIHostdevDevProps()` accesses the iommufd value
   - Value is added to device JSON properties with key "iommufd"
   - JSON is converted to QEMU device command line argument: `-device vfio-pci,...,iommufd=<id>`

4. **Structure → XML:**
   - When dumping domain XML, `virDeviceHostdevPCIDriverInfoFormat()` writes the iommufd attribute back

### Memory Management

- The `iommufd` field is a dynamically allocated string (`char *`)
- Memory is allocated by `virXMLPropString()` during parsing
- Memory is freed by `VIR_FREE()` in `virDeviceHostdevPCIDriverInfoClear()`
- The field is properly escaped when formatted to XML using `virBufferEscapeString()`

### JSON Property Type

The `iommufd` property uses the `"S:"` prefix in `virJSONValueObjectAdd()`, which means:
- **S** = String (optional) - the property will only be added to JSON if the string is non-NULL
- This prevents adding `"iommufd":null` to the QEMU command line when the attribute is not specified

### Deduplication Mechanism

To prevent multiple `-object iommufd` declarations for the same ID:

1. **Hash Table Tracking:**
   - A hash table `priv->iommufdObjects` is maintained in the domain private data
   - Key: iommufd ID string (e.g., "iommufd0")
   - Value: A dummy pointer (0x1) indicating the object was created

2. **Deduplication Process:**
   - When building the command line, `qemuBuildIOMMUFDCommandLine()` iterates through all hostdevs
   - For each hostdev with an iommufd attribute:
     - Check if the ID already exists in the hash table using `virHashHasEntry()`
     - If not found, create the `-object iommufd,id=<id>` argument and add the ID to the hash table
     - If found, skip object creation (it was already added by a previous hostdev)

3. **Example with Multiple Devices:**
   ```xml
   <hostdev mode='subsystem' type='pci'>
     <driver name='vfio' iommufd='iommufd0'/>
     <source><address domain='0x0000' bus='0x9a' slot='0x00' function='0x0'/></source>
   </hostdev>
   <hostdev mode='subsystem' type='pci'>
     <driver name='vfio' iommufd='iommufd0'/>
     <source><address domain='0x0000' bus='0x9a' slot='0x00' function='0x1'/></source>
   </hostdev>
   ```

   Results in:
   ```
   -object iommufd,id=iommufd0          (created once)
   -device vfio-pci,host=9a:00.0,iommufd=iommufd0
   -device vfio-pci,host=9a:00.1,iommufd=iommufd0
   ```

## Usage Example

### Complete Domain XML Example

See [example_iommufd_hostdev.xml](example_iommufd_hostdev.xml) for a complete working example.

```xml
<domain type='kvm'>
  <name>test-domain</name>
  <memory unit='KiB'>2097152</memory>
  <vcpu placement='static'>2</vcpu>
  <os>
    <type arch='x86_64' machine='pc'>hvm</type>
  </os>
  <devices>
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <driver name='vfio' iommufd='iommufd0'/>
      <source>
        <address domain='0x0000' bus='0x9a' slot='0x00' function='0x0'/>
      </source>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
    </hostdev>
  </devices>
</domain>
```

### Expected QEMU Command Line Fragment

```
-object iommufd,id=iommufd0
-device vfio-pci,host=0000:9a:00.0,id=hostdev0,bus=pci.0,addr=0x5,iommufd=iommufd0
```

Note: The `-object` line is automatically generated before the device.

## Testing

To test this implementation:

1. **Build libvirt:**
   ```bash
   meson setup build
   ninja -C build
   ```

2. **Define a domain with the new attribute:**
   ```bash
   virsh define example_iommufd_hostdev.xml
   ```

3. **Verify the XML is preserved:**
   ```bash
   virsh dumpxml test-domain
   ```

4. **Check the QEMU command line:**
   ```bash
   ps aux | grep qemu
   # Look for the iommufd parameter in the vfio-pci device arguments
   ```

## Compatibility

- This change is backward compatible - the `iommufd` attribute is optional
- Existing domain XMLs without the attribute will continue to work
- The attribute is only meaningful for VFIO driver (driver name='vfio')
- Requires QEMU with iommufd support

## References

- QEMU VFIO documentation on iommufd
- libvirt domain XML format documentation
- RelaxNG schema validation
