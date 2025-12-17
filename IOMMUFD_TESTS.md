# IOMMUFD Tests

This document describes the test cases added for the iommufd functionality.

## Test Files Created

### Test 1: Single Device with IOMMUFD

**Input XML**: [tests/qemuxmlconfdata/hostdev-pci-iommufd.xml](tests/qemuxmlconfdata/hostdev-pci-iommufd.xml)

Tests a single PCI hostdev with an iommufd attribute.

```xml
<hostdev mode='subsystem' type='pci' managed='yes'>
  <driver name='vfio' iommufd='iommufd0'/>
  <source>
    <address domain='0x0000' bus='0x06' slot='0x12' function='0x5'/>
  </source>
</hostdev>
```

**Expected QEMU Command Line**: [tests/qemuxmlconfdata/hostdev-pci-iommufd.x86_64-latest.args](tests/qemuxmlconfdata/hostdev-pci-iommufd.x86_64-latest.args)

Key generated arguments:
```
-object '{"qom-type":"iommufd","id":"iommufd0"}'
-device '{"driver":"vfio-pci","host":"0000:06:12.5","id":"hostdev0","bus":"pci.0","addr":"0x2","iommufd":"iommufd0"}'
```

**Normalized Output XML**: [tests/qemuxmlconfdata/hostdev-pci-iommufd.x86_64-latest.xml](tests/qemuxmlconfdata/hostdev-pci-iommufd.x86_64-latest.xml)

Includes auto-assigned PCI guest address.

### Test 2: Multiple Devices with IOMMUFD (Deduplication)

**Input XML**: [tests/qemuxmlconfdata/hostdev-pci-iommufd-multiple.xml](tests/qemuxmlconfdata/hostdev-pci-iommufd-multiple.xml)

Tests three PCI hostdevs:
- Two devices sharing `iommufd0` (to test deduplication)
- One device using `iommufd1`

```xml
<hostdev mode='subsystem' type='pci' managed='yes'>
  <driver name='vfio' iommufd='iommufd0'/>
  <source>
    <address domain='0x0000' bus='0x06' slot='0x12' function='0x0'/>
  </source>
</hostdev>
<hostdev mode='subsystem' type='pci' managed='yes'>
  <driver name='vfio' iommufd='iommufd0'/>
  <source>
    <address domain='0x0000' bus='0x06' slot='0x12' function='0x1'/>
  </source>
</hostdev>
<hostdev mode='subsystem' type='pci' managed='yes'>
  <driver name='vfio' iommufd='iommufd1'/>
  <source>
    <address domain='0x0000' bus='0x07' slot='0x00' function='0x0'/>
  </source>
</hostdev>
```

**Expected QEMU Command Line**: [tests/qemuxmlconfdata/hostdev-pci-iommufd-multiple.x86_64-latest.args](tests/qemuxmlconfdata/hostdev-pci-iommufd-multiple.x86_64-latest.args)

Key generated arguments (note: only 2 iommufd objects for 3 devices):
```
-object '{"qom-type":"iommufd","id":"iommufd0"}'
-object '{"qom-type":"iommufd","id":"iommufd1"}'
-device '{"driver":"vfio-pci","host":"0000:06:12.0","id":"hostdev0","bus":"pci.0","addr":"0x2","iommufd":"iommufd0"}'
-device '{"driver":"vfio-pci","host":"0000:06:12.1","id":"hostdev1","bus":"pci.0","addr":"0x3","iommufd":"iommufd0"}'
-device '{"driver":"vfio-pci","host":"0000:07:00.0","id":"hostdev2","bus":"pci.0","addr":"0x4","iommufd":"iommufd1"}'
```

**Normalized Output XML**: [tests/qemuxmlconfdata/hostdev-pci-iommufd-multiple.x86_64-latest.xml](tests/qemuxmlconfdata/hostdev-pci-iommufd-multiple.x86_64-latest.xml)

## Test Registration

Tests are registered in [tests/qemuxmlconftest.c](tests/qemuxmlconftest.c) at lines 2349-2350:

```c
DO_TEST_CAPS_LATEST("hostdev-pci-iommufd");
DO_TEST_CAPS_LATEST("hostdev-pci-iommufd-multiple");
```

## What the Tests Verify

### Test 1: hostdev-pci-iommufd
1. **XML Parsing**: Correctly parses `iommufd` attribute from driver element
2. **Object Generation**: Generates `-object iommufd,id=iommufd0` before device
3. **Device Property**: Adds `iommufd` property to vfio-pci device
4. **XML Formatting**: Preserves `iommufd` attribute when dumping XML

### Test 2: hostdev-pci-iommufd-multiple
1. **Deduplication**: Only creates one iommufd object for `iommufd0` despite two devices using it
2. **Multiple Objects**: Correctly creates separate objects for different IDs (`iommufd0` and `iommufd1`)
3. **Correct References**: Each device references the correct iommufd ID
4. **Order Independence**: Object creation happens before all devices regardless of declaration order

## Running the Tests

```bash
# Build libvirt
meson setup build
ninja -C build

# Run just the XML configuration tests
cd build
meson test qemuxmlconftest

# Run with verbose output
meson test qemuxmlconftest --verbose

# Regenerate expected output (if implementation changes)
VIR_TEST_REGENERATE_OUTPUT=1 meson test qemuxmlconftest
```

## Test Coverage

These tests cover:
- ✅ Single device with iommufd
- ✅ Multiple devices sharing same iommufd (deduplication)
- ✅ Multiple devices with different iommufd IDs
- ✅ XML parsing and formatting
- ✅ QEMU command line generation
- ✅ PCI address auto-assignment

## Future Test Ideas

Additional tests that could be added:
- Mixed hostdevs (some with iommufd, some without)
- Error cases (invalid iommufd ID format)
- Other architectures (aarch64, s390x with zpci)
- Hotplug scenarios (runtime device attach/detach)
- Migration compatibility
