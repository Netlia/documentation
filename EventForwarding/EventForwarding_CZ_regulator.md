## Teplotní regulátor
Reguluje teplotu radiátorů pomocí termostatických hlavic na základě dat měřených teploměrem. Skládá se z několika fyzických zařízení - jedné nebo více termostatických hlavic a teploměru. Standardně zasílá události s naměřenou teplotu a vlhkostí okolního prostředí.

Každou minutu měří teplotu a vlhkost. Po X měření (ve výchozím stavu 10) provede výpočet průměrné hodnoty a odešle událost `measured-humidity-temperature`.

> DeviceType: temperature-regulator

| EventType                                                                 | Popis                                                                      |
|:--------------------------------------------------------------------------|:---------------------------------------------------------------------------|
| [transport](#eventtype-transport)                                         | Přechod do transportního režimu - neaktivní stav s minimální spotřebou.    |
| [tamper](#eventtype-tamper)                                               | Sundání nebo nasazení hlavice, otevření nebo zavření krytu zařízení.       |
| [battery-alert](#eventtype-battery-alert)                                 | Upozornění na nízký stav baterie.                                          |
| [communication-alert](#eventtype-communication-alert)                     | Problém komunikace se zařízením.                                           |
| [physical-device-replaced](#eventtype-physical-device-replaced)           | Nastává při náhradě fyzického zařízení.                                    |
| [measured-humidity-temperature](#eventtype-measured-humidity-temperature) | Naměřené veličiny.                                                         |

### EventType measured-humidity-temperature

Nastává při odeslání naměřené hodnoty.

Dodatečné předávané parametry:

| Parametr    | Typ   | Povinný | Popis            |
|:------------|:------|:--------|:-----------------|
| temperature | float | ano     | naměřená teplota |
| humidity    | float | ano     | naměřená vlhkost |

Ukázka zaslané události:

```yaml
{
    "protocolVersion": 1,
    "deviceId": "d65f1ffb-aa60-4eff-9666-78a93a048b16",
    "deviceType": "temperature-regulator",
    "eventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "eventTime": "2021-05-03T14:25:31.8437511Z",
    "eventType": "measured-humidity-temperature",
    "temperature": 25.5,
    "humidity": 27.5
}
```

### EventType transport

Nastává při přechodu zařízení do transportního režimu, ke kterému dojde po vložení nové baterie nebo pomocí příkazu
zaslaného prostřednictvím downlink API.

Zařízení v transportním režimu má velmi nízkou spotřebu a neposílá žádné zprávy. Pro probuzení zařízení z transportního režimu
je třeba stisknout RESET tlačítko umístěné na plošném spoji.

Ukázka zaslané události:
```yaml
{
    "protocolVersion": 1,
    "deviceId": "d65f1ffb-aa60-4eff-9666-78a93a048b16",
    "physicalDeviceId": "abc123",
    "deviceType": "temperature-regulator",
    "eventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "eventTime": "2021-05-03T14:25:31.8437511Z",
    "eventType": "transport"
}
```

### EventType tamper

Nastává při otevření nebo zavření krytu zařízení, manipulace s bezpečnostním spínačem.

Ukázka zaslané události:

```yaml
{
    "protocolVersion": 1,
    "deviceId": "d65f1ffb-aa60-4eff-9666-78a93a048b16",
    "physicalDeviceId": "abc123",
    "deviceType": "temperature-regulator",
    "eventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "eventTime": "2021-05-03T14:25:31.8437511Z",
    "eventType": "tamper"
}
```

### EventType battery-alert

Upozornění při nízkém stavu baterie. Aktuálně vždy s hodnotou `"batteryStatus": "low"`, v budoucnu bude rozšířeno.

| Parametr         | Typ    | Povinný | Popis                      |
|:-----------------|:-------|:--------|:---------------------------|
| batteryStatus    | string | ano     | Informace o stavu baterie. |

Ukázka zaslané události:

```yaml
{
    "protocolVersion": 1,
    "deviceId": "d65f1ffb-aa60-4eff-9666-78a93a048b16",
    "physicalDeviceId": "abc123",
    "deviceType": "temperature-regulator",
    "eventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "eventTime": "2021-05-03T14:25:31.8437511Z",
    "eventType": "battery-alert",
    "batteryStatus": "low"
}
```

### EventType communication-alert

Upozornění při problému komunikace se zařízením. Nastává v situaci kdy ze zařízení nepřišla zpráva v pravidelném intervalu.

Ukázka zaslané události:

```yaml
{
    "protocolVersion": 1,
    "deviceId": "d65f1ffb-aa60-4eff-9666-78a93a048b16",
    "physicalDeviceId": "abc123",
    "deviceType": "temperature-regulator",
    "eventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "eventTime": "2021-05-03T14:25:31.8437511Z",
    "eventType": "communication-alert"
}
```

### EventType physical-device-replaced

Událost je odeslána při náhradě fyzického zařízení - informuje o tom, že původní fyzické zařízení s `physicalDeviceId` bylo nahrazeno novým fyzickým zařízením s `replacementPhysicalDeviceId`. Může nastat např. po výměně fyzického zařízení které je v poruše za nové.

Dodatečné předávané parametry:

| Parametr                    | Typ    | Povinný | Popis                                                 |
|:----------------------------|:-------|:--------|:------------------------------------------------------|
| replacementPhysicalDeviceId | string | ano     | PhysicalDeviceId nově přiřazeného fyzického zařízení. |

Ukázka zaslané události:

```yaml
{
    "protocolVersion": 1,
    "deviceId": "d65f1ffb-aa60-4eff-9666-78a93a048b16",
    "physicalDeviceId": "abc123",
    "deviceType": "temperature-regulator",
    "eventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "eventTime": "2021-05-03T14:25:31.8437511Z",
    "eventType": "physical-device-replaced",
    "replacementPhysicalDeviceId": "abc123"
}
```

### EventType device-created

Událost informuje o vytvoření zařízení. Součástí jsou informace o fyzických zařízeních ze kterých je složeno a dodatečné informace specifické pro partnera (lze využít např. pro informaci o umístění zařízení).

Dodatečné předávané parametry:

| Parametr                    | Typ      | Povinný | Popis                                                              |
|:----------------------------|:---------|:--------|:-------------------------------------------------------------------|
| physicalDevices             | string[] | ano     | Fyzická zařízení / komponenty ze kterých je zařízení složeno.      |
| customData                  | object   | ne      | Objekt s informacemi specifickými dle partnera.                    |

Mohou existovat zařízení u kterých bude `physicalDevices` obsahovat prázdné pole.

Ukázka zaslané události:

```yaml
{
    "protocolVersion": 1,
    "deviceId": "d65f1ffb-aa60-4eff-9666-78a93a048b16",
    "deviceType": "temperature-regulator",
    "eventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "eventTime": "2021-05-03T14:25:31.8437511Z",
    "eventType": "device-created",
    "physicalDevices": ["abc123", "abc456", "abc789"],
    "customData":
        {
            "sample-key-1": "sample-value-1",
            "sample-key-2": 123 
        }
}
```