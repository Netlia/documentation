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

Formát je definován specifikací [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601). `Z` na konci označuje ZULU časové
pásmo (+00:00).

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

### EventType device-failure

Upozornění na chybu, která nastala na zařízení. Příkladem může být vložení vybité baterie nebo HW chyba na zařízení.

| Parametr                    | Typ                             | Povinný | Popis                                                                                           |
|:----------------------------|:--------------------------------|:--------|:------------------------------------------------------------------------------------------------|
| physicalDeviceId            | string                          | ne      | ID fyzického zařízení, na kterém chyba nastala. Pokud je `null`, chyba se týká celého zařízení. |
| failureType                 | string (výčet níže)             | ano     | Typ selhání. Výčet možných typů je uveden níže.                                                 |
| severity                    | string (`warning` nebo `error`) | ano     | Označení závažnosti. Vysvětleno níže.                                                           |
| localizedWarningDescription | string                          | ano     | Popis selhání v jazyce vybraném partnerem.                                                      |
| IsResolvableByPartner       | bool                            | ano     | Značí, zda je možné chybu vyřešit z partnerské aplikace. Podrobné vysvětlení níže.              |

**`error`** - Značí chyby, které by měly být vyřešeny co nejrychleji, jelikož mohou způsobit nefunkčnost zařízení.
**`warning`** - Značí chyby, o kterých je dobré vědět, ale není nutně potřeba hned něco dělat.

**`IsResolvableByPartner`** - Značí, zda je možné chybu vyřešit z partnerské aplikace. Pokud je `true`, měl by mít
uživatel možnost chybu vyřešit, například stisknutím tlačítka (např. křížku) ve vaší aplikaci. Pokud je `false`, tak
problém buď nejde vůbec vyřešit (je potřeba výměna zařízení), nebo je nutné provést nějakou akci se zařízením (např.
vyměnit baterii). Pokud naše aplikace zjistí, že byl problém vyřešen (např. výměnou baterie), odesílá event
`failure-resolved`.
V některých případech je možné, že selhání může vyřešit jak uživatel, tak naše aplikace.

Pro vyřešení selhání je možné použít endpoint **PUT api/device-failure/resolve**.

Aktuálně podporované typy selhání (`failureType`), jejich závažnost (`severity`) a příznak `IsResolvableByPartner`:

| failureType                             | severity  | IsResolvableByPartner                                                    | Popis                                                                               |
|:----------------------------------------|:----------|:-------------------------------------------------------------------------|:------------------------------------------------------------------------------------|
| `inserted-discharged-battery`           | `error`   | `true` (může jít o false positive, proto umožňujeme vyřešení uživatelem) | Do zařízení byla vložena vybitá baterie.                                            |
| `inserted-partially-discharged-battery` | `warning` | `true` (může jít o false positive, proto umožňujeme vyřešení uživatelem) | Do zařízení byla vložena jen částečně nabitá baterie.                               |
| `generic-physical-device-error`         | `error`   | `false`                                                                  | Obecná chyba informující, že fyzické zařízení je nefunkční a je potřeba ho vyměnit. |

Ukázka zaslané události:

```yaml
{
  "protocolVersion": 1,
  "deviceId": "d65f1ffb-aa60-4eff-9666-78a93a048b16",
  "deviceType": "temperature-regulator",
  "eventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
  "eventTime": "2024-10-09T14:12:38.91Z",
  "physicalDeviceId": "abc123", // může být null
  "eventType": "device-failure",
  "failureType": "generic-physical-device-error",
  "severity": "error",
  "localizedWarningDescription": "Došlo k závadě na zařízení, proveďte prosím jeho výměnu.",
  "IsResolvableByPartner": false
}
```

### EventType device-failure-resolved

Doporučujeme nejdříve prostudovat event typu `device-failure`, jelikož je úzce spjatý s logikou tohoto eventu.

Událost je odeslána, když naše aplikace detekuje vyřešení selhání. Pokud je selhání vyřešeno partnerem pomocí endpointu
**PUT api/device-failure/resolve**, tento event se neodesílá.

| Parametr         | Typ                                      | Povinný | Popis                                                                                                 |
|:-----------------|:-----------------------------------------|:--------|:------------------------------------------------------------------------------------------------------|
| physicalDeviceId | string                                   | ne      | ID fyzického zařízení, na kterém chyba nastala. Pokud je `null`, byla vyřešena chyba celého zařízení. |
| failureType      | string (výčet u eventu `device-failure`) | ano     | Typ selhání.                                                                                          |

Ukázka zaslané události:

```yaml
{
  "protocolVersion": 1,
  "deviceId": "d65f1ffb-aa60-4eff-9666-78a93a048b16",
  "deviceType": "temperature-regulator",
  "eventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
  "eventTime": "2024-10-09T14:12:38.91Z",
  "physicalDeviceId": "abc123", // může být null
  "eventType": "device-failure-resolved",
  "failureType": "inserted-discharged-battery"
}
```

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

Událost informuje o vytvoření temperature-regulatoru. Součástí jsou informace o fyzických zařízeních ze kterých je
složeno a dodatečné
informace specifické pro partnera.

Dodatečné předávané parametry:

| Parametr        | Typ             | Povinný | Popis                                                                                                                    |
|:----------------|:----------------|:--------|:-------------------------------------------------------------------------------------------------------------------------|
| physicalDevices | PhysicalDevices | ano     | Fyzická zařízení / komponenty ze kterých je zařízení složeno.                                                            |
| note            | string          | ne      | Poznámka zadaná člověkem který zařízení instaloval.                                                                      |
| roomId          | string (uuid)   | ano     | Identifikátor místnosti ve které se zařízení nachází. Informace o místnostech by měla dodat netlia ještě před instalací. |

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
    "eventType": "temperature-regulator-created",
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

| Parametr          | Typ    | Povinný | Popis                                                                                             |
|:------------------|:-------|:--------|:--------------------------------------------------------------------------------------------------|
| targetTemperature | float  | ano     | Teplota které se aplikace snaží nově dosáhnout/udržovat.                                          |
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

Událost je odeslána při přiřazení fyzického zařízení k již existujícímu zařízení. Například, když je k regulátoru
teploty přiřazena nová termostatická hlavice nebo teploměr.

Dodatečné předávané parametry:

| Parametr           | Typ            | Povinný | Popis                                                                                |
|:-------------------|:---------------|:--------|:-------------------------------------------------------------------------------------|
| physicalDeviceId   | 	string        | 	ano    | 	Id přiřazeného fyzického zařízení.                                                  |
| physicalDeviceType | 	string (enum) | 	ano    | 	Typ přiřazeného fyzického zařízení. Může nabývat hodnot - thermo-head a thermometer |

Ukázka zaslané události:

```yaml
{
    "protocolVersion": 1,
    "deviceId": "d65f1ffb-aa60-4eff-9666-78a93a048b16",
    "deviceType": "temperature-regulator",
    "eventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "eventTime": "2024-10-09T14:12:38.91Z",
    "eventType": "physical-device-attached-to-device",
    "physicalDeviceId": "def456",
    "physicalDeviceType": "thermo-head"
}
```

### EventType physical-device-detached

Událost je odeslána při odpojení fyzického zařízení od zařízení. Například, když je od regulátoru teploty odpojena
termostatická hlavice nebo teploměr. Důvodem odpojení může být například porucha zařízení, vyjmutí z důvodu údržby nebo
trvalé odebrání zařízení.

Dodatečné předávané parametry:

| Parametr         | Typ     | Povinný | Popis                              |
|:-----------------|:--------|:--------|:-----------------------------------|
| physicalDeviceId | 	string | 	ano    | 	Id odpojeného fyzického zařízení. |

Ukázka zaslané události:

```yaml
{
    "protocolVersion": 1,
    "deviceId": "d65f1ffb-aa60-4eff-9666-78a93a048b16",
    "deviceType": "temperature-regulator",
    "eventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "eventTime": "2024-10-09T14:12:38.91Z",
    "eventType": "physical-device-detached-from-device",
    "physicalDeviceId": "def456"
}
```
