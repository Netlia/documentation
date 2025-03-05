## Teplotní regulátor

Reguluje teplotu radiátorů pomocí termostatických hlavic na základě dat měřených teploměrem. Skládá se z několika
fyzických zařízení - jedné nebo více termostatických hlavic a teploměru. Standardně zasílá události s naměřenou teplotu
a vlhkostí okolního prostředí.

Každou minutu měří teplotu a vlhkost. Po X měření provede výpočet průměrné hodnoty a odešle událost
`measured-humidity-temperature`.

## Obsah

Obsah zobrazíte následujícím způsobem:

![content](../images/table-of-contents.webp "Content")

## Základní datové typy

### Čas

#### UTC

Formát:

`2024-10-09T14:12:38.91Z`

Formát je definován specifikací [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601). `Z` na konci označuje ZULU časové pásmo (+00:00).

### UUID

Formát:

`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`


## Základní parametery

Většina zasílaných událostí obsahuje společné parametery kterými jsou:

| Parametr        | Typ              | Povinný | Popis                                                 |
|:----------------|:-----------------|:--------|:------------------------------------------------------|
| protocolVersion | int              | ano     | Verze protokolu (aktuálně vždy 1)                     |
| deviceId        | string (UUID)    | ne      | Id zařízení které vyvolalo event                      |
| deviceType      | string           | ano     | Typ zařízení (aktuálně vždy temperature-regulator)    |
| eventId         | string (UUID)    | ano     | Id události - slouží primárně k zajištění idempotence |
| eventTime       | string (UTC čas) | ano     | Čas kdy nastala událost                               |
| eventType       | string           | ano     | Typ události                                          |

Ukázka základních parametrů:

```yaml
{
    "protocolVersion": 1,
    "deviceId": "d65f1ffb-aa60-4eff-9666-78a93a048b16",
    "deviceType": "temperature-regulator",
    "eventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "eventTime": "2024-10-09T14:12:38.91Z",
    "eventType": "type-of-event",
}
```
## Eventy

### EventType measured-humidity-temperature

Informuje o změření teploty a vlhkosti.

Dodatečné předávané parametry:

| Parametr    | Typ   | Povinný | Popis                 |
|:------------|:------|:--------|:----------------------|
| temperature | float | ano     | naměřená teplota [°C] |
| humidity    | float | ano     | naměřená vlhkost [%]  |

Ukázka zaslané události:

```yaml
{
    "protocolVersion": 1,
    "deviceId": "d65f1ffb-aa60-4eff-9666-78a93a048b16",
    "deviceType": "temperature-regulator",
    "eventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "eventTime": "2024-10-09T14:12:38.91Z",
    "eventType": "measured-humidity-temperature",
    "temperature": 25.5,
    "humidity": 27.5
}
```

### EventType thermo-head-changed-position

Nastává při změně polohy jedné nebo více hlavic přiřazených k regulátoru (hlavice v jedné místnosti).

Dodatečné předávané parametry:

| Parametr            | Typ                 | Povinný | Popis                        |
|:--------------------|:--------------------|:--------|:-----------------------------|
| positionInformation | positionInformation | ano     | informace o pozicích hlavice |

objekt `positionInformation` obsahuje informaci o jedné nebo více změnách pozice a má následující formát:

| Parametr  | Typ        | Povinný | Popis                        |
|:----------|:-----------|:--------|:-----------------------------|
| positions | position[] | ano     | informace o pozicích hlavice |

objekt `position` má formát:

| Parametr         | Typ         | Povinný | Popis                                                                                                                  |
|:-----------------|:------------|:--------|:-----------------------------------------------------------------------------------------------------------------------|
| physicalDeviceId | string      | ano     | id fyzického zařízení                                                                                                  |
| newPosition      | int (0-100) | ano     | nová pozice. Hodonta je udávaná v procentech 0-100 kde 0 značí, že hlavice je zavřená a do radiátoru neteče horká voda |

Ukázka zaslané události:

```yaml
{
    "protocolVersion": 1,
    "deviceId": "d65f1ffb-aa60-4eff-9666-78a93a048b16",
    "deviceType": "temperature-regulator",
    "eventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "eventTime": "2024-10-09T14:12:38.91Z",
    "eventType": "thermo-head-changed-position",
    "positionInformation": {
        "positions":
        [
            {
                physicalDeviceId: "abc123",
                newPosition: 25,
            },
            {
                physicalDeviceId: "bcd123",
                newPosition: 30,
            }
        ]
    }
}
```

### EventType battery-alert

Upozornění při změně stavu baterie. Aktuálně posílá hodnotou `"batteryStatus": "low"` v případě téměř vybité baterie
nebo
`"batteryStatus": "high"` v případě vložení nabité baterie. V budoucnu může být rozšířeno o další hodnoty.

| Parametr      | Typ    | Povinný | Popis                      |
|:--------------|:-------|:--------|:---------------------------|
| batteryStatus | string | ano     | Informace o stavu baterie. |

Ukázka zaslané události:

```yaml
{
    "protocolVersion": 1,
    "deviceId": "d65f1ffb-aa60-4eff-9666-78a93a048b16",
    "physicalDeviceId": "abc123",
    "deviceType": "temperature-regulator",
    "eventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "eventTime": "2024-10-09T14:12:38.91Z",
    "eventType": "battery-alert",
    "batteryStatus": "low"
}
```

### EventType warning

Upozornění na stav zařízení nevyžadující okamžité řešení. Příkladem může být horší kvalita signálu, zhoršené mechanické
vlastnosti ventilu (tuhnutí), apod...

| Parametr                    | Typ    | Povinný | Popis                                             |
|:----------------------------|:-------|:--------|:--------------------------------------------------|
| warningType                 | string | ano     | Označení upozornění, unikátní v rámci deviceType. |
| localizedWarningDescription | string | ano     | Vysvětlení příčiny upozornění.                    |

Ukázka zaslané události:

```yaml
{
    "protocolVersion": 1,
    "deviceId": "d65f1ffb-aa60-4eff-9666-78a93a048b16",
    "physicalDeviceId": "abc123", // může být null
    "deviceType": "temperature-regulator",
    "eventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "eventTime": "2024-10-09T14:12:38.91Z",
    "eventType": "warning",
    "warningType": "some-warning",
    "localizedWarningDescription": "Popis zaslaného upozornění."
}
```

Aktuálně podporované warningType:
- replacedBatteryNotFull
  - localizedWarningDescription: "Vložena částečně vybitá baterie."

### EventType error

Upozornění na stav zařízení vyžadující okamžité řešení z důvodu neschopnosti jeho dalšího fungování které bude nutné
pravděpodobně řešit jeho výměnou. Příkladem může být hardwarový problém.

| Parametr                  | Typ    | Povinný | Popis                                        |
|:--------------------------|:-------|:--------|:---------------------------------------------|
| errorType                 | string | ano     | Označení chyby, unikátní v rámci deviceType. |
| localizedErrorDescription | string | ano     | Vysvětlení příčiny chyby.                    |

Ukázka zaslané události:

```yaml
{
    "protocolVersion": 1,
    "deviceId": "d65f1ffb-aa60-4eff-9666-78a93a048b16",
    "physicalDeviceId": "abc123", // může být null
    "deviceType": "temperature-regulator",
    "eventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "eventTime": "2024-10-09T14:12:38.91Z",
    "eventType": "error",
    "errorType": "some-error",
    "localizedErrorDescription": "Popis zaslané chyby."
}
```

Aktuálně podporované errorType:
- replacedBatteryStatusLow
  - localizedWarningDescription: "Vložena vybitá baterie."

### EventType user-requested-temperature-change

Událost je odeslána když uživatel klikne na tlačítko na termostatu -
uživatel tímto zmáčknutím vyžaduje snížení nebo zvýšení teploty.

Pokud 'temperature-regulator' nebosahuje fyzické zařízení 'termostat' tak se událost neodesílá.

Dodatečné předávané parametry:

| Parametr                   | Typ    | Povinný | Popis                                                                                   |
|:---------------------------|:-------|:--------|:----------------------------------------------------------------------------------------|
| requestedTemperatureChange | string | ano     | Může nabývat hodnot - increase, decrease. V budoucnu může být rozšířeno o další hodnoty |

Ukázka zaslané události:

```yaml
{
    "protocolVersion": 1,
    "deviceId": "d65f1ffb-aa60-4eff-9666-78a93a048b16",
    "physicalDeviceId": "abc123", 
    "deviceType": "temperature-regulator",
    "eventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "eventTime": "2024-10-09T14:12:38.91Z",
    "eventType": "user-requested-temperature-change",
    "requestedTemperatureChange": "increase",
}
```

### EventType physical-device-replaced

Událost je odeslána při náhradě fyzického zařízení - informuje o tom, že původní fyzické zařízení s
`replacedPhysicalDeviceId` bylo nahrazeno novým fyzickým zařízením s `replacementPhysicalDeviceId`. Může nastat např. po
výměně fyzického zařízení které je v poruše za nové.

Dodatečné předávané parametry:

| Parametr                    | Typ    | Povinný | Popis                                                 |
|:----------------------------|:-------|:--------|:------------------------------------------------------|
| replacedPhysicalDeviceId    | string | ano     | PhysicalDeviceId původního fyzického zařízení.        |
| replacementPhysicalDeviceId | string | ano     | PhysicalDeviceId nově přiřazeného fyzického zařízení. |

Ukázka zaslané události:

```yaml
{
    "protocolVersion": 1,
    "deviceId": "d65f1ffb-aa60-4eff-9666-78a93a048b16",
    "deviceType": "temperature-regulator",
    "eventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "eventTime": "2024-10-09T14:12:38.91Z",
    "eventType": "physical-device-replaced",
    "replacedPhysicalDeviceId": "abc123",
    "replacementPhysicalDeviceId": "abc456"
}
```

### EventType temperature-regulator-created

Událost informuje o vytvoření temperature-regulatoru. Součástí jsou informace o fyzických zařízeních ze kterých je složeno a dodatečné
informace specifické pro partnera.

Dodatečné předávané parametry:

| Parametr        | Typ             | Povinný | Popis                                                         |
|:----------------|:----------------|:--------|:--------------------------------------------------------------|
| physicalDevices | PhysicalDevices | ano     | Fyzická zařízení / komponenty ze kterých je zařízení složeno. |
| note      | string          | ne      | Poznámka zadaná člověkem který zařízení instaloval.               |
| roomId      | string (uuid)          | ano      | Identifikátor místnosti ve které se zařízení nachází. Informace o místnostech by měla dodat netlia ještě před instalací.                |

Objekt PhysicalDevices je definován následujícím způsobem:
| Parametr | Typ | Povinný | Popis |
|:----------------------------|:---------|:--------|:-------------------------------------------------------------------|
| Data | PhysicalDevice[] | ano | Pole objektů obsahující informace o jednotlivých zařízeních. |

Definice PhysicalDevice:

| Parametr           | Typ    | Povinný | Popis                                                                   |
|:-------------------|:-------|:--------|:------------------------------------------------------------------------|
| physicalDeviceId   | string | ano     | Id fyzického zařízení.                                                  |
| physicalDeviceType | string | ano     | Typ fyzického zařízení. Může nabívat hodnot - thermo-head a thermometer |

Ukázka zaslané události:

```yaml
{
    "protocolVersion": 1,
    "deviceId": "d65f1ffb-aa60-4eff-9666-78a93a048b16",
    "deviceType": "temperature-regulator",
    "eventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "eventTime": "2024-10-09T14:12:38.91Z",
    "eventType": "device-created",
    "note": string
    "physicalDevices":
    {
        "data":
        [
            {
                "physicalDeviceId": "abc123",
                "physicalDeviceType": "thermo-head"
            },
            {
                "physicalDeviceId": "abc456",
                "physicalDeviceType": "thermo-head"
            },
            {
                "physicalDeviceId": "abc789",
                "physicalDeviceType": "thermometer"
            }
        ]
    },
    "note": "Místnost je vyhřívána převážně z okolních místností",
    "roomId": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
}
```

### EventType heating-state-changed

Událost je odslána při změně cílové teploty a také při změně situace topení (např. ukončení předehřívání).
Cílová teplota se může změnit z několika důvodů - požadavek na změnu teploty, předehřívání místnosti,
změna teploty kvůli plánu atd.

Dodatečné předávané parametry:

| Parametr          | Typ    | Povinný | Popis                                                                                       |
|:------------------|:-------|:--------|:--------------------------------------------------------------------------------------------|
| targetTemperature | float  | ano     | Teplota které se aplikace snaží nově dosáhnout/udržovat.                                    |
| changeReason      | string | ano     | Může nabývat hodnot - `pre-heating-started`, `pre-heating-stopped`, `target-temperature-changed`. |

Ukázka zaslané události:

```yaml
{
    "protocolVersion": 1,
    "deviceId": "d65f1ffb-aa60-4eff-9666-78a93a048b16",
    "deviceType": "temperature-regulator",
    "eventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "eventTime": "2024-10-09T14:12:38.91Z",
    "eventType": "heating-state-changed",
    "targetTemperature": 25.5,
    "changeReason": "pre-heating-started"
}
```

### EventType physical-device-attached
Událost je odeslána při přiřazení fyzického zařízení k již existujícímu zařízení. Například, když je k regulátoru teploty přiřazena nová termostatická hlavice nebo teploměr.

Dodatečné předávané parametry:

| Parametr          | Typ    | Povinný | Popis                                                                                       |
|:------------------|:-------|:--------|:--------------------------------------------------------------------------------------------|
| attachedPhysicalDeviceId|	string|	ano|	Id přiřazeného fyzického zařízení. |
| physicalDeviceType|	string (enum)|	ano|	Typ přiřazeného fyzického zařízení. Může nabývat hodnot - thermo-head a thermometer|

Ukázka zaslané události:

```yaml
{
    "protocolVersion": 1,
    "deviceId": "d65f1ffb-aa60-4eff-9666-78a93a048b16",
    "deviceType": "temperature-regulator",
    "eventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "eventTime": "2024-10-09T14:12:38.91Z",
    "eventType": "physical-device-attached-to-device",
    "attachedPhysicalDeviceId": "def456",
    "physicalDeviceType": "thermo-head"
}
```

### EventType physical-device-detached
Událost je odeslána při odpojení fyzického zařízení od zařízení. Například, když je od regulátoru teploty odpojena termostatická hlavice nebo teploměr. Důvodem odpojení může být například porucha zařízení, vyjmutí z důvodu údržby nebo trvalé odebrání zařízení.

Dodatečné předávané parametry:

| Parametr          | Typ    | Povinný | Popis                                                                                       |
|:------------------|:-------|:--------|:--------------------------------------------------------------------------------------------|
|detachedPhysicalDeviceId|	string|	ano|	Id odpojeného fyzického zařízení.|

Ukázka zaslané události:

```yaml
{
    "protocolVersion": 1,
    "deviceId": "d65f1ffb-aa60-4eff-9666-78a93a048b16",
    "deviceType": "temperature-regulator",
    "eventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "eventTime": "2024-10-09T14:12:38.91Z",
    "eventType": "physical-device-detached-from-device",
    "detachedPhysicalDeviceId": "def456"
}
```
