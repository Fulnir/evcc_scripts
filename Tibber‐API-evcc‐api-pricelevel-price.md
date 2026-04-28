# Plugin für Tibber-Preislevel

Auch wenn man sich viele Funktionen wünscht, solle `evcc` nicht mit solchen überladen sein. Sollte doch mal mehr 
nötig sein, sind Erweiterungen per `plugin` eine gute Lösung. 

So kommt es vor, das die Heizung oder bestimmte Ladepunkte nicht bei sehr teurem Strom laufen sollen. Auch nicht wenn ein Teil aus Solarüberschuß stammt.
Statt laufendem Luftenfeuchter also lieber die Batterie aufladen.
Oder der Strompreis ist gerade sehr günstig, dann setzt man das `Costlimit` im Ladepunkt auf den aktuellen Preis leicht darüber (`costlimit = limit + 0.01`).
Dann `evcc` anhand des `Costlimit` entscheiden, ob der mode auf `now` umschaltet.

> [!NOTE]
> Anhand der Erfahrungswerte mit der eigenen Installation, weiß man ja welche Werte sinnvoll sind.
> 
> Es ist ja auch möglich andere Werte zu berücksichtigen. Wie weit ist der Wagen geladen, welche Wochentag, Uhrzeit u.s.w

Als Beispiele für Plugins die `evcc` bei der Steuerung im Zusammenhang mit Strompreisen unterstützen, sind 
hier 2 Plugins für sehr preiswerten und teuerem Strom.

![](https://raw.githubusercontent.com/Fulnir/Fulnir/refs/heads/main/tibber-evcc-prices.png)

## Tibber 👍🏻 - Very Cheap

Über die Tibber-API wird der aktuelle Preis und das aktuelle Preislevel abgefragt. Sollte der Preis unter dem im Code festgelegtem Limit und das Preslevelsollte `VERY_CHEAP` sein, 
wird der `smartcostlimit` Wert auf den aktuellen Tibberpreis gesetzt.  Ist der Preislevel nicht mehr sehr preiswert oder der Preis liegt über dem Limit, wird `smartcostlimit` wieder auf `limit` gesetzt.
Siehe: [Tibber-API](https://developer.tibber.com/docs/reference)

### Die Konfiguration für `Tibber 👍🏻`

Als Vorlage wird hier `Benutzerdefinierte Wärmepumpe` verwendet.

```yaml
setmaxpower:
    source: watchdog
    timeout: 600s
    set:
      source: go
      script: |
        limit := 0.11 
        costlimit := limit
        fmt.Printf("🐦‍🔥 pricelevel: %s  \n", pricelevel)
        if (pricelevel == "VERY_CHEAP") {
          if (costlimit > limit) {
            costlimit = limit + 0.01 // Oder leicht erhöhen: limit + 0.03
          }
        }
        fmt.Printf("🐦‍🔥 Set costlimit to: %f ct \n", costlimit)
        costlimit
      in:
        - name: pricelevel
          type: string
          config:
            source: http
            uri: https://api.tibber.com/v1-beta/gql
            method: POST
            headers:
              - content-type: application/json
            body: '{ "query": "{viewer{homes{currentSubscription{priceInfo{current{total,level}}}}}}" }'
            auth:
              type: bearer
              token: <tibber-token>
            jq: .data.viewer.homes.[0].currentSubscription.priceInfo.current.level
        - name: price
          type: float
          config:
            source: http
            uri: https://api.tibber.com/v1-beta/gql
            method: POST 
            headers:
              - content-type: application/json
            body: '{ "query": "{viewer{homes{currentSubscription{priceInfo{current{total,level}}}}}}" }'
            auth:
              type: bearer
              token: <tibber-token>
            jq: .data.viewer.homes.[0].currentSubscription.priceInfo.current.total
      out:
        - name: costlimit # Wall.E
          type: float
          config:
            source: http
            uri: "http://<evcc-ip>:7070/api/loadpoints/5/smartcostlimit/{{.costlimit}}"
            method: POST
        - name: costlimit # Keller
          type: float
          config:
            source: http
            uri: "http://<evcc-ip>:7070/api/loadpoints/1/smartcostlimit/{{.costlimit}}" 
            method: POST
        - name: costlimit # Heizung
          type: float
          config:
            source: http
            uri: "http://<evcc-ip>:7070/api/loadpoints/6/smartcostlimit/{{.costlimit}}"
            method: POST
          
features:
  - integrateddevice
  - heating

icon: compute # compute tool


temp:
  source: go
  script: |
    price * 100.0
  in:
    - name: price
      type: float
      config:
        source: http
        uri: https://api.tibber.com/v1-beta/gql
        method: POST 
        headers:
          - content-type: application/json
        body: '{ "query": "{viewer{homes{currentSubscription{priceInfo{current{total,level}}}}}}" }'
        auth:
          type: bearer
          token: <tibber-token>
        jq: .data.viewer.homes.[0].currentSubscription.priceInfo.current.total

power:
  source: const
  value: 0
energy:
  source: const
  value: 0

```

## Tibber 👎🏻 - Very Expensive

Über die Tibber-API wird das aktuelle Preislevel abgefragt. Sollte dieses `VERY_EXPENSIVE` sein, 
wird der loadpoint mode auf `off` gesetzt. Anschließend wird dieser `mode` an alle gewünschten loadpoints 
per evcc-api geschickt. Ist der Preislevel nicht mehr sehr teuer, wird der mode wieder auf `pv` gesetzt.

### Die Konfiguration für `Tibber 👎🏻`

Als Vorlage wird hier `Benutzerdefinierte Wärmepumpe` verwendet.

```yaml
setmaxpower:
    source: watchdog
    timeout: 600s
    set:
      source: go
      script: |
        fmt.Printf("🐦‍🔥 pricelevel: %s  \n", pricelevel)
        mode := "pv"
        if (pricelevel == "VERY_EXPENSIVE") {
          mode = "off"
        }
        mode
      in:
        - name: pricelevel
          type: string
          config:
            source: http
            uri: https://api.tibber.com/v1-beta/gql
            method: POST 
            headers:
              - content-type: application/json
            body: '{ "query": "{viewer{homes{currentSubscription{priceInfo{current{total,level}}}}}}" }'
            auth:
              type: bearer
              token: <tibber-token>
            jq: .data.viewer.homes.[0].currentSubscription.priceInfo.current.level
        - name: price
          type: float
          config:
            source: http
            uri: https://api.tibber.com/v1-beta/gql
            method: POST
            headers:
              - content-type: application/json
            body: '{ "query": "{viewer{homes{currentSubscription{priceInfo{current{total,level}}}}}}" }'
            auth:
              type: bearer
              token: <tibber-token>
            jq: .data.viewer.homes.[0].currentSubscription.priceInfo.current.total
      out:
        - name: mode # Wall.E
          type: string
          config:
            source: http
            uri: "http://<evcc-ip>:7070/api/loadpoints/5/mode/{{.mode}}" 
            method: POST
        - name: mode # Keller
          type: string
          config:
            source: http
            uri: "http://<evcc-ip>:7070/api/loadpoints/1/mode/{{.mode}}" 
            method: POST
        - name: mode # Heizung
          type: string
          config:
            source: http
            uri: "http://<evcc-ip>:7070/api/loadpoints/6/mode/{{.mode}}" 
            method: POST

features:
  - integrateddevice
  - heating

icon: compute # compute tool

temp:
  source: go
  script: |
    price * 100.0
  in:
    - name: price
      type: float
      config:
        source: http
        uri: https://api.tibber.com/v1-beta/gql
        method: POST
        headers:
          - content-type: application/json
        body: '{ "query": "{viewer{homes{currentSubscription{priceInfo{current{total,level}}}}}}" }'
        auth:
          type: bearer
          token: <tibber-token>
        jq: .data.viewer.homes.[0].currentSubscription.priceInfo.current.total

power:
  source: const
  value: 0
energy:
  source: const
  value: 0
```

> [!NOTE]
> Die beiden Ladepunkte auf `Schnell`stellen.
