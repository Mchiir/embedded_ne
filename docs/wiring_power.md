# Wiring, Power Architecture & Safety

## Connections

```
            ┌─────────────── ESP8266 (NodeMCU / Wemos D1 mini) ───────────────┐
 micro-USB  │  USB 5V ── on-board 3V3 regulator                               │
 (PC / hub) │                                                                  │
            │   D1 ───────────────► Servo SIGNAL (orange/white)        │
            │   GND ──────────┬───────────► Servo GND  (brown/black)           │
            └─────────────────┼────────────────────────────────────────────────┘
                              │  COMMON GROUND (mandatory)
                              │
        External 5V supply ───┴── Servo V+ (red)
        (5V, ≥1A SG90 / ≥2A MG996R)
```

| Servo wire    | Connect to                              |
|---------------|------------------------------------------|
| Signal (orange/white) | ESP8266 **D4 / GPIO2**           |
| V+ (red)      | **External 5V** supply (preferred) or VIN |
| GND (brown)   | **Common ground** with ESP8266 GND        |
