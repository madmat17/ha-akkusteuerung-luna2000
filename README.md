
# Prognosebasierte Steuerung zur schonenden Ladung von Huawei Luna2000 Akkus über HomeAssistant #

**DISCLAMER: Alles auf eigene Gefahr! Ich übernehme keine Verantwortung für Schäden oder Probleme die hiermit entstehen.**
Ich stehe in keinerlei zusammenhang mit Huawei. Dieses Projekt wird in keinster Weise von der Firma Huawei begleitet oder supported.
Desweiteren wird von mir **KEIN SUPPORT** geleistet. Ich teile hier lediglich, was ich für mein Szenario aufgebaut hatte.

Die hier gezeigte Lösung ist sehr stark an die Steuerung von @Optic00 (https://github.com/Optic00/ha-akkusteuerung-sma) angelehnt. Credits für die Vorarbeit gehen an Optic00!

(work in progress, noch nicht vollständig!)

## Einleitung 

Warum sollte eine Steuerung zur schonenden Ladung eines Akkus sinnvoll sein?
Ganz einfach: Damit man länger etwas von seinem Akku hat.

### Hintergrund: 
Beim Luna2000 von Huawei handelt es sich um einen Lithium-Ionen-Akku, welcher die Eigenschaft hat, dass verschiedene Ladegeschwindigkeiten (bzw. Ladeströme) den Akku stressen und schneller altern lassen können.
Das heißt, dass man die Lebensdauer es Akkus positiv beeinflussen kann, wenn man die Ladeleistung abhängig vom jeweiligen SoC (= State of Charge, Ladezustand) regelt. Dabei sind folgende Eckpunkte zu berücksichtigen:

#### 1. **Schonendes Laden bei hohen SoC**: 
Wenn der Akku bereits einen hohen Ladezustand (z.B. über 80 %) hat, ist es ratsam, die Ladeleistung zu reduzieren. Das Laden bei hohen SoC-Werten erzeugt mehr Stress auf die Elektroden des Akkus, was die Degradation beschleunigen kann.

#### 2. **Höhere Ladeleistung bei niedrigem SoC**: 
Bei niedrigem SoC (z.B. unter 30 %) kann der Akku normalerweise höhere Ladeleistungen vertragen, ohne dass dies die Lebensdauer stark beeinträchtigt. In diesem Bereich ist der Innenwiderstand geringer, und der Akku kann effizienter laden.

#### 3. **Vermeiden von 100 %-Ladungen**:
Li-Ionen-Akkus sollten idealerweise nicht ständig auf 100 % geladen werden, da dies die Degradation beschleunigen kann. Eine Ladegrenze von etwa 80–90 % kann die Lebensdauer verlängern.

#### 4. **Reduzierte Ladeleistung bei niedrigem SoC**:
Wenn der SoC sehr niedrig ist (z.B. unter 10 %), ist es ebenfalls ratsam, mit einer niedrigeren Ladeleistung zu laden, um die Schädigung der Batterie durch hohe Stromstärken zu vermeiden.

#### 5. **Temperaturabhängiges Laden**:
Die Ladeleistung sollte auch abhängig von der Temperatur geregelt werden. Bei niedrigen Temperaturen ist es sinnvoll, die Ladeleistung zu reduzieren, da der Akku dann empfindlicher ist.

#### 6. **Adaptive Ladeverfahren**:
Einige moderne Ladegeräte und Akkumanagementsysteme nutzen adaptive Ladeverfahren, die den Ladeprozess dynamisch an den aktuellen SoC und die Temperatur anpassen, um die Lebensdauer des Akkus zu maximieren.

### Empfohlene Ladeleistungen: 
Die Wahl des richtigen C-Werts, also des Verhältnisses zwischen Ladeleistung und Nennkapazität des Akkus, ist entscheidend, um die Lebensdauer des Akkus zu maximieren. 0,2C zB sind 20% der Nennkapazität. Beträgt die Nennkapazität 5.000 Wh, dann entspricht die 0,2C einer Ladeleistung von 1.000 W.
Empfehlungen für die jeweiligen Ladestände:

#### 1. **SoC von 0 % bis 10 %:**

-   **Empfohlener C-Wert:** **0,2C bis 0,3C**
-   **Begründung:** In diesem Bereich ist der Akku empfindlicher, daher ist eine moderate Ladeleistung wichtig, um die Lebensdauer zu schonen, während dennoch eine akzeptable Ladegeschwindigkeit erreicht wird.

#### 2. **SoC von 10 % bis 30 %:**

-   **Empfohlener C-Wert:** **0,3C bis 0,5C**
-   **Begründung:** Der Akku kann in diesem SoC-Bereich eine höhere Ladeleistung vertragen, ohne dass dies die Lebensdauer wesentlich beeinträchtigt. 0,5C bietet eine gute Balance zwischen schneller Ladung und Schonung des Akkus.

#### 3. **SoC von 30 % bis 80 %:**

-   **Empfohlener C-Wert:** **0,5C bis 0,7C**
-   **Begründung:** Hier kann der Akku effizient und relativ schnell geladen werden. Ein C-Wert von bis zu 0,7C bietet eine gute Ladegeschwindigkeit und ist immer noch schonend für den Akku.

#### 4. **SoC von 80 % bis 100 %:**

-   **Empfohlener C-Wert:** **0,2C bis 0,3C**
-   **Begründung:** In diesem Bereich wird die Ladegeschwindigkeit typischerweise reduziert, um die Lebensdauer zu maximieren. Eine geringere Ladeleistung hilft, die Degradation zu minimieren, die bei hohen SoC-Werten stärker auftritt.

#### Hinweis zur Temperaturabhängigkeit:

-   **Bei niedrigen Temperaturen (< 10°C):** C-Wert um 0,1C bis 0,3C senken.
-   **Bei hohen Temperaturen (> 35°C):** ebenfalls C-Wert reduzieren, um Überhitzung zu vermeiden.

### Umsetzung:
**Ladeleistungen:**  Diese können wir beeinflussen, indem wir die Ladeleistung über Modbus begrenzen.

**Vermeiden von 100%-Ladungen:** Das ist ein zweischneidiges Thema. Der Luna2000 benötigt immer wieder eine Ladung auf 100%, um sich kalibrieren zu können. Was eher sehr schädlich für die Li-On-Akkus ist, ist ein permanenter Ladestand auf 100%. Daher regelt diese Automation das Vollladen des Akkus so, dass von 80% auf 100% erst gegen Ende des Tages (= Sonnenstunden) geladen wird

**Temperatur:** Auf die Temperatur kann man kaum eingehen (eine optimale Ladetemperatur läge bei <35°C), da der Luna2000 scheinbar nicht die Zelltemperatur, sondern die Temperatur der internen Ladeelektronik meldet. Da der Luna2000 zudem aktiv gekühlt ist, erlaube ich mir das Temperatur-Thema zu vernachlässigen


# Anleitung # 

Was macht die Automation in diesem Zusammenhang?

Akkuladesteuerung über den WR via Modbus TCP in Abhängigkeit gesetzter Parameter und dem noch zu erwartenden PV-Ertrag (bzw. dem zu erwartetenden Überschuss aus dem PV-Ertrag).

Hierbei können die folgenden Parameter eingestellt werden...

**Automatische Akkusteuerung AN/AUS:** Definiert, ob die Automatik überhaupt genutzt werden soll

**Wallbox ignorieren:** Gibt an, ob die Ladeleistung einer ggf. genutzten Wallbox aus den Werten herausgerechnet werden soll. Das ist besonders dann sinnvoll, wenn die Wallbox selbst basierend auf Ertrag und Netzbezug/-einspeisung gesteuert wird und dem Hausakku in dieser Steuerung Vorrang eingeräumt wird.

**Wöchentlich vollladen:** Mit dieser Option wird der Akku under Zuhilfenahme von Netzstrom vollgeladen, falls die letzte Vollladung 7 Tage zurückliegt. Das kann u.U. in den Wintermonaten gut und sinnvoll sein. Mit der Zusatzoption des **preisoptimierten Vollladens** geschieht das Vollladen mit Netzstrom zu den Zeitpunkten, an denen der Strom besonders günstig ist (das ist vor allem bei stündlichen Tarifen interessant).

**Schonungsmodus:** Über den Schonungsmodus wird festgelegt, welche C-Werte (Ladeleistung in Abhängig zur Nennkapazität) bei welchem SoC zum Einsatz kommt. Zu Auswahl stehen "schonend" und "sehr schonend".


## **Abhängigkeiten & Inhalte**
Folgende Integrationen werden benötigt:

 - Huawei Solar Integration (https://github.com/wlcrs/huawei_solar)
 - HA Solcast PV Solar Forecast Integration (https://github.com/BJReplay/ha-solcast-solar)

**/packages/akkusteuerung-entities.yaml** - Beinhaltet die Entitäten, welche für die Akkusteuerung benötigt werden. 
**/packages/akkusteuerung-automation.yaml** - Beinhaltet die Automation zur Steuerung.

**[/collaterals/diagramm.pdf](/collaterals/diagramm.pdf)** - Stellt die Steuerlogik dar

Ein Controldashboard (u.a. unter Verwendung der HA-Badges) könnte wie folgt aussehen:
<img src="https://github.com/madmat17/ha-akkusteuerung-luna2000/blob/main/collaterals/dashboard.jpg" height="500px" width="703px">



    icon: mdi:battery-heart-variant
    path: akkusteuerung
    title: Akkusteuerung
    cards:
      - type: vertical-stack
        cards:
          - type: entities
            entities:
              - entity: input_boolean.battery_akkusteuerung_automatik
                name: Automatische Akkusteuerung
                secondary_info: last-changed
                icon: mdi:refresh-auto
            state_color: true
          - type: entities
            entities:
              - entity: input_boolean.battery_akkusteuerung_wallbox_ignorieren
                name: Wallbox ignorieren
                secondary_info: |
                  {{ states('sensor.battery_akkusteuerung_pv_uberschuss') }}
            state_color: true
            visibility:
              - condition: state
                entity: input_boolean.battery_akkusteuerung_automatik
                state: "on"
          - type: entities
            entities:
              - entity: input_boolean.battery_akkusteuerung_woechentlich_vollladen
                name: Wöchentlich vollladen
                secondary_info: last-changed
                icon: mdi:calendar-clock-outline
            state_color: true
            visibility:
              - condition: state
                entity: input_boolean.battery_akkusteuerung_automatik
                state: "on"
          - type: entities
            entities:
              - entity: input_boolean.battery_akkusteuerung_vollladung_preisoptimiert
                name: Preisoptimiert volladen
                secondary_info: last-changed
                icon: mdi:currency-eur
            state_color: true
            visibility:
              - condition: state
                entity: input_boolean.battery_akkusteuerung_automatik
                state: "on"
              - condition: state
                entity: input_boolean.battery_akkusteuerung_woechentlich_vollladen
                state: "on"
          - type: entities
            entities:
              - entity: input_select.battery_akkusteuerung_schonungsmodus
                name: Schonungsmodus
            state_color: false
          - type: horizontal-stack
            cards:
              - type: entity
                entity: sensor.battery_gesamtentladung
                name: Gesamtentladung
                state_color: false
              - type: entity
                entity: sensor.battery_gesamtladung
                name: Gesamtladung
          - type: entities
            entities:
              - entity: sensor.battery_akkusteuerung_pv_uberschuss
                name: PV-Überschuss
              - entity: sensor.battery_akkusteuerung_pv_uberschuss_ohne_wallbox
                name: PV-Überschuss (exkl. Wallbox)
              - entity: sensor.battery_akkusteuerung_soll_ladeleistung
                name: Ladeleistung Soll
                icon: mdi:bullseye-arrow
              - entity: sensor.battery_ladeleistung
                name: Ladeleistung Aktuell
                icon: mdi:bullseye
              - entity: sensor.battery_akkusteuerung_restladezeit_hh_mm
                name: Restladezeit
                icon: mdi:battery-clock-outline
              - entity: sensor.battery_akkusteuerung_tage_seit_100_soc
                name: Tage seit 100% SoC
            state_color: false
    badges:
      - type: entity
        entity: sensor.battery_lade_entladeleistung
        show_entity_picture: false
        color: green
      - type: entity
        entity: sensor.battery_batterieladung
        icon: mdi:battery-high
      - type: entity
        entity: sensor.battery_wirkungsgrad_entladung_vs_ladung
        icon: mdi:vector-difference
        color: orange
        state_content: "{{ states(entity) | round (1)}}% η"
      - type: entity
        entity: sensor.battery_kalkulierte_ladezyklen
        icon: mdi:counter
        color: yellow
      - type: entity
        entity: sensor.battery_battery_1_temperature
        icon: mdi:battery-alert-variant
        color: light-grey
      - type: entity
        entity: sensor.inverter_interne_temperatur
        icon: mdi:power-socket-it
        color: light-grey

