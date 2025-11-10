# FUNOFAISIH

## Frontend
  ```
-->html

<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>SIH1706 - Demo Chat UI</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <div class="container">
    <h2>SIH1706 — Secure &amp; Intelligent Chatbot (Demo)</h2>
    <div id="chatbox" class="chatbox"></div>
    <div class="controls">
      <input id="msg" placeholder="Type your question..." />
      <button id="send">Send</button>
    </div>
    <div class="auth">
      <input id="email" placeholder="your email (alice@example.com)" />
      <button id="reqotp">Request OTP (demo)</button>
      <input id="otp" placeholder="otp" />
      <button id="verify">Verify OTP</button>
    </div>
  </div>
<script src="script.js"></script>
</body>
</html>
-->css

body { font-family: Arial, sans-serif; background:#0f1724; color:#e6eef8; display:flex; align-items:center; justify-content:center; height:100vh; margin:0;}
.container { width:420px; background: linear-gradient(180deg,#0b1220,#122033); padding:16px; border-radius:12px; box-shadow: 0 8px 24px rgba(0,0,0,0.6);}
.chatbox { height:320px; overflow:auto; background:#071022; padding:12px; border-radius:8px; margin-bottom:8px;}
.msg { padding:8px 12px; border-radius:12px; display:inline-block; margin:6px 0; max-width:80%;}
.msg.user { background:#1f6feb; align-self:flex-end; color:white; margin-left:auto;}
.msg.bot { background:#223342; color:#d8e8ff;}
.controls { display:flex; gap:8px; }
.controls input { flex:1; padding:8px; border-radius:8px; border:1px solid #224; background:#0b1118; color:#e6eef8;}
.controls button { padding:8px 12px; border-radius:8px; border:none; background:#0ea5a4; color:#082026; cursor:pointer;}
.auth { margin-top:10px; display:flex; gap:6px; flex-wrap:wrap; }
.auth input { padding:6px; border-radius:6px; border:1px solid #224; background:#071022; color:#e6eef8;}
.auth button { padding:6px 8px; border-radius:6px; border:none; background:#7c3aed; color:white;}
-->
let token = null;
const chatbox = document.getElementById("chatbox");
function addMessage(text, cls="bot"){
  const d = document.createElement("div");
  d.className = "msg " + cls;
  d.textContent = text;
  chatbox.appendChild(d);
  chatbox.scrollTop = chatbox.scrollHeight;
}
document.getElementById("reqotp").onclick = async ()=>{
  const email = document.getElementById("email").value;
  if(!email){ alert("enter email"); return; }
  const res = await fetch("http://localhost:8000/auth/request-otp", {
    method:"POST", headers:{"Content-Type":"application/json"}, body: JSON.stringify({email})
  });
  const j = await res.json();
  alert("Demo OTP (visible): " + j.otp);
}
document.getElementById("verify").onclick = async ()=>{
  const email = document.getElementById("email").value;
  const otp = document.getElementById("otp").value;
  const res = await fetch("http://localhost:8000/auth/verify-otp", {
    method:"POST", headers:{"Content-Type":"application/json"}, body: JSON.stringify({email, otp})
  });
  const j = await res.json();
  token = j.token;
  alert("Token acquired (demo). Now you can chat.");
}
document.getElementById("send").onclick = async ()=>{
  const msg = document.getElementById("msg").value;
  if(!msg) return;
  addMessage(msg, "user");
  // demo: if no token, show warning
  if(!token){ addMessage("Please authenticate first (use Request OTP + Verify).","bot"); return; }
  const res = await fetch("http://localhost:8000/chat", {
    method:"POST", headers:{"Content-Type":"application/json"}, body: JSON.stringify({token, message: msg})
  });
  const j = await res.json();
  addMessage(j.reply, "bot");
}
```
## Backend:
```
-->app.py

from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel
from fastapi.middleware.cors import CORSMiddleware
import time, secrets, jwt

from .nlu_stub import NLU, summarize_text

app = FastAPI(title="SIH1706 - Demo Backend")

# CORS for local demo
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

# --- simple in-memory "database" ---
USERS = {
    "alice@example.com": {"name": "Alice", "otp": None}
}

JWT_SECRET = "demo-jwt-secret"
JWT_ALGO = "HS256"

class AuthRequest(BaseModel):
    email: str

class OTPVerifyRequest(BaseModel):
    email: str
    otp: str

class ChatRequest(BaseModel):
    token: str
    message: str

nlu = NLU()

@app.post("/auth/request-otp")
def request_otp(req: AuthRequest):
    if req.email not in USERS:
        raise HTTPException(400, "Unknown user")
    otp = str(secrets.randbelow(900000) + 100000)
    USERS[req.email]['otp'] = otp
    # In real system: send to email. Here we return it for demo.
    return {"msg": "OTP generated (demo)", "otp": otp}

@app.post("/auth/verify-otp")
def verify_otp(req: OTPVerifyRequest):
    user = USERS.get(req.email)
    if not user or user.get("otp") != req.otp:
        raise HTTPException(401, "Invalid OTP")
    payload = {"sub": req.email, "iat": int(time.time())}
    token = jwt.encode(payload, JWT_SECRET, algorithm=JWT_ALGO)
    # clear otp
    user['otp'] = None
    return {"token": token}

@app.post("/chat")
def chat(req: ChatRequest):
    try:
        payload = jwt.decode(req.token, JWT_SECRET, algorithms=[JWT_ALGO])
    except Exception as e:
        raise HTTPException(401, "Invalid token")
    user_email = payload.get("sub")
    intent, entities = nlu.parse(req.message)
    # Simple routing: if user asks to summarize, we return a demo summary.
    if intent == "summarize_document":
        doc = open("example_document.txt", "r").read()
        summary = summarize_text(doc)
        return {"reply": summary, "source": "example_document.txt", "intent": intent, "entities": entities}
    # Otherwise echo with intent label
    reply = f"[{intent}] I understood entities: {entities}. Your message: {req.message}"
    return {"reply": reply, "intent": intent, "entities": entities}
-->nlp

import re
from typing import Tuple

INTENTS = {
    "hr_query": ["leave", "salary", "increment", "pf", "pension", "appraisal"],
    "it_support": ["wifi", "internet", "vpn", "password", "login", "error", "bug"],
    "policy_query": ["policy", "circular", "guideline", "protocol"],
    "summarize_document": ["summarize", "summary", "summarise"],
    "general_faq": ["when", "how", "where", "who", "what"]
}

def _detect_intent(text: str):
    t = text.lower()
    for intent, keywords in INTENTS.items():
        for k in keywords:
            if k in t:
                return intent
    return "unknown"

def _extract_entities(text: str):
    ents = {}
    # emails
    m = re.search(r'[\w\.-]+@[\w\.-]+', text)
    if m:
        ents['email'] = m.group(0)
    # dates (very simple)
    m2 = re.search(r'\b(20\d{2}|\d{1,2}/\d{1,2}/\d{2,4})\b', text)
    if m2:
        ents['date'] = m2.group(0)
    return ents

class NLU:
    def parse(self, text: str) -> Tuple[str, dict]:
        intent = _detect_intent(text)
        ents = _extract_entities(text)
        return intent, ents

def summarize_text(text:str, max_sentences:int=3)->str:
    # trivial summarizer: pick first N sentences
    sents = re.split(r'(?<=[.!?])\s+', text.strip())
    return " ".join(sents[:max_sentences])
```
## output
<img width="586" height="838" alt="chat_ui_mockup" src="https://github.com/user-attachments/assets/06b82604-f636-44b6-a98f-8a73085a3671" />

<img width="586" height="838" alt="chat_ui_mockup" src="https://github.com/user-attachments/assets/6f8465e4-3cdb-4488-9b3b-44af2fa0429d" />
