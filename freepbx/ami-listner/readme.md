# 🧩 Asterisk (FreePBX) → FastAPI Integration via AMI Listener

**Version:** Final working setup
**Environment:**

* **FreePBX / Asterisk server:** `10.23.48.100`
* **AMI listener + FastAPI host:** `10.23.223.2`

---

## 🎯 Overview

The goal is for Asterisk to send live call events to FastAPI using an **AMI listener script**:

* When an **agent** starts a call with IVR `3459` → POST `/agent_start`
* When a **customer** joins the conversation → POST `/customer_join`
* When a **digit** is pressed in the IVR → POST `/rate`

All messages go to:
👉 `http://10.3.118.3:8000/` (FastAPI app)

---

## 1️⃣ FreePBX / Asterisk Configuration

### 1.1 Enable AMI for external access

Edit `/etc/asterisk/manager.conf`:

```ini
[general]
enabled = yes
port = 5038
bindaddr = 10.23.48.100      ; PBX LAN IP
displayconnects = no

[admin]
secret = AvGcGKBjcpPy
deny = 0.0.0.0/0.0.0.0
permit = 127.0.0.1/255.255.255.0
read = system,call,log,verbose,command,agent,user,config,command,dtmf,reporting,cdr,dialplan,originate,message
write = system,call,log,verbose,command,agent,user,config,command,dtmf,reporting,cdr,dialplan,originate,message

[ami_listener]
secret = StrongPass123
permit = 10.23.223.2/32      ; only allow the listener host
read = all
write = none

#include manager_additional.conf
#include manager_custom.conf
```

### 1.2 Reload Asterisk Manager

```bash
sudo asterisk -rx "manager reload"
```

### 1.3 Verify it’s listening

```bash
sudo ss -tlnp | grep 5038
```

Expected:

```
LISTEN  0  50 10.23.48.100:5038  *:*  users:(("asterisk",pid=2346,fd=12))
```

### 1.4 Allow listener IP in firewall

Your FreePBX uses raw iptables (not firewalld).
Run once and save:

```bash
sudo iptables -I INPUT -p tcp -s 10.23.223.2 --dport 5038 -j ACCEPT
sudo iptables-save > /etc/sysconfig/iptables
```

### 1.5 Test connection from listener

From host `10.23.223.2`:

```bash
telnet 10.23.48.100 5038
```

Expected:

```
Asterisk Call Manager/7.0.3
```

Then test login:

```
Action: Login
Username: ami_listener
Secret: StrongPass123

```

Response:

```
Response: Success
Message: Authentication accepted
```

✅ AMI connection working.

---

## 2️⃣ Listener Host (`10.23.223.2`)

### 2.1 Install requirements

```bash
sudo apt update
sudo apt install python3 python3-pip -y
pip install requests
```

### 2.2 Save listener script

File: `/usr/local/bin/ami_http_poster.py`

```python
#!/usr/bin/env python3
import socket, requests, threading, time

AMI_HOST='10.23.48.100'
AMI_PORT=5038
AMI_USER='ami_listener'
AMI_PASS='StrongPass123'
HTTP_BASE='http://10.3.118.3:8000'
IVR_EXT='3459'
conv_map={}

def post(path,p):
    try: requests.post(HTTP_BASE+path,json=p,timeout=2)
    except Exception as e: print("HTTP error:",e)

def handle(e):
    ev=e.get('Event')
    if ev=='Dial' and IVR_EXT in (e.get('Dest','')):
        uid=e['Uniqueid']; conv_map[uid]=uid
        post('/agent_start',{'conversation_id':uid,'agent_extension':e.get('CallerIDNum')})
    if ev=='DTMF':
        uid=e.get('Uniqueid'); d=e.get('Digit')
        if uid in conv_map: post('/rate',{'conversation_id':uid,'rate':d})

def read(sock):
    buf=''
    while True:
        data=sock.recv(4096).decode()
        buf+=data
        while '\r\n\r\n' in buf:
            blk,buf=buf.split('\r\n\r\n',1)
            e={k:v for k,v in [l.split(':',1) for l in blk.split('\r\n') if ':' in l]}
            if 'Event' in e: handle(e)

while True:
    try:
        s=socket.create_connection((AMI_HOST,AMI_PORT))
        s.sendall(f"Action: Login\r\nUsername: {AMI_USER}\r\nSecret: {AMI_PASS}\r\n\r\n".encode())
        threading.Thread(target=read,args=(s,),daemon=True).start()
        while True: time.sleep(5)
    except Exception as e:
        print("Reconnect...",e); time.sleep(5)
```

Make executable:

```bash
sudo chmod +x /usr/local/bin/ami_http_poster.py
```

### 2.3 Create systemd service

`/etc/systemd/system/ami_http_poster.service`

```ini
[Unit]
Description=Asterisk AMI → HTTP Poster
After=network.target

[Service]
ExecStart=/usr/bin/python3 /usr/local/bin/ami_http_poster.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Start and enable:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ami_http_poster.service
sudo journalctl -u ami_http_poster.service -f
```

✅ Listener now stays running and reconnects automatically.

---

## 3️⃣ FastAPI Server (`10.3.118.3`)

### 3.1 Install FastAPI

```bash
sudo apt update
sudo apt install python3-venv -y
python3 -m venv /opt/fastapi
source /opt/fastapi/bin/activate
pip install fastapi uvicorn
deactivate
```

### 3.2 Create app

`/opt/fastapi/app.py`

```python
from fastapi import FastAPI
from pydantic import BaseModel

app=FastAPI()

class Agent(BaseModel):
    conversation_id:str
    agent_extension:str

class Customer(BaseModel):
    conversation_id:str
    customer_id:str

class Rate(BaseModel):
    conversation_id:str
    rate:str

@app.post("/agent_start")
async def start(a:Agent):
    print("Agent start:",a)
    return {"status":"ok"}

@app.post("/customer_join")
async def join(c:Customer):
    print("Customer join:",c)
    return {"status":"ok"}

@app.post("/rate")
async def rate(r:Rate):
    print("Rate:",r)
    return {"status":"ok"}
```

### 3.3 systemd service

`/etc/systemd/system/fastapi_ami.service`

```ini
[Unit]
Description=FastAPI AMI Receiver
After=network.target

[Service]
WorkingDirectory=/opt/fastapi
ExecStart=/opt/fastapi/bin/uvicorn app:app --host 0.0.0.0 --port 8000
Restart=always

[Install]
WantedBy=multi-user.target
```

Enable + start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now fastapi_ami.service
sudo journalctl -u fastapi_ami.service -f
```

### 3.4 Open firewall port 8000

```bash
sudo ufw allow 8000/tcp
```

*(Or adjust your firewall if not using UFW.)*

---

## 4️⃣ End-to-End Test

1. Confirm listener and FastAPI services are running.
2. Make an agent call to IVR 3459 on FreePBX.
3. Watch:

   ```bash
   # On listener host
   sudo journalctl -u ami_http_poster.service -f
   # On FastAPI host
   sudo journalctl -u fastapi_ami.service -f
   ```
4. You should see:

   * `/agent_start` when call starts
   * `/rate` when customer presses a digit
   * `/customer_join` when you implement that event (optional)

---

## 5️⃣ Security & Maintenance

* `manager.conf` → only `permit 10.23.223.2/32`.
* Use a **strong secret** for `ami_listener`.
* Never expose port 5038 to the internet.
* Run both services with non-root users if possible.
* Back up `/etc/sysconfig/iptables` after changes.

---

## ✅ Quick verification commands

| Test              | Command                                                                                                                                      | Expected                  |                     |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------- | ------------------- |
| AMI listening     | `ss -tlnp                                                                                                                                    | grep 5038`                | `10.23.48.100:5038` |
| AMI login         | `printf "Action: Login\r\nUsername: ami_listener\r\nSecret: StrongPass123\r\n\r\n" \| nc 10.23.48.100 5038`                                  | “Authentication accepted” |                     |
| FastAPI reachable | `curl -X POST http://10.3.118.3:8000/agent_start -d '{"conversation_id":"t1","agent_extension":"3868"}' -H "Content-Type: application/json"` | `{"status":"ok"}`         |                     |
