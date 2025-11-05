# FreePBX Survey IVR Documentation

## Overview
This documentation describes the complete implementation of a survey IVR system integrated with FreePBX and Avaya. The system collects customer satisfaction ratings (1-5) after calls are transferred from Avaya to FreePBX extension 3459.

## System Architecture

### Call Flow
1. Customer calls FreePBX extension 3477 (Grandstream)
2. Extension 3477 calls Avaya extension 3868
3. Avaya agent handles the call
4. Avaya transfers the call to FreePBX extension 3459 (Survey IVR)
5. Customer hears survey prompt and presses a digit (1-5)
6. Response is saved to CSV file with timestamp and caller information

### Components
- **FreePBX**: VoIP PBX system running Asterisk
- **Avaya**: Legacy PBX system connected via H.323 protocol
- **OOH323**: Asterisk H.323 channel driver for Avaya connectivity
- **Survey IVR**: Custom Asterisk dialplan for DTMF collection
- **Python AGI Script**: Data processing and CSV storage

---

## Configuration Files

### 1. H.323 Trunk Configuration

**File Location:** `/etc/asterisk/ooh323.conf`

**File Permissions:**
```bash
-rw-r--r-- 1 asterisk asterisk /etc/asterisk/ooh323.conf
```

**Configuration:**
```ini
[general]
port=1720
bindaddr=0.0.0.0
disallow=all
allow=alaw
allow=ulaw
dtmfmode=inband
relaxdtmf=yes
faststart=yes
h245tunneling=yes
logfile=/var/log/asterisk/ooh323_log
loglevel=5

[Avaya]
type=friend
context=from-internal
host=10.23.48.15         ; Avaya IP - DO NOT CHANGE
port=1720
disallow=all
allow=alaw
allow=ulaw
dtmfmode=inband
relaxdtmf=yes
faststart=yes
h245tunneling=yes
```

**Configuration Notes:**
- `host=10.23.48.15`: Avaya PBX IP address (must not be changed)
- `dtmfmode=inband`: DTMF detection mode for proper tone recognition
- `relaxdtmf=yes`: Relaxed DTMF detection for better compatibility
- `context=from-internal`: Incoming calls from Avaya route to this context

**Apply Changes:**
```bash
asterisk -rx "core reload"
```

---

### 2. Survey IVR Dialplan

**File Location:** `/etc/asterisk/extensions_custom.conf`

**File Permissions:**
```bash
-rw-r--r-- 1 asterisk asterisk /etc/asterisk/extensions_custom.conf
```

**Configuration:**
```ini
[survey-ivr]
exten => s,1,NoOp(===== Survey IVR Starting =====)
 same => n,Set(SURVEY_CALLER=${CALLERID(num)})
 same => n,Answer()
 same => n,Wait(1)
 same => n,Set(TIMEOUT(digit)=5)
 same => n,Set(TIMEOUT(response)=10)
 same => n,Background(custom/luvvoice-com-20251104-Tim51Q)
 same => n,WaitExten(10)
 same => n,Playback(vm-goodbye)
 same => n,Hangup()

; When digit 1 is pressed
exten => 1,1,Goto(custom-save-digit1,s,1)

; When digit 2 is pressed
exten => 2,1,Goto(custom-save-digit2,s,1)

; When digit 3 is pressed
exten => 3,1,Goto(custom-save-digit3,s,1)

; When digit 4 is pressed
exten => 4,1,Goto(custom-save-digit4,s,1)

; When digit 5 is pressed
exten => 5,1,Goto(custom-save-digit5,s,1)

; Invalid input
exten => i,1,Playback(invalid)
 same => n,Goto(s,1)

; Timeout
exten => t,1,Playback(vm-goodbye)
 same => n,Hangup()

exten => 3459,1,Goto(survey-ivr,s,1)

[custom-save-digit1]
exten => s,1,NoOp(Saving digit 1 from ${CALLERID(num)})
 same => n,AGI(save_input.py,${CALLERID(num)},1)
 same => n,Playback(thank-you-for-calling)
 same => n,Hangup()

[custom-save-digit2]
exten => s,1,NoOp(Saving digit 2 from ${CALLERID(num)})
 same => n,AGI(save_input.py,${CALLERID(num)},2)
 same => n,Playback(thank-you-for-calling)
 same => n,Hangup()

[custom-save-digit3]
exten => s,1,NoOp(Saving digit 3 from ${CALLERID(num)})
 same => n,AGI(save_input.py,${CALLERID(num)},3)
 same => n,Playback(thank-you-for-calling)
 same => n,Hangup()

[custom-save-digit4]
exten => s,1,NoOp(Saving digit 4 from ${CALLERID(num)})
 same => n,AGI(save_input.py,${CALLERID(num)},4)
 same => n,Playback(thank-you-for-calling)
 same => n,Hangup()

[custom-save-digit5]
exten => s,1,NoOp(Saving digit 5 from ${CALLERID(num)})
 same => n,AGI(save_input.py,${CALLERID(num)},5)
 same => n,Playback(thank-you-for-calling)
 same => n,Hangup()

[from-internal-custom]
exten => 3459,1,Goto(survey-ivr,s,1)
```

**Dialplan Notes:**
- Extension 3459 is the survey IVR entry point
- `Background()` plays audio while accepting DTMF input
- `WaitExten(10)` waits up to 10 seconds for digit input
- Digit timeout: 5 seconds between digits
- Response timeout: 10 seconds total
- Audio file: `custom/luvvoice-com-20251104-Tim51Q` (survey prompt)

**Apply Changes:**
```bash
asterisk -rx "dialplan reload"
```

---

### 3. AGI Data Collection Script

**File Location:** `/var/lib/asterisk/agi-bin/save_input.py`

**File Permissions:**
```bash
-rwxr-xr-x 1 asterisk asterisk /var/lib/asterisk/agi-bin/save_input.py
```

**Set Permissions:**
```bash
sudo chmod +x /var/lib/asterisk/agi-bin/save_input.py
sudo chown asterisk:asterisk /var/lib/asterisk/agi-bin/save_input.py
```

**Script Content:**
```python
#!/usr/bin/python
import sys
import datetime
import csv
import os

# Direct log file for debugging
LOG_FILE = '/var/log/asterisk/save_input_debug.log'

def agi_output(msg):
    sys.stdout.write(msg + '\n')
    sys.stdout.flush()

def agi_read():
    try:
        line = sys.stdin.readline().strip()
        return line
    except:
        return ""

def log_message(msg):
    sys.stderr.write("[save_input.py] " + msg + '\n')
    sys.stderr.flush()
    # Also write to file
    try:
        with open(LOG_FILE, 'a') as f:
            f.write("[{0}] {1}\n".format(datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S'), msg))
    except:
        pass

try:
    log_message("=== Script starting ===")

    # Read AGI environment (will be empty lines when done)
    agi_env = {}
    while True:
        line = agi_read()
        if not line or line == "":
            break
        if ':' in line:
            key, value = line.split(':', 1)
            agi_env[key.strip()] = value.strip()

    log_message("AGI environment read")

    # Get arguments passed from dialplan
    caller_id = sys.argv[1] if len(sys.argv) > 1 else 'UNKNOWN'
    survey_input = sys.argv[2] if len(sys.argv) > 2 else 'NONE'

    log_message("Arguments: caller_id={0}, survey_input={1}".format(caller_id, survey_input))

    # Prepare data to save
    timestamp = datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    csv_file = '/var/log/asterisk/survey_responses.csv'

    log_message("CSV file: {0}".format(csv_file))

    # Create CSV file with headers if it doesn't exist
    if not os.path.exists(csv_file):
        log_message("Creating new CSV file")
        with open(csv_file, 'w') as f:
            writer = csv.writer(f)
            writer.writerow(['Timestamp', 'Caller_ID', 'Survey_Input'])
        os.chmod(csv_file, 0o664)
        log_message("CSV file created")

    # Append the survey response
    log_message("Opening CSV file for append")
    with open(csv_file, 'a') as f:
        writer = csv.writer(f)
        writer.writerow([timestamp, caller_id, survey_input])
        f.flush()
        os.fsync(f.fileno())

    log_message("Data written: {0}, {1}, {2}".format(timestamp, caller_id, survey_input))
    agi_output('VERBOSE "Survey response saved: caller={0} input={1}" 1'.format(caller_id, survey_input))

    log_message("=== Script completed successfully ===")
    sys.exit(0)

except Exception as e:
    log_message("ERROR: {0}".format(str(e)))
    import traceback
    log_message("Traceback: {0}".format(traceback.format_exc()))
    agi_output('VERBOSE "Error saving survey: {0}" 1'.format(str(e)))
    sys.exit(1)
```

**Script Notes:**
- Shebang: `#!/usr/bin/python` (uses system Python, typically Python 2.7 or 3.6)
- Arguments: `caller_id` (from ${CALLERID(num)}) and `survey_input` (digit 1-5)
- Output: CSV file with timestamp, caller ID, and survey response
- Logging: Writes debug information to `/var/log/asterisk/save_input_debug.log`

---

## Output Files

### Survey Response Data

**File Location:** `/var/log/asterisk/survey_responses.csv`

**File Permissions:**
```bash
-rw-rw-r-- 1 asterisk asterisk /var/log/asterisk/survey_responses.csv
```

**Set Permissions:**
```bash
sudo touch /var/log/asterisk/survey_responses.csv
sudo chown asterisk:asterisk /var/log/asterisk/survey_responses.csv
sudo chmod 664 /var/log/asterisk/survey_responses.csv
```

**CSV Format:**
```csv
Timestamp,Caller_ID,Survey_Input
2025-11-05 15:40:00,3477,3
2025-11-05 15:42:15,1234567890,5
```

**Columns:**
- `Timestamp`: Date and time of survey response (YYYY-MM-DD HH:MM:SS)
- `Caller_ID`: Caller's phone number (from CALLERID)
- `Survey_Input`: Digit pressed by caller (1-5)

---

### Debug Log

**File Location:** `/var/log/asterisk/save_input_debug.log`

**File Permissions:**
```bash
-rw-rw-rw- 1 asterisk asterisk /var/log/asterisk/save_input_debug.log
```

**Set Permissions:**
```bash
sudo touch /var/log/asterisk/save_input_debug.log
sudo chown asterisk:asterisk /var/log/asterisk/save_input_debug.log
sudo chmod 666 /var/log/asterisk/save_input_debug.log
```

**Log Format:**
```
[2025-11-05 15:40:00] === Script starting ===
[2025-11-05 15:40:00] AGI environment read
[2025-11-05 15:40:00] Arguments: caller_id=3477, survey_input=3
[2025-11-05 15:40:00] CSV file: /var/log/asterisk/survey_responses.csv
[2025-11-05 15:40:00] Opening CSV file for append
[2025-11-05 15:40:00] Data written: 2025-11-05 15:40:00, 3477, 3
[2025-11-05 15:40:00] === Script completed successfully ===
```

---

## Installation Instructions

### Step 1: Configure H.323 Trunk
```bash
sudo nano /etc/asterisk/ooh323.conf
# Copy and paste the ooh323.conf configuration above
sudo chown asterisk:asterisk /etc/asterisk/ooh323.conf
sudo chmod 644 /etc/asterisk/ooh323.conf
asterisk -rx "module reload chan_ooh323.so"
```

### Step 2: Configure Survey Dialplan
```bash
sudo nano /etc/asterisk/extensions_custom.conf
# Copy and paste the extensions_custom.conf configuration above
sudo chown asterisk:asterisk /etc/asterisk/extensions_custom.conf
sudo chmod 644 /etc/asterisk/extensions_custom.conf
asterisk -rx "dialplan reload"
```

### Step 3: Install AGI Script
```bash
sudo nano /var/lib/asterisk/agi-bin/save_input.py
# Copy and paste the save_input.py script above
sudo chmod +x /var/lib/asterisk/agi-bin/save_input.py
sudo chown asterisk:asterisk /var/lib/asterisk/agi-bin/save_input.py
```

### Step 4: Create Output Files
```bash
# Create CSV file
sudo touch /var/log/asterisk/survey_responses.csv
sudo chown asterisk:asterisk /var/log/asterisk/survey_responses.csv
sudo chmod 664 /var/log/asterisk/survey_responses.csv

# Create debug log
sudo touch /var/log/asterisk/save_input_debug.log
sudo chown asterisk:asterisk /var/log/asterisk/save_input_debug.log
sudo chmod 666 /var/log/asterisk/save_input_debug.log
```

### Step 5: Verify Installation
```bash
# Check H.323 peer status
asterisk -rx "ooh323 show peers"

# Verify dialplan is loaded
asterisk -rx "dialplan show survey-ivr"

# Test AGI script manually
echo "" | sudo -u asterisk python /var/lib/asterisk/agi-bin/save_input.py "TEST" "5"

# Check if data was written
cat /var/log/asterisk/survey_responses.csv
```

---

## Testing Procedures

### Test 1: Direct IVR Call
```bash
# From extension 3477, dial 3459
# Expected: Hear survey prompt, press digit 1-5, hear thank you message
```

### Test 2: Full Call Flow with Avaya Transfer
```bash
# 1. From extension 3477, dial 3868 (Avaya)
# 2. Avaya answers the call
# 3. Avaya transfers to 3459
# 4. Hear survey prompt
# 5. Press digit 1-5
# 6. Hear thank you message
# 7. Call ends
```

### Test 3: Verify Data Collection
```bash
# Check CSV file
cat /var/log/asterisk/survey_responses.csv

# Check debug log
cat /var/log/asterisk/save_input_debug.log

# Watch live logs
sudo tail -f /var/log/asterisk/full | grep -E "survey-ivr|save_input"
```

---

## Monitoring and Troubleshooting

### View Real-Time Logs
```bash
# Monitor survey activity
sudo tail -f /var/log/asterisk/full | grep -E "Survey IVR|Saving digit|AGI"

# Monitor DTMF detection
sudo tail -f /var/log/asterisk/full | grep -i dtmf

# Monitor H.323 activity
sudo tail -f /var/log/asterisk/ooh323_log
```

### Common Issues

**Issue: DTMF Not Detected**
```bash
# Solution: Verify DTMF mode in ooh323.conf
asterisk -rx "ooh323 show peer Avaya"
# Should show: dtmfmode=inband
```

**Issue: AGI Script Not Executing**
```bash
# Check script permissions
ls -la /var/lib/asterisk/agi-bin/save_input.py
# Should show: -rwxr-xr-x asterisk asterisk

# Test script manually
echo "" | sudo -u asterisk python /var/lib/asterisk/agi-bin/save_input.py "3477" "3"
```

**Issue: Data Not Saved to CSV**
```bash
# Check CSV file permissions
ls -la /var/log/asterisk/survey_responses.csv
# Should show: -rw-rw-r-- asterisk asterisk

# Check debug log for errors
tail -20 /var/log/asterisk/save_input_debug.log
```

**Issue: Avaya Connection Failed**
```bash
# Verify H.323 trunk status
asterisk -rx "ooh323 show peers"

# Reload H.323 module
asterisk -rx "module reload chan_ooh323.so"

# Check Avaya IP reachability
ping 10.23.48.15
```

---

## File Permissions Summary

| File Path | Owner | Permissions | Octal |
|-----------|-------|-------------|-------|
| `/etc/asterisk/ooh323.conf` | asterisk:asterisk | -rw-r--r-- | 644 |
| `/etc/asterisk/extensions_custom.conf` | asterisk:asterisk | -rw-r--r-- | 644 |
| `/var/lib/asterisk/agi-bin/save_input.py` | asterisk:asterisk | -rwxr-xr-x | 755 |
| `/var/log/asterisk/survey_responses.csv` | asterisk:asterisk | -rw-rw-r-- | 664 |
| `/var/log/asterisk/save_input_debug.log` | asterisk:asterisk | -rw-rw-rw- | 666 |

---

## System Requirements

- **Operating System:** Linux (CentOS/RHEL recommended)
- **FreePBX:** Version 14 or higher
- **Asterisk:** Version 13 or higher with OOH323 module
- **Python:** Version 2.7 or 3.6+
- **Network:** IP connectivity between FreePBX and Avaya (10.23.48.15)
- **Codecs:** alaw, ulaw

---

## Maintenance

### Backup Configuration Files
```bash
# Backup configuration
sudo tar -czf /root/survey_ivr_backup_$(date +%Y%m%d).tar.gz \
  /etc/asterisk/ooh323.conf \
  /etc/asterisk/extensions_custom.conf \
  /var/lib/asterisk/agi-bin/save_input.py \
  /var/log/asterisk/survey_responses.csv
```

### Archive Old Survey Data
```bash
# Move old data to archive
sudo mv /var/log/asterisk/survey_responses.csv \
  /var/log/asterisk/survey_responses_$(date +%Y%m%d).csv

# Create new empty file
sudo touch /var/log/asterisk/survey_responses.csv
sudo chown asterisk:asterisk /var/log/asterisk/survey_responses.csv
sudo chmod 664 /var/log/asterisk/survey_responses.csv
```

### Clear Debug Logs
```bash
# Truncate debug log
sudo truncate -s 0 /var/log/asterisk/save_input_debug.log
```

---

## Support and Contact

For technical support or questions regarding this implementation, refer to:
- Asterisk documentation: https://docs.asterisk.org
- FreePBX forums: https://community.freepbx.org
- OOH323 module documentation: https://www.voip-info.org/asterisk-h-323

---

**Document Version:** 1.0  
**Last Updated:** November 5, 2025  
**Author:** System Administrator
