# Smart Health Cane (SHC)



## Hardware

| Component | Function |
|---|---|
| Yolo:UNO (Arduino Uno–compatible) | Main controller |
| MAX30102 | Heart rate |
| MPU6050 | Fall detection via Z-axis accelerometer deviation |
| Ultrasonic sensor | Obstacle detection, 3–200 cm range |
| Light sensor | Ambient light sensing → auto LED |
| AI Camera Kit V2 (OhStem) | Runs a Google Teachable Machine model for signage / crosswalk-marking recognition |
| Buzzer module | Audio alerts (3.3–5V, ~2.5 kHz) |
| Sound Player module | Plays recorded alert clips |
| E-Ra IoT Platform (WiFi) | Streams live heart rate + X/Y/Z accelerometer data to a web dashboard and mobile app; pushes fall/heart-rate alerts to a caregiver's phone in real time |

## AI training

Images were sorted into 3 labeled folders (one per recognition class) and trained using **Google Teachable Machine** — a no-code AI training tool. The resulting model runs on-device on the AI Camera Kit for real-time classification (crosswalk markings vs. pedestrian signage).

<p align="center">
  <img src="docs/ai_dataset.jpg" width="380"/>
  <img src="docs/ai_result.jpg" width="380"/>
</p>
<p align="center"><i>Training data — one labeled folder per class &nbsp;•&nbsp; live classification result on-device</i></p>

## How it was built

Programmed using **Yolo Code**, a block-based (drag-and-drop) visual programming environment for the Yolo:UNO board — conceptually similar to Scratch/Blockly. The board itself supports Scratch, Python, and C++, but this project's logic (sensor init, priority ranking, alert dispatch, AI camera response) was designed and assembled block-by-block rather than typed as raw text code.

> Screenshots of the block program are below — this is the actual source, since the block editor doesn't export to a standalone text file.

<p align="center">
  <img src="docs/priority_blocks.jpg" width="760"/>
</p>

## Live monitoring & alerts

Heart rate and fall-detection data stream to the E-Ra IoT dashboard in real time. When a fall or abnormal heart rate is confirmed, a push notification is sent directly to the caregiver's phone.

<p align="center">
  <img src="docs/dashboard.jpg" width="500"/>
</p>
<p align="center"><i>E-Ra dashboard — live heart rate and accelerometer (X/Y/Z) readout</i></p>

<p align="center">
  <img src="docs/alert_heart.jpg" width="370"/>
  <img src="docs/alert_fall.jpg" width="370"/>
</p>
<p align="center"><i>Real alert — abnormal heart rate &nbsp;•&nbsp; real alert — fall detected</i></p>

## Key design decisions

### 1. Handling simultaneous events — the "priority" variable
Multiple sensor events (abnormal heart rate, a fall) can trigger at once. Rather than letting alerts overwrite each other, a `PRIORITY` variable ranks which condition fires first: abnormal heart rate **and** a fall together (highest), heart rate alone, then a fall alone.

*Pseudocode below — a readable translation of the block logic in `docs/`, not the literal source (the block editor has no text export).*

```
function CHECK():
    if (heart_rate < 60 OR heart_rate > 100) AND abs(Z_baseline - Z_now) >= 0.3:
        PRIORITY = 1   # both conditions — most critical
    elif heart_rate < 60 OR heart_rate > 100:
        PRIORITY = 2   # abnormal heart rate only
    elif abs(Z_baseline - Z_now) >= 0.3:
        PRIORITY = 3   # fall only

function ALERT():
    if PRIORITY == 1:
        alert_heart_rate()
        wait(4 seconds)
        alert_fall()
    elif PRIORITY == 2:
        alert_heart_rate()
    elif PRIORITY == 3:
        alert_fall()
```

### 2. Fixing a false-positive assumption
Initial version treated "cane drops" as equivalent to "user falls" (using accelerometer Z-axis deviation alone as the fall trigger). Feedback question — *"what if only the cane falls, not the person?"* — exposed the flawed assumption.

Current implementation still triggers on Z-axis deviation ≥ 0.3, but a fall alert is deliberately staged **after** the heart-rate check (see `PRIORITY == 1` path above — a 4-second pause between the two alerts) so an abnormal heart rate reading, if present, gets flagged and cross-referenced rather than the fall alert firing in isolation.

*Again, pseudocode translated from the block program — not literal source.*

```
function ALERT_HEART_RATE():
    if 100 < heart_rate < 160:
        play_sound(track=2); wait(5s); play_alert_tone(1000ms)
    elif 0 < heart_rate < 70:
        play_sound(track=3); wait(5s); play_alert_tone(1000ms)

function ALERT_FALL():
    if abs(Z_baseline - Z_now) >= 0.3:
        play_sound(track=1); wait(6s); play_alert_tone(1000ms)
```

## Repo structure

```
smart-health-cane/
├── docs/             # priority_blocks.jpg, ai_dataset.jpg, ai_result.jpg,
│                     # dashboard.jpg, alert_heart.jpg, alert_fall.jpg
├── media/            # video demo (nếu có)
└── README.md
```

---
