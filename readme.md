# Precision Agriculture IoT Platform

Livestock tracking & geofencing system using GPS collars, LoRa, Firebase, and a mobile app.

---

## System Architecture

```mermaid
graph LR
    subgraph "Layer 1 – Sensor<br/>(On Animal)"
        SAT[🛰 GPS Satellites]
        GPS[GPS Module<br/>Neo-6M]
        MCU[ESP32 Node]
        LORA_TX[LoRa TX<br/>SX1276]
        BUZ[🔔 Buzzer<br/>85-100 dB]
        BAT[🔋 Battery<br/>5-7 days]
    end

    subgraph "Layer 2 – Gateway<br/>(On Farm)"
        LORA_RX[LoRa RX<br/>SX1276]
        GW[ESP32 Gateway]
        GEO[Geofencing Engine<br/>Ray-Casting]
        WIFI[WiFi / 4G]
    end

    subgraph "Layer 3 – Cloud<br/>(Google)"
        FB[Firebase<br/>Realtime DB]
        GCS[Cloud Storage<br/>Archive]
        CF[Cloud Functions]
        FCM[FCM Push<br/>Notifications]
    end

    subgraph "Layer 4 – App<br/>(User Device)"
        APP[📱 Mobile App<br/>iOS / Android]
    end

    SAT -->|1575 MHz| GPS
    GPS -->|NMEA 9600 baud| MCU
    BAT -.->|3.7V| MCU
    MCU -->|18-byte packet| LORA_TX
    LORA_TX -->|915 MHz · 10+ km| LORA_RX
    LORA_RX --> GW
    GW --> GEO
    GEO -->|breach alarm| LORA_TX2[LoRa TX back]
    LORA_TX2 -->|alarm signal| BUZ
    GW --> WIFI
    WIFI -->|HTTPS| FB
    FB --> GCS
    FB --> CF
    CF --> FCM
    FCM -->|WebSocket| APP
```

---

## Data Flow – Normal Tracking

```mermaid
sequenceDiagram
    participant SAT as 🛰 GPS Satellite
    participant GPS as GPS Module
    participant NODE as ESP32 Node
    participant LORA as LoRa Radio
    participant GW as ESP32 Gateway
    participant GEO as Geofence Engine
    participant FB as Firebase
    participant APP as 📱 Mobile App

    SAT->>GPS: NMEA sentences (5 Hz)
    GPS->>NODE: Lat, Lon, Time (9600 baud)
    NODE->>NODE: Parse & validate
    NODE->>LORA: 18-byte packet
    LORA->>GW: 915 MHz (10+ km)
    GW->>GEO: Point-in-polygon check
    GEO-->>GW: ✅ Inside boundary
    GW->>FB: Location update (HTTPS)
    FB->>APP: Real-time sync (WebSocket)
    APP->>APP: Update map marker
```

---

## Data Flow – Breach Detection

```mermaid
sequenceDiagram
    participant NODE as ESP32 Node
    participant LORA as LoRa Radio
    participant GW as ESP32 Gateway
    participant GEO as Geofence Engine
    participant BUZ as 🔔 Buzzer
    participant FB as Firebase
    participant CF as Cloud Functions
    participant FCM as FCM
    participant APP as 📱 Mobile App

    NODE->>LORA: Location packet
    LORA->>GW: 915 MHz
    GW->>GEO: Point-in-polygon check
    GEO-->>GW: ❌ Outside boundary!

    par Alarm Path
        GW->>LORA: Send alarm signal
        LORA->>NODE: Alarm command
        NODE->>BUZ: Activate buzzer (85+ dB)
    and Cloud Path
        GW->>FB: Breach event (HTTPS)
        FB->>CF: Trigger function
        CF->>FCM: Send push notification
        FCM->>APP: 🚨 Breach alert!
        APP->>APP: Show alert + sound
    end
```

---

## Sensor Node Components

```mermaid
graph TD
    subgraph "Livestock Collar (~200g, IP65)"
        A[🛰 GPS Antenna<br/>Ceramic Patch] -->|RF| B[GPS Module<br/>Neo-6M/M9N]
        B -->|NMEA · GPIO 16/17| C[ESP32<br/>Microcontroller]
        C -->|SPI| D[LoRa SX1276<br/>915 MHz · 20 dBm]
        C -->|GPIO 25| E[🔔 Piezo Buzzer<br/>85-100 dB]
        F[🔋 18650 Li-ion<br/>3.7V] -->|TP4056 + AMS1117| C
    end

    style A fill:#e1f5fe
    style B fill:#e1f5fe
    style C fill:#fff3e0
    style D fill:#e8f5e9
    style E fill:#fce4ec
    style F fill:#f3e5f5
```

---

## Gateway Processing

```mermaid
graph TD
    A[LoRa RX<br/>Receive packet] --> B[CRC Check<br/>Validate]
    B --> C{Valid?}
    C -->|No| D[Drop packet]
    C -->|Yes| E[Extract<br/>Node ID · Lat · Lon · Time]
    E --> F[Geofence Engine<br/>Ray-Casting Algorithm]
    F --> G{Inside<br/>boundary?}
    G -->|Yes| H[Log location]
    G -->|No| I[🚨 BREACH]
    I --> J[LoRa TX alarm<br/>→ Node buzzer]
    I --> K[Firebase upload<br/>breach event]
    H --> L[Firebase upload<br/>location update]
    K --> M[Cloud Functions<br/>→ FCM → App]
    L --> N[App map update]

    style I fill:#ff8a80
    style J fill:#ff8a80
    style K fill:#ffcc80
    style H fill:#c8e6c9
```

---

## Cloud Architecture

```mermaid
graph TD
    subgraph "Firebase"
        A[Realtime Database] -->|collections| B["farms · boundaries · nodes<br/>locationHistory · breachEvents · alerts"]
    end

    subgraph "Storage"
        C[Cloud Storage] -->|archive| D["Daily locations · Monthly reports<br/>Parquet analytics · Long-term retention"]
    end

    subgraph "Compute"
        E[Cloud Functions] -->|triggers| F["Boundary validation<br/>Alert distribution<br/>Analytics pipeline"]
    end

    subgraph "Messaging"
        G[FCM] -->|push| H["Breach alerts · Battery warnings<br/>Status updates · Event summaries"]
    end

    A --> C
    A --> E
    E --> G

    style A fill:#ffecb3
    style C fill:#b3e5fc
    style E fill:#c8e6c9
    style G fill:#f8bbd0
```

---

## Mobile App Screens

```mermaid
graph LR
    APP[📱 Mobile App] --> DASH[Dashboard<br/>Live map · Markers<br/>Battery · Signal]
    APP --> BOUND[Boundary Mgmt<br/>Draw polygons<br/>Edit · Calculate area]
    APP --> ALERT[Alerts<br/>Breach notifications<br/>History · Acknowledge]
    APP --> LIVE[Livestock<br/>Add/remove animals<br/>Assign nodes]
    APP --> ANAL[Analytics<br/>Grazing patterns<br/>Movement stats]
    APP --> SET[Settings<br/>Profile · Config<br/>Notifications]

    style DASH fill:#e1f5fe
    style BOUND fill:#e8f5e9
    style ALERT fill:#fce4ec
    style LIVE fill:#fff3e0
    style ANAL fill:#f3e5f5
    style SET fill:#f5f5f5
```

---

## Key Design Decisions

```mermaid
graph TD
    A[Edge Geofencing] -->|✅| A1["< 100ms response<br/>Works offline"]
    A -->|⚠️| A2["Needs boundary<br/>sync from cloud"]

    B[LoRa Communication] -->|✅| B1["10+ km range<br/>Low power · Rural-friendly"]
    B -->|⚠️| B2["Slow data rate<br/>50-250 bps"]

    C[Firebase Realtime] -->|✅| C1["Real-time sync<br/>Simple JSON · Auto backups"]
    C -->|⚠️| C2["Not ideal for<br/>heavy analytics"]

    D[Mobile-First] -->|✅| D1["Easy for farmers<br/>Multi-platform"]
    D -->|⚠️| D2["Needs internet<br/>Phone battery drain"]

    style A fill:#e8f5e9
    style B fill:#e1f5fe
    style C fill:#fff3e0
    style D fill:#f3e5f5
    style A1 fill:#c8e6c9
    style B1 fill:#b3e5fc
    style C1 fill:#ffe0b2
    style D1 fill:#e1bee7
    style A2 fill:#fff9c4
    style B2 fill:#fff9c4
    style C2 fill:#fff9c4
    style D2 fill:#fff9c4
```

---

## Scalability

```mermaid
graph LR
    subgraph "Current"
        NOW[Single Gateway<br/>50-100 nodes<br/>10+ km radius]
    end

    subgraph "Scale Up"
        MULTI[Multi-Gateway<br/>Different frequencies<br/>Mesh networking]
    end

    subgraph "Future"
        FUT[AWS IoT Core<br/>BigQuery Analytics<br/>ML Anomaly Detection]
    end

    NOW --> MULTI --> FUT
```


prepare the document for the above description




