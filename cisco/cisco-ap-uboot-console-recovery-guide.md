# Cisco Access Point U-Boot Console Recovery - Technical Command Reference

## Prerequisites

- Console cable connected to AP
- Tera Term terminal emulator
- IOS firmware image in `.bin` format (not `.tar`)
- Direct console connection (not through terminal server)

---

## Initial Console Configuration

### Tera Term Serial Port Settings

```
Setup > Serial Port
├── Port: COM# (your port)
├── Speed: 9600 baud (initial)
├── Data: 8 bit
├── Parity: None
├── Stop bits: 1 bit
└── Flow control: None
```

### Enter U-Boot Mode

Power cycle the AP and press **ESC** repeatedly during boot until you see:

```
u-boot>
```

---

## Command Sequence for Console Upload

### Step 1: Increase Baud Rate for Faster Transfer

```
u-boot> setenv baudrate 115200
u-boot> saveenv
```

**Action Required**: Console will freeze. Change Tera Term speed to 115200:
```
Setup > Serial Port > Speed: 115200
```

Press Enter multiple times to restore the prompt.

---

### Step 2: Verify Environment Variables

```
u-boot> printenv
```

Review output for flash configuration and memory addresses.

---

### Step 3: Initiate File Receive (Choose One)

**Option A - Ymodem (Recommended)**:
```
u-boot> loady 0x80060000
```

**Option B - Xmodem (Alternative)**:
```
u-boot> loadx 0x80060000
```

Console displays:
```
## Ready for binary (ymodem) download to 0x80060000 at 115200 bps...
C
```

The repeating "C" means ready to receive.

---

### Step 4: Send File via Tera Term

**For Ymodem**:
```
File > Transfer > YMODEM > Send
Select: your_image.bin
```

**For Xmodem**:
```
File > Transfer > XMODEM > Send
Select: your_image.bin
```

Wait 30-60 minutes for completion. Do not interrupt.

---

### Step 5: Verify Upload Success

Console displays upon completion:
```
## Total Size = 0x00d85a3f = 14179903 Bytes
```

Verify filesize variable:
```
u-boot> printenv filesize
```

Expected output:
```
filesize=d85a3f
```

---

### Step 6: Prepare Flash Memory

Initialize SPI flash:
```
u-boot> sf probe
```

Expected output:
```
SF: Detected with page size 256 Bytes, erase size 4 KiB, total 16 MiB
```

Erase flash partition (adjust addresses for your model):
```
u-boot> sf erase 0x100000 0x1000000
```

Where:
- `0x100000` = Flash start offset
- `0x1000000` = Size to erase (16MB)

Expected output:
```
SF: 16777216 bytes @ 0x100000 Erased: OK
```

---

### Step 7: Write Image to Flash

Copy from RAM to flash:
```
u-boot> sf write 0x80060000 0x100000 ${filesize}
```

Parameters:
- `0x80060000` = RAM address (where image was loaded)
- `0x100000` = Flash destination offset
- `${filesize}` = Size variable set during upload

Expected output:
```
SF: 14179903 bytes @ 0x100000 Written: OK
```

**Critical**: Do not power off during this operation.

---

### Step 8: Configure Boot Parameters

**Method 1 - Full Boot Command**:
```
u-boot> setenv bootcmd 'sf probe; sf read 0x80060000 0x100000 0x1000000; bootm 0x80060000'
u-boot> saveenv
```

**Method 2 - Simplified Boot Flag** (if supported):
```
u-boot> setenv boot_flash 1
u-boot> saveenv
```

Expected output:
```
Saving Environment to Flash...
SF: Detected with page size 256 Bytes, erase size 4 KiB, total 16 MiB
Erasing SPI flash...Writing to SPI flash...done
OK
```

---

### Step 9: Restore Console Speed (Optional)

```
u-boot> setenv baudrate 9600
u-boot> saveenv
```

Change Tera Term back to 9600: `Setup > Serial Port > Speed: 9600`

---

### Step 10: Boot the Access Point

**Option A - Reset AP**:
```
u-boot> reset
```

**Option B - Boot Immediately**:
```
u-boot> boot
```

---

## Verification Commands

### After AP Boots to IOS

Check IOS version:
```
AP> enable
AP# show version
```

Verify boot configuration:
```
AP# show boot
```

Expected output:
```
BOOT path-list      : flash:/ap3g2-k9w8-mx.bin
```

---

## Alternative Memory Addresses by Model

### Common RAM Load Addresses

| AP Model | RAM Address | Notes |
|----------|-------------|-------|
| Catalyst 9100 series | 0x80060000 | Standard |
| Aironet 2800/3800 | 0x80060000 | Standard |
| Older Aironet models | 0x81000000 | Check documentation |

### Common Flash Offsets

| AP Model | Flash Offset | Size |
|----------|--------------|------|
| Catalyst 9100 series | 0x100000 | 16MB-32MB |
| Aironet 2800/3800 | 0x100000 | 16MB |
| Older Aironet models | Varies | Check model specs |

---

## Troubleshooting Commands

### Check Memory Contents

Verify data loaded to RAM:
```
u-boot> md 0x80060000
```

### Display Flash Information

```
u-boot> sf probe
u-boot> flinfo
```

### View All Environment Variables

```
u-boot> printenv
```

### Reset Environment to Defaults

```
u-boot> env default -a
u-boot> saveenv
```

### Manual Boot Test

```
u-boot> bootm 0x80060000
```

---

## Quick Reference Command Summary

```
# Increase speed
setenv baudrate 115200
saveenv

# Load image to RAM (Ymodem)
loady 0x80060000

# [Send file via Tera Term: File > Transfer > YMODEM > Send]

# Verify upload
printenv filesize

# Write to flash
sf probe
sf erase 0x100000 0x1000000
sf write 0x80060000 0x100000 ${filesize}

# Configure boot
setenv bootcmd 'sf probe; sf read 0x80060000 0x100000 0x1000000; bootm 0x80060000'
saveenv

# Reboot
reset
```

---

## Important Technical Notes

- **Memory Address `0x80060000`**: Typical SDRAM location for temporary image storage during transfer
- **Flash Offset `0x100000`**: Standard starting position in SPI flash for firmware images
- **Variable `${filesize}`**: Automatically set by `loady`/`loadx` commands, contains uploaded file size in hex
- **Command `sf probe`**: Initializes Serial Flash interface and detects flash chip parameters
- **Command `bootm`**: Boots a Linux kernel image from memory address
- **Ymodem vs Xmodem**: Ymodem uses 1024-byte blocks vs Xmodem's 128-byte blocks, resulting in 2-3x faster transfers

---

## Protocol-Specific Commands

### Ymodem Commands

```
loady [loadAddress]           # Load binary file
loady 0x80060000             # Load to specific address
loady 0x80060000 115200      # Load with specific baud
```

### Xmodem Commands

```
loadx [loadAddress]           # Load binary file
loadx 0x80060000             # Load to specific address
loadx 0x80060000 115200      # Load with specific baud
```

### Kermit Commands (Rarely Used)

```
loadb [loadAddress]           # Load via Kermit protocol
```

---

## Transfer Performance Specifications

| Baud Rate | Protocol | Image Size | Expected Duration |
|-----------|----------|------------|-------------------|
| 9600 | Xmodem | 15 MB | 4-6 hours |
| 9600 | Ymodem | 15 MB | 3-4 hours |
| 57600 | Xmodem | 15 MB | 45-60 minutes |
| 57600 | Ymodem | 15 MB | 35-45 minutes |
| 115200 | Xmodem | 15 MB | 35-45 minutes |
| 115200 | Ymodem | 15 MB | 30-40 minutes |

**Note**: Actual throughput is approximately 35-40% of baud rate due to protocol overhead and error checking.

---

## Common Troubleshooting Scenarios

### Console Hangs After Baud Rate Change

**Symptom**: No response after setting baudrate to 115200

**Resolution**:
1. Disconnect Tera Term session
2. Reconnect with updated speed (115200 baud)
3. Press Enter multiple times to restore prompt

---

### Transfer Fails with "NAK on sector" Errors

**Symptom**: File transfer repeatedly fails or corrupts

**Resolution**:
1. Verify correct protocol selection (Ymodem vs Xmodem)
2. Ensure image file is not corrupted (verify MD5 checksum)
3. Try reducing baud rate to 57600
4. Confirm direct console connection

---

### AP Returns to U-Boot After Reset

**Symptom**: AP boots to u-boot> prompt instead of loading IOS

**Resolution**:
1. Verify boot parameters: `u-boot> printenv bootcmd`
2. Confirm image was written successfully to flash
3. Manually boot: `u-boot> bootm 0x80060000`
4. Reset boot configuration using Step 8 commands

---

### Flash Write Fails

**Symptom**: Error during `sf write` command

**Resolution**:
1. Verify sufficient flash space: `u-boot> sf probe`
2. Confirm flash erase completed successfully
3. Check filesize variable: `u-boot> printenv filesize`
4. Verify RAM contains data: `u-boot> md 0x80060000`

---

## Model-Specific Considerations

### Catalyst 9100 Series (9115, 9120, 9130)

- Use `boardinit` command for certain image types
- May require specific flash addresses
- Support both U-Boot and alternate boot methods

### Aironet 2800/3800 Series

- Traditional tar extraction not applicable for console upload
- Use `.bin` image files extracted from tar archives
- May use different flash memory addresses than Catalyst series

### Older Aironet Models (1140, 1260, 2600)

- Some models use `ap:` prompt instead of `u-boot>`
- Different command syntax for older bootloaders
- Verify compatibility before attempting recovery

---

## Best Practices

### Before Starting Recovery

1. Document current AP configuration and version
2. Verify image compatibility with wireless controller
3. Ensure uninterrupted power supply during process
4. Test TFTP recovery first if network available
5. Have correct image file ready and verified

### During Recovery Process

1. Never power cycle during flash write operations
2. Monitor console output for error messages
3. Do not interrupt serial transfer in progress
4. Keep detailed notes of commands used

### After Recovery Complete

1. Verify IOS version and boot configuration
2. Test basic AP functionality before deployment
3. Update documentation with new firmware version
4. Configure AP for production environment
5. Test wireless connectivity thoroughly

---

## Security Considerations

- Always verify image MD5 checksum before upload
- Download firmware only from official Cisco sources
- Maintain image version compatibility with controller
- Document all firmware changes
- Restrict physical access to console ports

---

## Revision Information

This documentation reflects procedures for Cisco access points using U-Boot bootloader. Specific command syntax and memory addresses may vary by model and firmware version. Always consult official Cisco documentation for your specific AP model before proceeding with recovery operations.

---

*Document Version: 1.0*  
*Last Updated: October 2025*  
*Applicable Models: Catalyst 9100 Series, Aironet 2800/3800 Series*
