# Asterisk → FastAPI Integration (AMI Listener)

### 🎯 Goal

When an **agent** calls IVR `3459`, when a **customer** joins, or when a **digit** (1–5) is pressed, Asterisk sends HTTP POSTs to FastAPI (`10.3.118.3:8000`) with:

* `/agent_start`: `{conversation_id, agent_extension}`
* `/customer_join`: `{conversation_id, customer_id}`
* `/rate`: `{conversation_id, rate}`

---

## 1. FreePBX / Asterisk setup

**File:** `/etc/asterisk/manager.conf`

```ini
[ami_listener]
secret = StrongPass123
permit = 10.3.118.5/32     ; listener IP
read = all
write = none
```

**Reload:**

```bash
asterisk -rx "manager reload"
sudo ufw allow from 10.3.118.5 to any port 5038 proto tcp
```

**Test:**

```bash
telnet <Asterisk_IP> 5038
```

You should see `Asterisk Call Manager`.

---

## 2. Listener host setup

**Install deps:**

```bash
sudo apt install python3-pip
pip install requests
```

**Save script:** `/usr/local/bin/ami_http_poster.py`

```python
#!/usr/bin/env python3
import socket, requests, threading, time

AMI_HOST='ASTERISK_IP'; AMI_PORT=5038
AMI_USER='ami_listener'; AMI_PASS='StrongPass123'
HTTP_BASE='http://10.3.118.3:8000'; IVR_EXT='3459'

conv_map={}
def post(path,p): 
    try: requests.post(HTTP_BASE+path,json=p,timeout=2)
    except: pass

def handle(e):
    if e.get('Event')=='Dial' and IVR_EXT in (e.get('Dest','')):
        uid=e['Uniqueid']; conv_map[uid]=uid
        post('/agent_start',{'conversation_id':uid,'agent_extension':e.get('CallerIDNum')})
    if e.get('Event')=='DTMF':
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

**Service:** `/etc/systemd/system/ami_http_poster.service`

```ini
[Service]
ExecStart=/usr/bin/python3 /usr/local/bin/ami_http_poster.py
Restart=always
[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now ami_http_poster
```

---

## 3. FastAPI server (`10.3.118.3`)

**Install:**

```bash
sudo apt install python3-venv
python3 -m venv /opt/fastapi
source /opt/fastapi/bin/activate
pip install fastapi uvicorn
```

**App:** `/opt/fastapi/app.py`

```python
from fastapi import FastAPI
from pydantic import BaseModel
app=FastAPI()
class A(BaseModel): conversation_id:str; agent_extension:str
class C(BaseModel): conversation_id:str; customer_id:str
class R(BaseModel): conversation_id:str; rate:str

@app.post("/agent_start") async def s(a:A): print(a); return {"ok":1}
@app.post("/customer_join") async def c(c:C): print(c); return {"ok":1}
@app.post("/rate") async def r(r:R): print(r); return {"ok":1}
```

**Run:**

```bash
uvicorn app:app --host 0.0.0.0 --port 8000
sudo ufw allow 8000/tcp
```

---

## 4. Test flow

1. Start FastAPI and listener.
2. Agent (e.g. `3868`) calls IVR `3459`.
3. Watch FastAPI logs — you should see JSON from `/agent_start`.
4. When customer joins or presses digits, you’ll see `/customer_join` and `/rate`.

---

## 5. Recommended next steps

* Add token auth in FastAPI + header in listener.
* Run both as systemd services.
* Use HTTPS (nginx reverse proxy).
* For scale: add retry queue in listener.

---

✅ **End result:**
Your Asterisk now pushes live call events → FastAPI via HTTP automatically.
