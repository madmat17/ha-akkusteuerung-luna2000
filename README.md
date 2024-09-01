
# Prognosebasierte Steuerung zur schonenden Ladung von Huawei Luna2000 Akkus über HomeAssistant #

**DISCLAMER: Alles auf eigene Gefahr! Ich übernehme keine Verantwortung für Schäden oder Probleme die hiermit entstehen.**
Ich stehe in keinerlei zusammenhang mit Huawei. Dieses Projekt wird in keinster Weise von der Firma Huawei begleitet oder supported.
Desweiteren wird von mir **KEIN SUPPORT** geleistet. Ich teile hier lediglich, was ich für mein Szenario aufgebaut hatte.

Die hier gezeigte Lösung ist sehr stark an die Steuerung von @Optic00 (https://github.com/Optic00/ha-akkusteuerung-sma) angelehnt. Credits für die Vorarbeit gehen an Optic00!

(work in progress, noch nicht vollständig!)

# Einleitung #

Warum sollte eine Steuerung zur schonenden Ladung eines Akkus sinnvoll sein?
Ganz einfach: Damit man länger etwas von seinem Akku hat.

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

Durch die Anpassung der Ladeleistung in Abhängigkeit vom SoC kannst du also die Belastung des Akkus minimieren und seine Lebensdauer verlängern. Es ist jedoch wichtig, dass dies in einem gut ausbalancierten Lade- und Energiemanagementsystem erfolgt, um eine optimale Leistung und Lebensdauer zu gewährleisten.


# Anleitung # 

Was macht das hier eigentlich?

Akkuladesteuerung über den WR via Modbus TCP

Ein Part ist die Reine Akku Lade-/Entladesteuerung die man manuell auswählen kann, der andere Part die Opti-Automatik welche die Ladestärke auf 0.2C (oder einen gewünschte Ladestärke) begrenzt, den Akku morgens erstmal auf 50% lädt und dann pausiert bis die gewünschte Restproduktionsprognose erreicht ist. Dann wird der Akku bis 90% weiter mit 0.2C beladen, danach mit 1kW bis 100%.

Hilfreich ist die Huawei Solar Integration (https://github.com/wlcrs/huawei_solar), welche sehr einfach die benötigten Entitäten und Sensoren zur verfügung stellt, sowie ein Solcast Account für die Prognose der PV-Erträge!

**opti-automatik.yaml** - Hiermit wird über den SHM 2.0 und freigeschaltetem GGC der Akku mittels der weiteren Automation gezielt geladen, pausiert und zuende geladen mit 0.2C bzw. 1kW. 

**sma-se-akku-steuerung.yaml** - Falls man den WR noch direkt ansteuern kann und die letzten Updates nicht hat / die neue Beta Firmware (siehe oben) kann man diese Steuer-Automatik nutzen.

**configuration.yaml** - Eintrag zum Wechselrichter. Den Sensor für die Temperatur bitte mitnehmen, ohne Sensor funktioniert die Modbus Integration nicht zuverlässig.

Wer erstmal nur die reine Akkusteuerung möchte, braucht nur die "sma-se-akku-steuerung.yaml" als Automation anlegen und u.g. Helfer und Überschuss Akkuladung anlegen.

**ToDo:**
- Akku im Winter mindestens 1x die Woche automatisch auf 100% Laden
- Evtl. Ladegeschwindigkeit ab 95-98% auf 500 Watt begrenzen
- SBS Version
- HACS Version für die reine Akkusteuerung
- English Version of this?

Den Eintrag aus der configuration.yaml bei Homeassistant in die gleichnamige einfügen. 

Man benötigt einen Sensor der den möglichen Überschuss für den Akku berechnet und einen für den aktuellen Hausverbrauch. 


**Beispiel: Hausverbrauch:**

    - unique_id: hausverbrauchsleistung
      device_class: power
      state_class: measurement
      name: Hausverbrauchsleistung
      unit_of_measurement: W
      state: >-
        {% set inv_active_power = states('sensor.inverter_active_power')|float(0) %}
        {% set pm_active_power = states('sensor.power_meter_active_power')|float(0) %}
        {% set house_power =  (inv_active_power - pm_active_power)|float(0)|round(0) %}
        {% if house_power > 0 %}
          {{ house_power }}
        {% else %}
          {{ states('sensor.house_consumption_power')|float(0) }}
        {% endif %}
      availability: >-
        {{ (states('sensor.inverter_active_power')|is_number)
            and (states('sensor.power_meter_active_power')|is_number) }}

**Beispiel: Netzbezug**

    - unique_id: power_meter_netzbezug
      device_class: power
      state_class: measurement
      name: "Power meter Netzbezug"
      unit_of_measurement: W
      state: >-
        {% set pm_wirkleistung = states('sensor.power_meter_wirkleistung') | float(0) %}
        {% if pm_wirkleistung < 0 %}
          {{ -pm_wirkleistung }}
        {% else %}
          {{ 0 }}
        {% endif %}

**Beispiel: PV-Überschuss**

    - unique_id: inverter_pv_ueberschuss
      device_class: power
      state_class: measurement
      name: "Inverter PV Überschuss"
      unit_of_measurement: W
      state: "{{ (states('sensor.inverter_eingangsleistung') | float) - (states('sensor.hausverbrauchsleistung') | float) - (states('sensor.power_meter_netzbezug') | float) }}"

**Beispiel: PV-Überschuss für Akku-Ladung (PV-Überschuss zzgl. aktueller Wallbox-Leistung):**

    - unique_id: inverter_maximaler_ueberschuss_fuer_akkuladung
      device_class: power
      state_class: measurement
      name: Maximaler "Inverter Maximaler Überschuss für Akkuladung"
      unit_of_measurement: W
      state: "{{ (states('sensor.inverter_pv_ueberschuss') | float) + (states('sensor.wallbox_ladeleistung')  | float ) }}"


**Hier als Extra zwei Sensoren die Wirkungsgrad und Akku-Zyklen in HA trackien**

    - unique_id: battery_wirkungsgrad_entladung_vs_ladung
      device_class: power_factor
      name: "Battery Wirkungsgrad (Entladung vs. Ladung)"
      unit_of_measurement: %
      state: "{{ ((states('sensor.battery_gesamtentladung') | float) / (states('sensor.battery_gesamtladung') | float) * 100) | round(2) }}"

    - unique_id: battery_kalkulierte_ladezyklen
      name: "Battery kalkulierte Ladezyklen"
      state: "{{ (((states('sensor.battery_gesamtentladung') | float) + (states('sensor.battery_gesamtladung') | float)) / ((states('input_number.battery_speicherkapazitaet') | float) *2)) | round(0) }}"


dieser HA-Helfer zur Auswahl des Akku-Modus muss angelegt werden:

<img width="322" alt="image" src="https://github.com/user-attachments/assets/be8c061f-681d-4acb-bbc4-47f3ca9a9a49">

Dann noch einen Schalter **akku_opti_automatik** ob diese Ladeoptimerung überhaupt laufen darf anlegen:

Vier Input Numbers anlegen, Minimaler Wert 100, maximaler Wert 10000 (Watt) für:

- input_number.akkusteuerung_ladestaerke_soll und 
- input_number.akkusteuerung_entladestaerke_soll
- input_number.akkusteuerung_02c_ladestaerke

Dieser Helfer sollte den Wert oder einen leicht niedrigeren Wert enthalten der die 70% Abregelung enthält. (z.B. 7000 Watt bei einer 10kW Watt Anlage oder leicht drunter, etwa 6800 Watt)

- input_number.akkusteuerung_wr_70proz_ueberschuss_grenze

Dieser Helfer bekommt den Wert der maximalen AC Einspeiseleistung des Wechselrichters. Bei einem SMA STP SE 10.0 also 10000 Watt bzw. leicht drunter etwa 9900. Die Soll-Ladestärke wird dann auf 100 Watt gesetzt und so autoamtisch der AC-Überschuss in den Akku geladen (falls dieser noch nicht voll ist)

- input_number.akkusteuerung_wr_ac_ueberschuss_grenze

für den letzten z.B. 1-100kWh, dies steuert die Schwelle ab wann der Akku von 50% aufwärts geladen wird.

- input_number.akkusteuerung_ab_welchem_restertrag_vollladen

<img width="500" alt="image" src="https://github.com/Optic00/ha-smase-akkusteuerung/assets/20187253/6a1ae098-817a-4029-b732-442eeee4ae6d">

So schaut es im HA aktuell aus. Hab noch einen Dummy-Schalter für den Fall das die Module/Anlage mal ausfällt. Da ist aber noch nichts aktives in der Automation enthalten.

<img width="505" alt="image" src="https://github.com/user-attachments/assets/82cfe4d3-9034-4cf5-953c-65b624f5250e">

    type: vertical-stack
    cards:
      - type: custom:mushroom-title-card
        title: Akkusteuerung SMA STP-SE
      - type: custom:mushroom-chips-card
        chips:
          - type: conditional
            conditions:
              - condition: numeric_state
                entity: sensor.sn_30XXXXXXXX_battery_power_discharge_total
                above: 0.001
            chip:
              type: template
              entity: sensor.sn_30XXXXXXXX_battery_power_discharge_total
              content: '{{( states(entity) | float / 1000) | round(2) }}  kW '
              icon: mdi:battery-minus
              icon_color: red
          - type: conditional
            conditions:
              - condition: numeric_state
                entity: sensor.sn_30XXXXXXXX_battery_power_charge_total
                above: 0.001
            chip:
              type: entity
              entity: sensor.sn_30XXXXXXXX_battery_power_charge_total
              icon: mdi:battery-positive
              icon_color: green
          - type: entity
            entity: sensor.sn_30XXXXXXXX_battery_soc_total
            icon_color: blue
          - type: template
            entity: sensor.byd_12_8_akku_wirkungsgrad_ladung_und_entladung
            content: '{{ states(entity) | round (1)}}% η'
            icon: mdi:vector-difference
            icon_color: orange
          - type: template
            entity: sensor.byd_12_8_akku_zyklen
            content: '{{ states(entity)}}'
            icon: mdi:counter
            icon_color: yellow
          - type: entity
            entity: sensor.sn_30XXXXXXXX_battery_temp_a
            icon: mdi:battery-charging-wireless-outline
          - type: entity
            entity: sensor.sma_stp_se_temperatur
            icon: mdi:power-socket-it
      - type: custom:mushroom-select-card
        entity: input_select.akkusteuerung_sma_wr
        name: Akkusteuerung
        primary_info: name
        secondary_info: last-changed
      - type: tile
        entity: input_boolean.akku_opti_automatik
      - type: tile
        entity: input_boolean.akku_nach_preis_laden
      - type: tile
        entity: input_boolean.pv_module_nicht_verfugbar
        name: PV-Module NA (z.B. Schnee bedeckt)
      - type: horizontal-stack
        cards:
          - type: tile
            entity: sensor.sn_30XXXXXXXX_battery_discharge_total
            name: Entladen Watt
          - type: tile
            entity: sensor.sn_30XXXXXXXX_battery_charge_total
            name: Laden Watt
      - type: entities
        entities:
          - entity: input_number.akkusteuerung_ladestaerke_soll
          - entity: input_number.akkusteuerung_entladestaerke_soll
            name: Entladestärke
          - entity: input_number.akkusteuerung_ab_welchem_restertrag_vollladen
            name: Restladeschwelle
          - entity: input_number.akkusteuerung_02c_ladestaerke
            name: Ladestärke 0.2C
          - entity: input_number.akkusteuerung_wr_ac_ueberschuss_grenze
            name: WR AC-Grenze
          - entity: input_number.akkusteuerung_wr_70proz_ueberschuss_grenze
            name: 70% Grenze
          - entity: sensor.house_battery_runtime_raw
            name: Akkulaufzeit
          - entity: sensor.ueberschuss_pv_watt

