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
-device vfio-pci,host=9a:00.0,bus=pci.5,iommufd=iommufd0
```

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

### 4. src/qemu/qemu_command.c (lines 4774-4784)
**Purpose:** Added `iommufd` property to QEMU device command line generation

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

## Implementation Details

### Data Flow

1. **XML → Structure:**
   - XML attribute `iommufd="iommufd0"` is parsed by `virDeviceHostdevPCIDriverInfoParseXML()`
   - Value is stored in `virDeviceHostdevPCIDriverInfo.iommufd` field

2. **Structure → QEMU:**
   - During QEMU command line generation, `qemuBuildPCIHostdevDevProps()` accesses the iommufd value
   - Value is added to JSON properties with key "iommufd"
   - JSON is converted to QEMU device command line argument

3. **Structure → XML:**
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
-device vfio-pci,host=0000:9a:00.0,id=hostdev0,bus=pci.0,addr=0x5,iommufd=iommufd0
```

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
