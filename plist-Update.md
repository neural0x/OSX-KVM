# OpenCore plist Update

## Steps to Update OpenCore Configuration

### 1. Load NBD Module
```bash
sudo modprobe nbd max_part=8
```

### 2. Connect QCOW2 Image to NBD Device
```bash
sudo qemu-nbd --connect=/dev/nbd0 OpenCore/OpenCore.qcow2
```

### 3. Create Mount Point
```bash
mkdir -p /tmp/opencore_efi
```

### 4. Mount EFI Partition
```bash
sudo mount /dev/nbd0p1 /tmp/opencore_efi
```

### 5. Copy Updated Configuration
```bash
sudo cp OpenCore/config.plist /tmp/opencore_efi/EFI/OC/config.plist
```

### 6. Verify Configuration Update
```bash
cat /tmp/opencore_efi/EFI/OC/config.plist | grep -i iMacPro1,1
```

### 7. Unmount EFI Partition
```bash
sudo umount /tmp/opencore_efi
```

### 8. Disconnect NBD Device
```bash
sudo qemu-nbd --disconnect /dev/nbd0
```

### 9. Clean Up Mount Point
```bash
rmdir /tmp/opencore_efi
```