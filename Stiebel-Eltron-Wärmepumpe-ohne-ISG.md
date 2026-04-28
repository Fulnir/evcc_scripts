# SG-Ready Alternative ohne ISG für Stiebel Eltron Wärmepumpe

Für SG-Ready kompatible Stiebel Eltron Wärmepumpen gibt es eine Alternative, um die Heizung über den WPM zu steuern. Anstatt SG-Ready  mit dem ISG zu verwenden um die Heizleistung zu regeln, besteht die Möglichkeit die Puffertemperatur über die WPM Anschlüße X.1.14 und X.1.15 anzusteuern. Der Vorteil dieser Lösung ist, dass keine zusätzliche teure Hardware wie ein ISG benötigt wird und die Steuerung direkt über die WPM Anschlüsse erfolgt. Allerdings ist zu beachten, dass diese Lösung nicht offiziell von Stiebel Eltron unterstützt wird und möglicherweise nicht mit allen Wärmepumpenmodellen kompatibel ist. Es ist daher ratsam, vor der Umsetzung dieser Alternative die Kompatibilität der Wärmepumpe und WPM mit dieser Lösung abzuklären.

> [!NOTE]
> Bei geeigneter Wärmepumpe und Heizungsinstallation auch zum Kühlen bei Solarüberschuß geeignet.

## WPM - Wäarmepumenmanager

Die Anschlüsse X.1.14 und X.1.15 sind analoge Eingänge, die zur Steuerung der Heizung verwendet werden können. X.1.14 dient zur Steuerung des Heiz- oder Kühlbetriebs, während X.1.15 zur Regelung der Puffertemperatur verwendet wird [^1][^2].

### X1.14

Je nach anliegender Spannung beginnt die Wärmepumpe mit dem
Heiz- oder Kühlbetrieb oder die Funktion wird ausgeschaltet [^2].
| Spannung | Auswirkung |
|----------|------------|
| 0-1 V    | AUS        |
| 1-5 V    | Heizen     |
| 5-6 V    | AUS        |
| 6-10 V   | Kühlen     |

### X1.15

Je nach anliegender Spannung wird die Puffertemperatur reguliert.
Der Temperaturbereich wird im WPM eingestellt  2[^2].
| Spannung | Auswirkung |
|----------|------------|
| 1 V      | Minimale Temperatur z.B 15°C |
| 10 V     | Maximale Temperatur z.B 65°C |

> [!TIP]
> Bei 15-65 °C ist die Steuerung über Prozent einfacher. _2% =  1°C_

### WPM Einstellung

Die Einstellung der Puffertemperatur erfolgt über die WPM Software. Hier kann die minimale und maximale Puffertemperatur eingestellt werden. Je nach anliegender Spannung an X1.15 wird die Puffertemperatur entsprechend geregelt. Zum Beispiel könnte bei 1V die minimale Puffertemperatur von 15°C eingestellt sein und bei 10V die maximale Puffertemperatur von 65°C.

#### Konfiguration
Um die Heizung über die WPM Anschlüsse zu steuern, müssen folgende Einstellungen vorgenommen werden:

`Menu>Inbetriebnahme>IO Konfiguration>Eingang X.1.14` auf `Ein`

`Menu>Inbetriebnahme>IO Konfiguration>Eingang X.1.15>Heizen>Temperaturvorgabe 1V` auf `15.0°C`

`Menu>Inbetriebnahme>IO Konfiguration>Eingang X.1.15>Heizen>Temperaturvorgabe 10V` auf `65.0°C`

## Die Hardware

Benötigte Shellys: 2 Stück `Shelly Dimmer 0/1-10V PM Gen3 × 2`, 1 `Shelly Add-On`, `Temperaturfühler` für die Temperaturmessung am Vorlauf.

Die Spannung an den Anschlüssen X.1.14 und X.1.15 kann jeweils mit einem `Shelly Dimmer 0/1-10V PM Gen3` gesteuert werden. Dieser ermöglicht es, die Spannung im Bereich von 1-10V zu regeln, um die gewünschte Heizleistung und Puffertemperatur zu erreichen. Im Gegensatz zu den anderen Shelly Dimmern, liefern diese die benötigte Spannung von 1-10V.

> [!CAUTION]
> ACHTUNG: Arbeiten an 230V sind lebensgefährlich! Also Vorsicht bei der Installation und Verdrahtung. 
> Wem sich die Verdrahtung nicht erschließt sollte unbedingt die Finger davon lassen!


Die Anschlüsse X.1.14 und X.1.15 befinden sich im WPM. Es ist wichtig, die richtigen Anschlüsse zu verwenden und die Verdrahtung sorgfältig durchzuführen, um eine sichere und zuverlässige Steuerung der Heizung zu gewährleisten. Die Belegung der Anschlüße finden sich [hier](https://www.stiebel-eltron.de/toolbox/content/docs/legende/Elektroanschluss_WPM_WPE_de.pdf).

### Puffertemperatur

Da  ohne ISG gearbeitet wird, steht auch die Puffertemperatur nicht mehr zur Verfügung. Um die Regelung entsprechend anzupassen, kann die aktuelle Vorlauftemperatur über einen Temperatursensor gemessen werden, der an einen Shelly angeschlossen ist. Die Regelung kann dann entsprechend angepasst werden, um die gewünschte Heizleistung zu erreichen. Hier wird die Vorlauftemperatur um `1.5°C` erhöht, damit die Temperaturmessung näher an der tatsächlichen Puffertemperatur des WPM liegt.

```yaml
soc:
  source: http
  uri: http://<shelly-addon-ip>/rpc/Shelly.GetStatus
  jq: .["temperature:101"].tC + 1.5
```

### Außentemperatur

Die Außentemperatur kann über eine Wetter-API abgerufen werden, um die Regelung entsprechend anzupassen. Hier wird die aktuelle Außentemperatur von der Open-Meteo API abgerufen. Wer es nicht so mit APIs hat, kann die Außentemperatur auch über einen Temperatursensor messen, der an einen Shelly angeschlossen ist.

```yaml
name: outdoorTemp
type: float
config:
  source: http
  uri: https://api.open-meteo.com/v1/forecast?latitude=<your-home-latitude>1&longitude=<your-home-longitude>2&current=temperature_2m$0
  method: GET
  jq: .current.temperature_2m
```

## Config für `Benutzerdefinierte Heizung` Loadpoint

Wer den Überschuß zum Heizen und zum Kühlen nutzen möchte, muß eventuell die Konfiguration für den `Benutzerdefinierte Heizung` Loadpoint anpassen, oder einen zweiten Loadpoint anlegen..

```yaml
enable:
    source: go
    script: |
      x114x115on := false
      if enable {
        x114x115on = true
      } else {
        x114x115on = false
      }
      // !! Dieses Plug-In nur zum heizen, zum kühlen ein zweites Plug-In verwenden !!
      raumSoll := 22.0
      if ((outdoorTemp - 2.0) >= raumSoll ) {
        // Kühlen
        x114x115on = false
      }
      x114x115on
```

Die Konfiguration für die `Benutzerdefinierte Heizung`:

```yaml
# --- Status der Heizung (C=Heizen, B=Kühlen, A=Aus) ---
status:
  source: http
  method: GET
  uri: http://<shelly-x115-ip>/rpc/Light.GetStatus?id=0
  jq: |
        if .output == true then "C"
        elif .output == false then "B"
        else "A"
        end

# --- Prüfen, ob das Gerät an ist ---
enabled:
  source: http
  method: GET
  uri: http://<shelly-x115-ip>/rpc/Light.GetStatus?id=0
  jq: .output

# --- Ein-/Ausschalten (now/pv/off) ---
enable:
    source: go
    script: |
      x114x115on := false
      if enable {
        x114x115on = true
      } else {
        x114x115on = false
      }
      // !! Dieses Plug-In nur zum heizen, zum kühlen ein zweites Plug-In verwenden !!
      raumSoll := 22.0
      if ((outdoorTemp - 2.0) >= raumSoll ) {
        // Kühlen
        x114x115on = false
      }
      x114x115on
    in:
      - name: outdoorTemp
        type: float
        config:
          source: http
          uri: https://api.open-meteo.com/v1/forecast?latitude=51.11697501056601&longitude=6.158331962470202&current=temperature_2m$0
          method: GET
          jq: .current.temperature_2m
    out:
      - name: x114x115on
        type: bool
        config:
          source: http
          uri: http://<shelly-x114-ip>/rpc/Light.Set
          headers:
            Content-Type: application/json
          body: |
            {
              "id": 0,
              "on": {{ .x114x115on}}
            }
      - name: x114x115on
        type: bool
        config:
          source: http
          uri: http://<shelly-x115-ip>/rpc/Light.Set
          headers:
            Content-Type: application/json
          body: |
            {
              "id": 0,
              "on": {{ .x114x115on}}
            }

maxcurrent:
  source: go
  script: |
    // x.1.14 Muß be 1V-5V liegen, damit die Steuerung über x.1.15 startet.
    // Das sollte direkt einmalig am Shelly Dimmer eingestellt werden. (33% Dimmen)
    raumSoll := 22.0
    steigung := 0.93
    vorlaufSoll := 5.0 + (2.0 * raumSoll + (steigung * 0.5 * (raumSoll - outdoorTemp))) - math.Pow(1.07, (outdoorTemp + 24.0))
    // Den Solarüberschuß zur Erhöhung der Vorlauftemperatur um 5° nutzen.
    // x.1.15 Hier entspricht 1V-10V, 15-65 °C am WPM, Kühlen 11-15 °C (Nur im Sommerbetrieb bei geeigneten Wärmepumpen)
    newPct := vorlaufSoll - 15.0
    // 1% - 100% Entspricht 0-50 °C Temperaturänderung, 2% = 1 °C
    newPct = (2 * newPct)
    if (newPct < 1) {
        newPct = 1
    }
    if (newPct > 100) {
        newPct = 100
    }
    x115Pct := int(newPct)
    x115Pct
  in:
    #- name: vorlaufIst
    #  type: float
    #  config:
    #    source: http
    #    uri: http://<shelly-addon-ip>/rpc/Shelly.GetStatus
    #    jq: .["temperature:101"].tC
    - name: outdoorTemp
      type: float
      config:
        source: http
        uri: https://api.open-meteo.com/v1/forecast?latitude=51.11697501056601&longitude=6.158331962470202&current=temperature_2m$0
        method: GET
        jq: .current.temperature_2m
  out:
    - name: x115Pct
      type: int
      config:
        source: http
        uri: http://<shelly-x115-ip>/rpc/Light.Set
        method: POST
        body: |
          {
            "id": 0,
            "brightness": {{ .x115Pct}},
          }
        headers:
          Content-Type: application/json
    # Set x1.14 1V-5V for heating
    - name: x115Pct
      type: int
      config:
        source: http
        uri: http://<shelly-x114-ip>/rpc/Light.Set
        method: POST
        body: |
          {
            "id": 0,
            "brightness": 33,
          }
        headers:
          Content-Type: application/json

features:
  - integrateddevice
  - heating
icon: heatpump

# Dimming Prozent wird wieder auf die Vorlauftemperatur zurückgerechnet,
# damit die Anzeige etwa der eingestellten Vorlaufteperatur entspricht.
limitsoc:
  source: http
  uri: http://<shelly-x115-ip>/rpc/Shelly.GetStatus
  jq: .["light:0"].brightness / 2 + 15

# Aktuelle Vorlauftemperatur, damit die Regelung entsprechend anpassen kann.
# Gemessen wird die Vorlauftemperatur über einen Temperatursensor, der an einen Shelly angeschlossen ist. (add-on)
soc:
  source: http
  uri: http://<shelly-addon-ip>/rpc/Shelly.GetStatus
  jq: .["temperature:101"].tC

# (Wärmepumpe + Warmwasser) - Warmwasser
# In diesem Fall misst der 3EM Wärmepumpe und Warmwasserbereiter. 
# Die Leistung des Warmwasserbereiters wird über die Leistungsmessung eines Shelly 1PM. Die Leistung wird dann
# von der Gesamtleistung der Wärmepumpe abgezogen, um die Leistung der Wärmepumpe zu erhalten.
power:
  source: calc
  add:
  - source: http
    uri: http:/<shelly-3EM-ip>/rpc/Shelly.GetStatus
    jq: ."em:0".total_act_power
  - source: http
    uri: http://<shelly-1PM-Warmwasser-ip>/rpc/Shelly.GetStatus
    jq: .["switch:0"].apower
    scale: -1



```

## Quellen

[^1]: [Elektroanschluss WPM WPE](https://www.stiebel-eltron.de/toolbox/content/docs/legende/Elektroanschluss_WPM_WPE_de.pdf).

[^2]: Inbertiebnamhme -  Wärmepumpen-Manager WPM Seite 41
