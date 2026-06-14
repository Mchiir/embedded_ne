Deployment Architecture

Physical deployment of the system components.

```mermaid
flowchart LR

    USER[Speaker]

    CAMERA[USB Camera]

    PC[PC Running BENAX]

    MQTT[(Mosquitto Broker)]

    WIFI[Wi-Fi Network]

    ESP[ESP8266]

    SERVO[Servo]

    USER --> CAMERA

    CAMERA --> PC

    PC --> MQTT

    MQTT --> WIFI

    WIFI --> ESP

    ESP --> SERVO
```