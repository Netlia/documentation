# Netlia events documentation

Tato dokumentace popisuje události týkající se měření a regulace teploty, stavu zařízení a správy místností.

Související operace jsou popsány v [dokumentaci REST API](../RestApi/TemperatureRegulator.md).

## Obsah

Obsah zobrazíte následujícím způsobem:

![content](../images/table-of-contents.webp "Content")

## Zasílání událostí a konfigurace

Události jsou zasílány metodou HTTP `POST` na nakonfigurovanou adresu partnera. Tělo požadavku obsahuje data ve formátu JSON.
Požadavek má hlavičku `Content-Type: application/json`. S požadavkem jsou odeslány také hlavičky nastavené pro danou trasu.

Pokud to partner vyžaduje, lze v cílové URL použít následující zástupné parametry:

| Parametr            | Význam                                             |
|:--------------------|:---------------------------------------------------|
| `{ProtocolVersion}` | Verze protokolu.                                   |
| `{EventType}`       | Název události, například `heating-state-changed`. |
| `{EventId}`         | ID události.                                       |

Příklad cílové adresy:

```text
https://partner.example/events/v{ProtocolVersion}/{EntityType}/{EventType}/{EventId}
```

## Sdílené parametry

Příklad události:

```json
{
    "protocolVersion": 2,
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000001",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "measured-temperature",
    "data": {
        "entity": {
            "id": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
            "type": "room"
        },
        "deviceId": "6e748f20-846e-4e89-a831-000000000004",
        "temperature": 25.5
    }
}
```

Všechny události obsahují sdílené položky na nejvyšší úrovni objektu:

| Parametr        | Typ              | Povinný | Popis                                             |
|:----------------|:-----------------|:--------|:--------------------------------------------------|
| protocolVersion | int              | ano     | Verze protokolu, vždy `2`.                        |
| eventId         | string (UUID)    | ano     | ID události, slouží k zajištění idempotence.      |
| eventTime       | string (UTC čas) | ano     | Čas, kdy událost nastala.                         |
| eventType       | string           | ano     | Typ události. Výčet a příklady jsou uvedeny níže. |
| data            | object nebo null | ne      | Data specifická pro daný typ události.                       |

Objekt `data` obsahuje parametry popsané v části „Data obsahují:“ u příslušného typu události.
Názvy parametrů v těchto částech jsou relativní k objektu `data`, například `entity` znamená `data.entity`.
Ukázky událostí uvádějí vždy celý objekt včetně sdílených parametrů a objektu `data`.

Typy událostí:

| eventType | Význam                                               |
|:----------|:-----------------------------------------------------|
| [`room-created`](#eventtype-room-created) | Vytvoření místnosti včetně údajů o budově a podlaží. |
| [`room-renamed`](#eventtype-room-renamed) | Změna názvu místnosti.                               |
| [`room-deleted`](#eventtype-room-deleted) | Smazání místnosti.                                   |
| [`room-floor-changed`](#eventtype-room-floor-changed) | Přesun místnosti do jiného podlaží stejné budovy.    |
| [`entity-note-changed`](#eventtype-entity-note-changed) | Změna poznámky entity                                |
| [`device-installed`](#eventtype-device-installed) | Instalace zařízení.                                  |
| [`device-uninstalled`](#eventtype-device-uninstalled) | Odinstalace zařízení.                                |
| [`device-replaced`](#eventtype-device-replaced) | Výměna zařízení.                                     |
| [`measured-temperature`](#eventtype-measured-temperature) | Naměřená teplota.                                    |
| [`thermo-head-changed-position`](#eventtype-thermo-head-changed-position) | Změna polohy termostatických hlavic.                 |
| [`heating-state-changed`](#eventtype-heating-state-changed) | Změna cílové teploty nebo stavu topení.              |
| [`battery-alert`](#eventtype-battery-alert) | Změna stavu baterie zařízení.                        |
| [`failure`](#eventtype-failure) | Problém entity nebo jejího zařízení.                 |
| [`failure-resolved`](#eventtype-failure-resolved) | Vyřešení problému entity nebo jejího zařízení.       |

## Základní datové typy

### Čas

Čas je předáván jako řetězec v UTC, například `2026-09-23T14:12:38Z`.
Může obsahovat také zlomky sekundy, například `2026-09-23T14:12:38.91Z`.
Přípona `Z` označuje časové pásmo UTC (+00:00).

### UUID

Identifikátory místností, entit, zařízení a událostí mají formát UUID:

`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`

### Nepovinné hodnoty

U nepovinných parametrů v tabulkách může být hodnota `null`. Příjemce by měl akceptovat také jejich vynechání.

### Objekt Entity

Entita označuje místo nebo prvek budovy, ke kterému je zařízení přiřazeno, například místnost, stoupačku nebo stranu budovy.

Objekt `Entity` obsahuje identifikátor a typ entity:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| id | string (UUID) | ano | ID entity. Pro `type: "room"` jde o ID místnosti. |
| type | string (výčet níže) | ano | Typ entity, které se událost týká. |

Definované typy entit:

| type | Význam | Aktuální podpora |
|:-----------|:-------|:-----------------|
| `room` | Místnost. | Události se odesílají. |
| `riser` | Stoupačka. | Vyhrazeno pro budoucí rozšíření. |
| `building-side` | Strana budovy, například severní nebo východní, pro venkovní teploměry. | Vyhrazeno pro budoucí rozšíření. |

Ukázka objektu `Entity`:

```json
{
    "id": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
    "type": "room"
}
```

### Device

Device označuje zařízení, které se instaluje u zákazníka.

Definované hodnoty `deviceType`:

| deviceType    | Význam                                             |
|:--------------|:---------------------------------------------------|
| `thermo-head` | Termostatická hlavice ovládající ventil radiátoru. |
| `thermometer` | Teploměr měřící teplotu.                           |

Termostat je pro účely tohoto kontraktu uváděn jako `thermometer`.

> Nové typy událostí, zařízení a entit i další hodnoty mohou kdykoli přibývat. Kód příjemce by měl s těmito
> situacemi počítat a umět zpracovat i neznámé hodnoty – typicky vrátit odpověď OK a zaznamenat chybu do logu.

## Entity events

### EventType room-created

Informuje o vytvoření místnosti a obsahuje údaje o jejím podlaží a budově. Pokud podlaží nebo budova
v cílovém systému ještě neexistují, příjemce je vytvoří z předaných údajů.

Data obsahují:

| Parametr | Typ | Povinný | Popis                                |
|:---------|:----|:--------|:-------------------------------------|
| roomId | string (UUID) | ano | ID místnosti, které se událost týká. |
| name | string | ano | Název místnosti.                     |
| note | string | ano | Poznámka místnosti.                  |
| partnerId | string (UUID) | ano | ID partnera, kterému budova patří.   |
| buildingId | string (UUID) | ano | ID budovy.                           |
| buildingName | string | ano | Název budovy.                        |
| floorId | string (UUID) | ano | ID podlaží.                          |
| floorName | string | ano | Název podlaží.                       |

Ukázka zaslané události:

```json
{
    "protocolVersion": 2,
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000010",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "room-created",
    "data": {
        "roomId": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
        "partnerId": "9381f1af-4187-4478-8c80-391c8882f421",
        "buildingId": "fdc2ea0d-d0f9-4d2f-aeca-5116f514093f",
        "buildingName": "Administrativní budova",
        "floorId": "d77c48f1-f7f3-4a02-b075-69b989914463",
        "floorName": "1. patro",
        "name": "Kancelář 101",
        "note": "Teploměr je umístěn za rohem"
    }
}
```

### EventType room-renamed

Informuje o změně názvu místnosti. Identifikátor `roomId` se přejmenováním nemění.

Data obsahují:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| roomId | string (UUID) | ano | ID místnosti, které se událost týká. |
| name | string | ano | Nový název místnosti. |

Ukázka zaslané události:

```json
{
    "protocolVersion": 2,
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000011",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "room-renamed",
    "data": {
        "roomId": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
        "name": "Zasedací místnost"
    }
}
```

### EventType room-deleted

Smazání místnosti. Místnost lze smazat pouze tehdy, pokud k ní není přiřazen žádný device.

Data obsahují:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| roomId | string (UUID) | ano | ID smazané místnosti. |

```json
{
    "protocolVersion": 2,
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000012",
    "eventTime": "2026-09-25T14:12:38Z",
    "eventType": "room-deleted",
    "data": {
        "roomId": "d65f1ffb-aa60-4eff-9666-78a93a048b17"
    }
}
```

### EventType room-floor-changed

Informuje o přesunu místnosti do jiného podlaží **ve stejné budově**. K přesunu může dojít během instalace,
pokud byl původní plán podlaží chybný a místnost byla nejprve zařazena do nesprávného podlaží.

Cílové podlaží musí existovat a patřit do stejné budovy.

Data obsahují:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| roomId | string (UUID) | ano | ID přesunuté místnosti. |
| floorId | string (UUID) | ano | ID cílového podlaží. |

```json
{
    "protocolVersion": 2,
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000013",
    "eventTime": "2026-09-25T14:12:38Z",
    "eventType": "room-floor-changed",
    "data": {
        "roomId": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
        "floorId": "c4056fc4-d433-4d2c-bb7f-000000000014"
    }
}
```

### EventType entity-note-changed

Informovat o změně poznámky na entitě.

Data obsahují:

| Parametr | Typ    | Povinný | Popis                                            |
|:---------|:-------|:--------|:-------------------------------------------------|
| entity   | Entity | ano     | ID a typ entity, jejíž poznámka se změnila.      |
| note     | string | ano     | Nová hodnota poznámky, případně prázdný řetězec. |

```json
{
    "protocolVersion": 2,
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000015",
    "eventTime": "2026-09-25T14:12:38Z",
    "eventType": "entity-note-changed",
    "data": {
        "entity": {
            "id": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
            "type": "room"
        },
        "note": "Neobsazená místnost"
    }
}
```

## Device installation events

### EventType device-installed

Informuje o instalaci jednoho nebo více zařízení u entity určené objektem `entity`.

Data obsahují:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| entity | Entity | ano | Entita, které se událost týká. |
| devices | device[] | ano | Pole nainstalovaných zařízení. |

Objekt `device` má následující formát:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| deviceId | string (UUID) | ano | ID nainstalovaného zařízení. |
| deviceType | string | ano | Typ zařízení: `thermo-head` nebo `thermometer` pro instalaci topení v místnosti. |

Termostat je pro účely tohoto kontraktu uváděn jako `thermometer`.

Ukázka zaslané události:

```json
{
    "protocolVersion": 2,
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000007",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "device-installed",
    "data": {
        "entity": {
            "id": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
            "type": "room"
        },
        "devices": [
            {
                "deviceId": "6e748f20-846e-4e89-a831-000000000001",
                "deviceType": "thermo-head"
            },
            {
                "deviceId": "6e748f20-846e-4e89-a831-000000000003",
                "deviceType": "thermo-head"
            },
            {
                "deviceId": "6e748f20-846e-4e89-a831-000000000004",
                "deviceType": "thermometer"
            }
        ]
    }
}
```

### EventType device-uninstalled

Informuje o odinstalaci zařízení, například při odebrání porouchané hlavice z místnosti.
Místnost zůstává zachována. Při odinstalaci jednoho zařízení obsahuje pole jedno ID.

Data obsahují:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| entity | Entity | ano | Entita, které se událost týká. |
| deviceIds | string (UUID)[] | ano | ID odinstalovaných zařízení. |

Ukázka zaslané události:

```json
{
    "protocolVersion": 2,
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000009",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "device-uninstalled",
    "data": {
        "entity": {
            "id": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
            "type": "room"
        },
        "deviceIds": [
            "6e748f20-846e-4e89-a831-000000000001"
        ]
    }
}
```

### EventType device-replaced

Informuje o výměně zařízení přiřazeného k entitě. Zařízení `replacedDeviceId` bylo
nahrazeno zařízením `replacementDeviceId`, například při výměně porouchané hlavice.
Místnost určuje objekt `entity` s typem `room` a ID místnosti.

Náhradní zařízení má vždy stejný typ jako nahrazované zařízení.
Při výměně se odesílá událost `device-replaced`; události `device-installed` ani `device-uninstalled`
se pro tuto výměnu neodesílají.

Data obsahují:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| entity | Entity | ano | Entita, které se událost týká. |
| replacedDeviceId | string (UUID) | ano | ID původního zařízení. |
| replacementDeviceId | string (UUID) | ano | ID nového zařízení. |

Ukázka zaslané události:

```json
{
    "protocolVersion": 2,
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000006",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "device-replaced",
    "data": {
        "entity": {
            "id": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
            "type": "room"
        },
        "replacedDeviceId": "6e748f20-846e-4e89-a831-000000000001",
        "replacementDeviceId": "6e748f20-846e-4e89-a831-000000000002"
    }
}
```

## Measurement and state events

### EventType measured-temperature

Informuje o změření teploty pro entitu určenou objektem `entity`.
`deviceId` označuje zařízení, ze kterého měření pochází.

Data obsahují:

| Parametr         | Typ    | Povinný | Popis                                                       |
|:-----------------|:-------|:--------|:------------------------------------------------------------|
| entity | Entity | ano | Entita, které se událost týká. |
| deviceId | string (UUID) | ano     | ID zařízení, které provedlo měření.               |
| temperature      | float  | ano     | Naměřená teplota [°C], zaokrouhlená na dvě desetinná místa. |

Ukázka zaslané události:

```json
{
    "protocolVersion": 2,
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000001",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "measured-temperature",
    "data": {
        "entity": {
            "id": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
            "type": "room"
        },
        "deviceId": "6e748f20-846e-4e89-a831-000000000004",
        "temperature": 25.5
    }
}
```

### EventType thermo-head-changed-position

Informuje o změně polohy jedné nebo více termostatických hlavic v místnosti.

Data obsahují:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| roomId | string (UUID) | ano | ID místnosti, které se událost týká. |
| positionInformation | positionInformation | ano | Informace o nových polohách hlavic. |

Objekt `positionInformation` má následující formát:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| positions | position[] | ano | Pole změn poloh hlavic. |

Objekt `position` má následující formát:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| deviceId | string (UUID) | ano | ID zařízení – termostatické hlavice. |
| newPosition | int (0–100) | ano | Nová poloha v procentech. `0` znamená zavřenou hlavici, `100` plně otevřenou. |

Ukázka zaslané události:

```json
{
    "protocolVersion": 2,
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000002",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "thermo-head-changed-position",
    "data": {
        "roomId": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
        "positionInformation": {
            "positions": [
                {
                    "deviceId": "6e748f20-846e-4e89-a831-000000000001",
                    "newPosition": 25
                },
                {
                    "deviceId": "6e748f20-846e-4e89-a831-000000000003",
                    "newPosition": 30
                }
            ]
        }
    }
}
```

### EventType heating-state-changed

Informuje o změně cílové teploty nebo stavu topení v místnosti, například o zahájení nebo ukončení předehřívání.
Změnu může vyvolat požadavek uživatele či partnerské aplikace nebo zpracování plánu.

Data obsahují:

| Parametr          | Typ                     | Povinný | Popis                                                                    |
|:------------------|:------------------------|:--------|:-------------------------------------------------------------------------|
| roomId | string (UUID) | ano | ID místnosti, které se událost týká. |
| targetTemperature | float nebo null         | ano     | Cílová teplota [°C]. Může být `null`, pokud není pro danou změnu určena. |
| changeReason      | string (výčet níže)     | ano     | Důvod změny.                                                             |
| sourceRequestType | string nebo null        | ne      | Původ požadavku: `by-user` nebo `by-admin-app`.                          |
| sourceRequestId   | string (UUID) nebo null | ne      | ID původního požadavku, pokud je k dispozici.                            |

Hodnoty `changeReason`:

| changeReason                 | Popis                          |
|:-----------------------------|:-------------------------------|
| `pre-heating-started`        | Začalo předehřívání místnosti. |
| `pre-heating-stopped`        | Předehřívání skončilo.         |
| `target-temperature-changed` | Změnila se cílová teplota.     |
| `diagnostic-started`         | Začal diagnostický režim.      |
| `diagnostic-stopped`         | Diagnostický režim skončil.    |

Pokud změna navazuje na požadavek partnerské aplikace, `sourceRequestType` má hodnotu `by-admin-app` a
`sourceRequestId` může obsahovat `requestId` původního volání API. Při změně vyvolané tlačítkem termostatu
má `sourceRequestType` hodnotu `by-user`. U změn bez tohoto kontextu mohou být oba parametry `null`.

Ukázka zaslané události:

```json
{
    "protocolVersion": 2,
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000008",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "heating-state-changed",
    "data": {
        "roomId": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
        "targetTemperature": 25.5,
        "changeReason": "target-temperature-changed",
        "sourceRequestType": "by-admin-app",
        "sourceRequestId": "30a86332-7a75-4e55-8217-1b69c1d6b301"
    }
}
```

## Device health and failure events

### EventType battery-alert

Upozorňuje na změnu stavu baterie zařízení.

Data obsahují:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| entity | Entity | ano | Entita, které se událost týká. |
| deviceId | string (UUID) | ano | ID zařízení. |
| batteryStatus | string (výčet níže) | ano | Stav baterie. |

Definované hodnoty `batteryStatus`:

- `high` – nabitá baterie.
- `low` – téměř vybitá baterie.
- `dead` – vybitá baterie, zařízení již nekomunikuje.
- `unknown` – stav baterie není znám.

Ukázka zaslané události:

```json
{
    "protocolVersion": 2,
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000003",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "battery-alert",
    "data": {
        "entity": {
            "id": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
            "type": "room"
        },
        "deviceId": "6e748f20-846e-4e89-a831-000000000001",
        "batteryStatus": "low"
    }
}
```

### EventType failure

Upozorňuje na problém týkající se celé entity nebo jednoho jejího zařízení. Povinný objekt `entity`
určuje entitu, které se problém týká. Událost se může týkat libovolného typu entity.

- `deviceId: null` označuje problém celé entity. Parametr může být také vynechán.
- Vyplněné `deviceId` označuje problém konkrétního zařízení v dané entitě, například vložení vybité baterie
  nebo hardwarovou závadu.

Pro každou kombinaci `entity.type`, `entity.id`, `deviceId` a `type` může být aktivní nejvýše jedno selhání.
Hodnota `deviceId: null` a vynechaný parametr označují stejný případ – problém celé entity.

Data obsahují:

| Parametr              | Typ                     | Povinný | Popis                                                                                                     |
|:----------------------|:------------------------|:--------|:----------------------------------------------------------------------------------------------------------|
| entity | Entity | ano | Entita, které se problém týká. |
| deviceId | string (UUID) nebo null | ne | ID zařízení, kterého se problém týká. Hodnota `null` nebo vynechání označuje problém celé entity. |
| type                  | string (výčet níže)     | ano     | Typ selhání.                                                                                              |
| severity              | string                  | ano     | Závažnost: `warning` nebo `error`.                                                                        |
| localizedDescription  | string                  | ano     | Uživatelský popis selhání.                                                                                |
| isResolvableByPartner | bool                    | ano     | Zda může partner označit problém jako vyřešený.                                                           |

`error` označuje problém, který může způsobit nefunkčnost a měl by být vyřešen co nejrychleji.
`warning` označuje upozornění, které nemusí vyžadovat okamžitý zásah.

Pokud je `isResolvableByPartner` nastaveno na `true`, může partnerská aplikace nabídnout uživateli možnost
označit problém jako vyřešený. Hodnota `false` může znamenat, že je nejprve potřeba fyzický zásah, například
výměna zařízení nebo baterie. Závažnost a možnost vyřešení vždy určují hodnoty konkrétní události.

Definované hodnoty `type` pro problémy konkrétního zařízení (vyžadují vyplněné `deviceId`):

| type                                    | Popis                                              |
|:----------------------------------------|:---------------------------------------------------|
| `inserted-discharged-battery`           | Do zařízení byla vložena vybitá baterie.           |
| `inserted-partially-discharged-battery` | Do zařízení byla vložena částečně nabitá baterie.  |
| `generic-device-error`         | Obecná porucha zařízení.                 |
| `thermo-head-position-setting-failed`   | Nepodařilo se nastavit požadovanou polohu hlavice. |

Konkrétní hodnoty `type` pro problémy celé entity zatím nejsou definovány.

Pro označení podporovaného selhání za vyřešené slouží endpoint `PUT api/device-failure/resolve`.

Ukázka selhání konkrétního zařízení:

```json
{
    "protocolVersion": 2,
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000004",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "failure",
    "data": {
        "entity": {
            "id": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
            "type": "room"
        },
        "deviceId": "6e748f20-846e-4e89-a831-000000000001",
        "type": "generic-device-error",
        "severity": "error",
        "localizedDescription": "Došlo k závadě na zařízení, proveďte prosím jeho výměnu.",
        "isResolvableByPartner": false
    }
}
```

### EventType failure-resolved

Informuje o vyřešení selhání oznámeného událostí `failure`.
Vyřešené selhání určuje kombinace `type`, `entity.id`, `entity.type` a `deviceId`, která odpovídá
hodnotám v události `failure`.

- U problému celé entity má parametr `deviceId` hodnotu `null` nebo je vynechán.
- U problému konkrétního zařízení obsahuje parametr `deviceId` ID tohoto zařízení.

Parametr `eventId` je jedinečný identifikátor události vyřešení.

Událost se odesílá, když systém zjistí, že byl problém vyřešen. Pokud problém vyřeší partner pomocí endpointu
`PUT api/device-failure/resolve`, tato událost se neodesílá.

Data obsahují:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| entity | Entity | ano | Entita, jejíž problém byl vyřešen. |
| deviceId | string (UUID) nebo null | ne | ID zařízení, jehož problém byl vyřešen. Hodnota `null` nebo vynechání označuje problém celé entity. |
| type | string | ano | Typ vyřešeného selhání; stejné hodnoty jako u `failure`. |

Ukázka zaslané události:

```json
{
    "protocolVersion": 2,
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000005",
    "eventTime": "2026-09-23T14:22:38Z",
    "eventType": "failure-resolved",
    "data": {
        "entity": {
            "id": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
            "type": "room"
        },
        "deviceId": "6e748f20-846e-4e89-a831-000000000001",
        "type": "generic-device-error"
    }
}
```
