# Besties AI — ระบบ AI Judging สำหรับการประกวดนวัตกรรม

> **Last Updated**: 2026-04-29  
> **Status**: Active Development — เหลือ Pending item 1 ก่อน production

---

## 📋 ภาพรวมระบบ

**Besties AI** คือระบบ AI Judging Automation สำหรับงานประกวด Innovation Pitch ภายในองค์กร ประกอบด้วย:
- Speech-to-Text (STT) จับเสียงผู้นำเสนอแบบ real-time
- AI วิเคราะห์และให้คะแนนผ่าน Google Gemini
- Text-to-Speech (TTS) ให้ Besties Avatar พูดตอบกลับ
- หน้าแสดงผลสาธารณะบน TV/จอใหญ่
- ฟอร์มให้คะแนนสำหรับกรรมการบน mobile
- Audio Visualizer Avatar แทน static image

---

## 🏗️ สถาปัตยกรรมระบบ

```
┌──────────────────────────────────────────────────────────┐
│                     Frontend (HTML)                       │
│                                                           │
│  besties-ai-system.html  ──BC──▶  besties-display.html   │
│       (Operator)          chan         (TV Screen)        │
│                                                           │
│  besties-jury.html                besties-avatar-b.html  │
│  (Judge Mobile Form)              (Avatar Visualizer)     │
└──────────────────────┬───────────────────────────────────┘
                       │ HTTP POST/GET
┌──────────────────────▼───────────────────────────────────┐
│                  n8n Workflows                            │
│  besties-ai-n8n-workflow.json     (KB + Sessions + Jury) │
│  besties-ai-analyze-workflow.json (AI Analysis)          │
└──────────────────────┬───────────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────────┐
│               External Services                           │
│  Google Sheets (Data Storage)   Google Gemini (AI/TTS)   │
└──────────────────────────────────────────────────────────┘
```

### BroadcastChannel (`besties-display`)

`besties-ai-system.html` ส่งข้อความหา `besties-display.html` (และ `besties-avatar-b.html`) ผ่าน BroadcastChannel API — ต้องเปิดทั้งสอง tab ในเบราว์เซอร์เดียวกัน

| type | ผู้ส่ง | ผลที่ display |
|------|--------|--------------|
| `session` | system | แสดง teamName + sessionId ที่ header |
| `score` | system | แสดง score panel พร้อม bars + feedback |
| `question` | system | แสดง question card + Avatar พูดคำถาม |
| `question_clear` | system | ซ่อน question card |
| `reset` | system | reset หน้าจอกลับ waiting state |
| `gesture` | system | แสดง emoji animation บน avatar |
| `timer_start` | system | เริ่มนับเวลา countdown |
| `timer_stop` | system | หยุด timer |

---

## 📁 โครงสร้างไฟล์

```
avatar-ai/
├── Besties.png                           ← Avatar image (illustration style)
├── besties-ai-system.html                ← Operator control panel
├── besties-display.html                  ← Public display screen (TV)
├── besties-jury.html                     ← Mobile jury scoring form
├── besties-avatar-b.html                 ← Audio Visualizer avatar (Path B)
├── besties-avatar-test.html              ← D-ID test page (ทดลองแล้ว ไม่ใช้)
├── besties-ai-n8n-workflow.json          ← n8n: KB + Sessions + Jury endpoints
└── besties-ai-analyze-workflow.json      ← n8n: AI Analysis (⚠️ ยังต้องแก้เกณฑ์)
```

---

## 📡 n8n Workflows

### 1. `besties-ai-n8n-workflow.json` — KB, Sessions, Jury

#### HTTP Endpoints

| Endpoint | Method | วัตถุประสงค์ |
|----------|--------|-------------|
| `/besties-kb` | POST | บันทึก STT / project_brief / qa_answer ลง KnowledgeBase |
| `/besties-kb` | GET | ดึง KB context ของ session (พร้อม Build Context) |
| `/besties-sessions` | POST | สร้าง session ใหม่ |
| `/besties-sessions` | GET | ดูรายการ sessions ทั้งหมด |
| `/besties-jury` | POST | บันทึกคะแนนกรรมการ |
| `/besties-jury` | GET | ดึงคะแนนกรรมการของ session |

#### Payload Format (flat JSON — ไม่มี body wrapper)

```json
// POST /besties-kb
{ "type": "stt", "text": "...", "session": "id", "ts": "ISO8601" }
{ "type": "project_brief", "text": "...", "session": "id", "ts": "ISO8601" }
{ "type": "qa_answer", "text": "...", "session": "id", "ts": "ISO8601" }

// POST /besties-sessions
{ "session": "id", "teamName": "Team Alpha" }

// POST /besties-jury
{
  "session": "id", "judgeName": "ชื่อกรรมการ",
  "businessImpact": 8, "aiFit": 7, "dataFeasibility": 8,
  "workflowAdoption": 7, "riskScalability": 6,
  "comment": "ความเห็น", "weightedScore": 74
}
```

> ⚠️ **สำคัญ**: n8n อ่าน body โดยอัตโนมัติเป็น `$json.body.X` — ห้าม wrap ด้วย `{ body: ... }` อีกชั้น (จะกลาย double-nested)

#### Build Context Node (node-0007)

ดึง KB ของ session แล้วจัดลำดับ context:
```
[ข้อมูลโครงการล่วงหน้า]      ← type: project_brief หรือ workshop
(Project Brief / Workshop data)

[การนำเสนอสด]                ← type: stt (เฉพาะ N รายการล่าสุด)
(STT transcript)
```

#### Google Sheets Structure

| Sheet | คอลัมน์หลัก |
|-------|------------|
| Besties AI — KnowledgeBase | Timestamp, Session, Type, Text |
| Besties AI — Sessions | Session ID, Team Name, Created At |
| Besties AI — Results | Session, Team, Scores, Summary, ... |
| Jury Scores | Session, Judge, 5 criteria scores, Weighted, Comment, Timestamp |

- **Spreadsheet ID**: `1dd45yr9_ceioBxb6E6mQ-XQydVug77V4Ti5fFvdCB1o`

---

### 2. `besties-ai-analyze-workflow.json` — AI Analysis

#### ⚠️ PENDING: เกณฑ์ยังใช้อยู่ไม่ถูก (ต้องแก้)

ปัจจุบันยังใช้ 4 เกณฑ์เก่า ต้องอัปเดตเป็น 5 เกณฑ์อย่างเป็นทางการ:

| เกณฑ์ปัจจุบัน (ผิด) | เกณฑ์ที่ถูกต้อง | น้ำหนัก |
|--------------------|----------------|--------|
| creativity (0-25) | **Business Impact** | 30% |
| feasibility (0-25) | **AI Fit** | 20% |
| clarity (0-25) | **Data Feasibility** | 20% |
| impact (0-25) | **Workflow Adoption** | 20% |
| — | **Risk & Scalability** | 10% |

งานที่ต้องทำ:
1. แก้ "Build Prompt" node — เปลี่ยน rubric ใน prompt
2. แก้ "Parse Result" node — map field ใหม่ทั้ง 5
3. แก้ "Sheets: Save Result" node — คอลัมน์ตรงกับ 5 เกณฑ์
4. แก้ `besties-display.html` — score bar labels + `onScore()` function

#### Data Flow

```
Webhook: /besties-analyze (POST: session, apiKey)
  ↓
Sheets: Read KB (อ่าน STT ของ session)
  ↓
Build Prompt (สร้าง prompt + rubric)
  ↓
Gemini API (gemini-2.0-flash หรือ gemini-3-flash-preview)
  ↓
Parse Result (แยก JSON: scores, summary, strengths, improvements)
  ↓
Sheets: Save Result
  ↓
Respond: Analysis Result
```

#### Response Format (ปัจจุบัน — จะเปลี่ยนเมื่อแก้ Pending)

```json
{
  "ok": true,
  "session": "id",
  "totalScore": 78,
  "breakdown": {
    "creativity": 22, "feasibility": 19,
    "clarity": 20,   "impact": 17
  },
  "summary": "สรุปผล",
  "strengths": ["..."],
  "improvements": ["..."]
}
```

---

## 🖥️ Frontend Pages

### 1. `besties-ai-system.html` — Operator Control Panel

**Dark theme** | ใช้งานโดย operator ระหว่างงาน

#### Layout (2-column)
- **Left Sidebar**: Avatar panel, session setup, KB viewer, project brief input
- **Right Main**: Chat log, STT control, mode selector, analyze button

#### Mode การทำงาน

| Mode | ปุ่ม STT | ส่ง n8n | ส่ง display |
|------|---------|--------|------------|
| `pitch` | จับเสียงนำเสนอ | `type: stt` (บันทึกลง KB) | ไม่แสดงคำถาม |
| `qa` | จับเสียงตอบคำถาม | `type: qa_answer` (ไม่ลง KB) | — |

#### Project Brief Pre-load (item 3 ✅)

Operator สามารถพิมพ์ข้อมูล brief ของทีมก่อน session เริ่ม → บันทึกเป็น `type: project_brief` ใน KB → Gemini จะเห็นข้อมูลนี้ก่อน STT ใดๆ

#### AI Models (4 บทบาท)

| Model | บทบาท | ทำงานเมื่อ |
|-------|--------|-----------|
| Model A | Presentation Analyst | mode: pitch |
| Model B | Smart Interrogator | mode: qa (ถามคำถาม → แสดงบนจอ display) |
| Model C | Feedback Harmonizer | วิเคราะห์รอบสุดท้าย |
| Model D | Bestie Persona + TTS | ทุก mode (พูดตอบกลับ) |

#### Gemini TTS
- Model: `gemini-3.1-flash-tts-preview`
- เล่น audio และแสดง waveform animation พร้อมกัน
- เสียง Model D จะ broadcast ไปยัง display screen ผ่าน `type: question`

---

### 2. `besties-display.html` — Public Display Screen

**Dark theme** | แสดงบน TV/projector สำหรับผู้ชม

#### Sections

```
┌─────────────────────────────────────────────┐
│  TOP BAR: Logo | Team Name | Session ID      │
├──────────┬──────────────┬────────────────────┤
│          │   AVATAR +   │                    │
│ SCORE    │   WAVEFORM   │  (empty / timer)   │
│ PANEL    │              │                    │
│ (right)  │   TIMER      │                    │
├──────────┴──────────────┴────────────────────┤
│  BOTTOM: Status dot + message                │
└─────────────────────────────────────────────┘
```

#### Question Panel (item 2 ✅)

เมื่อ Model B ถามคำถาม → `type: question` broadcast → แสดง card:
```
┌────────────────────────┐
│  AI Bestie ถาม         │
│  คำถามสำหรับทีม        │
│  [ข้อความคำถาม...]     │
│  ตอบคำถามนี้ภายในเวลา  │
└────────────────────────┘
```

#### Score Panel (ยัง Pending — labels เก่า)
Score bars ยังชื่อ creativity/feasibility/clarity/impact → ต้องอัปเดตพร้อม analyze workflow

---

### 3. `besties-jury.html` — Mobile Jury Scoring Form (item 4 ✅)

**Mobile-first** | กรรมการใช้บน smartphone ระหว่างงาน

#### Features
- Setup card: N8N URL, Session ID, Judge Name (บันทึกใน localStorage)
- รองรับ URL params: `?session=XXX&team=YYY`
- Slider 0–10 สำหรับแต่ละเกณฑ์ (แสดง weight %)
- คำนวณ weighted score real-time: `Σ(score × weight × 10)`
- Submit → `POST {n8nUrl}/besties-jury` (flat JSON)
- Success screen + ปุ่ม "ประเมินทีมถัดไป"

#### 5 เกณฑ์การให้คะแนน (คะแนน 0–10 × weight)

| เกณฑ์ | Weight | Max contribution |
|------|--------|-----------------|
| Business Impact | 30% | 30 pts |
| AI Fit | 20% | 20 pts |
| Data Feasibility | 20% | 20 pts |
| Workflow Adoption | 20% | 20 pts |
| Risk & Scalability | 10% | 10 pts |
| **รวม** | **100%** | **100 pts** |

---

### 4. `besties-avatar-b.html` — Audio Visualizer Avatar

**Full-screen** | ทำงานได้ offline — ใช้ Besties.png ตัวเดิม

#### เหตุผลที่ใช้ Path B แทน D-ID
- D-ID Streaming API ต้องการ realistic human face (ไม่ใช่ illustration)
- Besties.png เป็น illustrated character → D-ID reject ทุกครั้ง
- Path B ทำงาน offline ไม่มี external dependency

#### Features
- **Canvas visualizer**: 90 bars รอบ avatar react กับ audio amplitude
- **Simulated sync**: ถ้าไม่มี real audio stream ใช้ sine wave simulation ที่ดูน่าเชื่อ
- **Particle effects**: กระจายออกจาก avatar เมื่อพูด
- **Idle breathing**: pulse เบาๆ ตลอดเวลา
- **Reactive glow**: แสงขยายตาม amplitude
- **BroadcastChannel**: รับ event จาก system → auto-speak ด้วย SpeechSynthesis API
- **Color schemes**: 4 แบบ (purple/blue/fire/rainbow)
- **TTS**: Web SpeechSynthesis API (รองรับ Thai voice)

#### การ Integrate เข้า besties-display.html (ยังไม่ได้ทำ)
ถ้าต้องการ replace static Besties.png ด้วย visualizer ใน display screen หลัก

---

## 🔄 Session Flow (ครบวงจร)

```
1. Operator เปิด besties-ai-system.html
2. ใส่ Project Brief ของทีม → บันทึกลง KB (type: project_brief)
3. สร้าง Session → POST /besties-sessions
4. เริ่ม Timer → broadcast type: timer_start
5. เปิด mode: pitch → STT จับเสียง → POST /besties-kb (type: stt)
6. Gemini ตอบกลับเป็น Model A/D → TTS เล่นเสียง
7. เปลี่ยน mode: qa → Gemini ถามคำถาม (Model B)
   → broadcast type: question → แสดงบน display screen
8. STT จับเสียงตอบ → POST /besties-kb (type: qa_answer, ไม่บันทึกลง KB)
9. กด Analyze → POST /besties-analyze → Gemini ให้คะแนน
   → broadcast type: score → แสดง score panel บน display
10. กรรมการให้คะแนนผ่าน besties-jury.html → POST /besties-jury
```

---

## ⚠️ Pending Items

### 1. แก้ Analyze Workflow (HIGH — ต้องทำก่อน production)

ต้องแก้ `besties-ai-analyze-workflow.json` **และ** `besties-display.html` พร้อมกัน:

**`besties-ai-analyze-workflow.json`:**
- "Build Prompt" node: เปลี่ยน rubric เป็น 5 เกณฑ์ใหม่
- "Parse Result" node: map fields ใหม่ (`businessImpact`, `aiFit`, `dataFeasibility`, `workflowAdoption`, `riskScalability`)
- "Sheets: Save Result" node: คอลัมน์ตรงกับ 5 fields

**`besties-display.html`:**
- Score bar IDs + labels → 5 ชื่อใหม่
- `onScore()` function → อ่าน fields ใหม่

### 2. Integrate Avatar Visualizer เข้า besties-display.html (OPTIONAL)

Replace static Besties.png ใน display screen ด้วย canvas visualizer จาก `besties-avatar-b.html`

---

## 🐛 Bug ที่แก้แล้ว

| Bug | สาเหตุ | วิธีแก้ |
|-----|--------|--------|
| n8n รับ `body.body.session` | ส่ง `{ body: payload }` ทำ double-nest | ส่ง flat JSON ตรงๆ ทุก endpoint |
| Model B คำถามไม่ขึ้นจอ display | ไม่มี broadcast `type: question` | เพิ่มใน `geminiChat()` เมื่อ mode=qa |
| STT mode qa ปนกับ KB | `processInput` ส่ง `saveKB()` ทุก mode | แยก branch: qa → `type: qa_answer` + skip saveKB |

---

## ⚙️ Configuration

### Credentials ที่ต้องใช้

| Service | ใช้ที่ |
|---------|-------|
| Google Sheets OAuth2 | n8n credentials |
| Google Gemini API Key | ใส่ใน system.html ตอน analyze |
| n8n Webhook Base URL | ใส่ใน system.html + jury.html |

### n8n Webhook Base URL Format
```
http://<n8n-host>:5678/webhook
```

---

## 📚 Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | HTML5 + CSS3 + Vanilla JS |
| Avatar Animation | Canvas 2D API + Web Audio API (simulated) |
| TTS | Google Gemini TTS (`gemini-3.1-flash-tts-preview`) + Web SpeechSynthesis |
| STT | Web Speech API (continuous mode) |
| Backend | n8n workflow automation |
| AI Analysis | Google Gemini (gemini-2.0-flash / gemini-3-flash-preview) |
| Database | Google Sheets |
| Real-time comms | BroadcastChannel API (same-origin tabs) |
| Fonts | Sarabun (Thai), IBM Plex Mono |

---

## 🔐 Security Notes (Production Checklist)

- [ ] เพิ่ม webhook authentication ใน n8n
- [ ] ย้าย Gemini API key จาก request body → server-side proxy
- [ ] จำกัด CORS บน n8n endpoints
- [ ] Google Sheets: ตั้ง access เฉพาะ service account ที่จำเป็น
