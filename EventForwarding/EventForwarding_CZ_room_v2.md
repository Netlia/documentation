# Netlia events documentation

Tato dokumentace popisuje události týkající se měření a regulace teploty, stavu zařízení a správy místností a radiátorů.

Související operace jsou popsány v [dokumentaci REST API](../RestApi/regulation_api.md).

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
https://partner.example/events/v{ProtocolVersion}/{EventType}/{EventId}
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
| [`building-upserted`](#eventtype-building-upserted) | Aktuální stav budovy: podlaží a místnosti na nich. |
| [`room-upserted`](#eventtype-room-upserted) | Aktuální obsah místnosti: poznámka, zařízení a radiátory. |
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

Identifikátory místností, radiátorů, entit, zařízení a událostí mají formát UUID:

`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`

### Nepovinné hodnoty

U nepovinných parametrů v tabulkách může být hodnota `null`. Příjemce by měl akceptovat také jejich vynechání.

### Objekt Entity

Entita označuje místo nebo prvek budovy, ke kterému je zařízení přiřazeno, například místnost, radiátor, stoupačku
nebo stranu budovy.

Radiátor vždy patří do jedné místnosti. Termostatické hlavice jsou namontovány na radiátorech, proto události
týkající se hlavice uvádějí jako entitu radiátor. Ostatní zařízení, například teploměr nebo termostat,
patří přímo k místnosti.

Objekt `Entity` obsahuje identifikátor a typ entity:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| id | string (UUID) | ano | ID entity. Pro `type: "room"` jde o ID místnosti, pro `type: "radiator"` o ID radiátoru. |
| type | string (výčet níže) | ano | Typ entity, které se událost týká. |

Definované typy entit:

| type | Význam | Aktuální podpora |
|:-----------|:-------|:-----------------|
| `room` | Místnost. | Události se odesílají. |
| `radiator` | Radiátor v místnosti, na kterém jsou namontovány termostatické hlavice. | Události se odesílají. |
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

## Building and room events

Stav budov a místností se předává dvěma typy událostí, z nichž každá vždy obsahuje úplný aktuální stav:

- `building-upserted` určuje, které budovy, podlaží a místnosti existují, jak se jmenují a na kterém podlaží
  která místnost je,
- `room-upserted` určuje obsah jedné místnosti – poznámku, zařízení a radiátory.

Každá budova a každá místnost má vlastní `version`, která se s každou změnou zvyšuje. Události nemusí dorazit ve stejném
pořadí, v jakém vznikly, a mohou být doručeny opakovaně. Příjemce proto událost zpracuje pouze tehdy, je-li její
`version` vyšší než verze budovy nebo místnosti, kterou už zpracoval; jinak ji ignoruje.

### EventType building-upserted

Obsahuje úplný aktuální stav budovy: její název, podlaží a místnosti na nich. Odesílá se po každé změně budovy, jejích
podlaží nebo jejích místností – po vytvoření, přejmenování, přesunu nebo smazání místnosti a po importu půdorysu.

Příjemce budovu vytvoří, pokud ji ještě nezná, a jinak ji nahradí přijatým stavem. Podlaží a místnosti, které
v události nejsou, odebere; místnost odebere včetně jejích zařízení a radiátorů. Místnost přesunutá na jiné podlaží
se v události objeví u nového podlaží. Nově uvedená místnost nemá žádná zařízení ani radiátory, dokud pro ni
nepřijde událost `room-upserted`.

Data obsahují:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| buildingId | string (UUID) | ano | ID budovy. |
| version | int | ano | Verze stavu budovy. Vyšší číslo znamená novější stav. |
| partnerId | string (UUID) | ano | ID partnera, kterému budova patří. |
| name | string | ano | Název budovy. |
| floors | floor[] | ano | Podlaží budovy. Prázdné pole, pokud budova žádné nemá. |

Objekt `floor` má následující formát:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| floorId | string (UUID) | ano | ID podlaží. |
| name | string | ano | Název podlaží. |
| rooms | room[] | ano | Místnosti na podlaží. Prázdné pole, pokud na podlaží žádná není. |

Objekt `room` má následující formát:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| roomId | string (UUID) | ano | ID místnosti. |
| name | string | ano | Název místnosti. |

Ukázka zaslané události:

```json
{
    "protocolVersion": 2,
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000009",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "building-upserted",
    "data": {
        "buildingId": "fdc2ea0d-d0f9-4d2f-aeca-5116f514093f",
        "version": 4,
        "partnerId": "9381f1af-4187-4478-8c80-391c8882f421",
        "name": "Administrativní budova",
        "floors": [
            {
                "floorId": "d77c48f1-f7f3-4a02-b075-69b989914463",
                "name": "1. patro",
                "rooms": [
                    {
                        "roomId": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
                        "name": "Kancelář 101"
                    }
                ]
            },
            {
                "floorId": "0b6d6f2e-2f7a-4c55-9a1e-3c1c2b5f0a10",
                "name": "2. patro",
                "rooms": []
            }
        ]
    }
}
```

### EventType room-upserted

Obsahuje úplný aktuální obsah místnosti: poznámku, zařízení umístěná přímo v místnosti a radiátory s jejich
termostatickými hlavicemi. Odesílá se po každé změně obsahu místnosti – po změně poznámky, instalaci, odinstalaci
nebo výměně zařízení a po změně radiátorů. Název místnosti a podlaží, na kterém je, určuje událost
`building-upserted`.

Příjemce obsah místnosti nahradí přijatým stavem: radiátory a zařízení, které v události nejsou, z místnosti odebere.
Pokud místnost ještě nezná, protože událost `building-upserted`, která ji uvádí, zatím nedorazila, může zpracování
odložit a zopakovat později.

Data obsahují:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| roomId | string (UUID) | ano | ID místnosti. |
| version | int | ano | Verze obsahu místnosti. Vyšší číslo znamená novější stav. |
| note | string | ano | Poznámka k místnosti, případně prázdný řetězec. |
| devices | device[] | ano | Zařízení umístěná přímo v místnosti – teploměr nebo termostat. Prázdné pole, pokud místnost žádné nemá. |
| radiators | radiator[] | ano | Radiátory místnosti. Prázdné pole, pokud místnost žádné nemá. |

Objekt `device` má následující formát:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| deviceId | string (UUID) | ano | ID zařízení. |
| deviceType | string | ano | Typ zařízení, viz [Device](#device). |

Objekt `radiator` má následující formát:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| radiatorId | string (UUID) | ano | ID radiátoru. Odpovídá `entity.id` u entity typu `radiator` v ostatních událostech. |
| positionFromLeft | int nebo null | ano | Pořadí radiátoru v místnosti počítané zleva od dveří, začíná od `1`. `null` u radiátorů, jejichž pořadí zatím nebylo zadáno. |
| note | string nebo null | ano | Poznámka k radiátoru. |
| devices | device[] | ano | Termostatické hlavice namontované na radiátoru. Prázdné pole, pokud na radiátoru žádná není. |

Místnost má nejvýše jeden teploměr nebo termostat a na každém radiátoru je namontována nejvýše jedna hlavice.
Zařízení se mezi místnostmi nepřesouvá přímo: nejprve je odinstalováno z jedné místnosti a teprve potom
nainstalováno do jiné.

Ukázka zaslané události:

```json
{
    "protocolVersion": 2,
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000010",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "room-upserted",
    "data": {
        "roomId": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
        "version": 7,
        "note": "Teploměr je umístěn za rohem",
        "devices": [
            {
                "deviceId": "6e748f20-846e-4e89-a831-000000000004",
                "deviceType": "thermometer"
            }
        ],
        "radiators": [
            {
                "radiatorId": "3f2b8c1e-6a4d-4f0e-9b7a-000000000021",
                "positionFromLeft": 1,
                "note": null,
                "devices": [
                    {
                        "deviceId": "6e748f20-846e-4e89-a831-000000000001",
                        "deviceType": "thermo-head"
                    }
                ]
            },
            {
                "radiatorId": "3f2b8c1e-6a4d-4f0e-9b7a-000000000022",
                "positionFromLeft": 2,
                "note": "Pod oknem",
                "devices": []
            }
        ]
    }
}
```

## Measurement and state events

### EventType measured-temperature

Informuje o změření teploty pro entitu určenou objektem `entity`. Aktuálně jde vždy o místnost.
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
U každé hlavice je uveden radiátor, na kterém je namontována.

Data obsahují:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| positionInformation | positionInformation | ano | Informace o nových polohách hlavic. |

Objekt `positionInformation` má následující formát:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| positions | position[] | ano | Pole změn poloh hlavic. |

Objekt `position` má následující formát:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| deviceId | string (UUID) | ano | ID zařízení – termostatické hlavice. |
| entity | Entity | ano | Entita, na které je hlavice namontována – radiátor. |
| newPosition | int (0–100) | ano | Nová poloha v procentech. `0` znamená zavřenou hlavici, `100` plně otevřenou. |

Ukázka zaslané události:

```json
{
    "protocolVersion": 2,
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000002",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "thermo-head-changed-position",
    "data": {
        "positionInformation": {
            "positions": [
                {
                    "deviceId": "6e748f20-846e-4e89-a831-000000000001",
                    "entity": {
                        "id": "3f2b8c1e-6a4d-4f0e-9b7a-000000000021",
                        "type": "radiator"
                    },
                    "newPosition": 25
                },
                {
                    "deviceId": "6e748f20-846e-4e89-a831-000000000003",
                    "entity": {
                        "id": "3f2b8c1e-6a4d-4f0e-9b7a-000000000022",
                        "type": "radiator"
                    },
                    "newPosition": 30
                }
            ]
        }
    }
}
```

### EventType heating-state-changed

Informuje o změně cílové teploty nebo stavu topení entity, například o zahájení nebo ukončení předehřívání.
Aktuálně je entitou vždy místnost. Změnu může vyvolat požadavek uživatele či partnerské aplikace nebo zpracování plánu.

Data obsahují:

| Parametr          | Typ                     | Povinný | Popis                                                                    |
|:------------------|:------------------------|:--------|:-------------------------------------------------------------------------|
| entity | Entity | ano | Entita, které se událost týká. Aktuálně vždy typu `room`. |
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
        "entity": {
            "id": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
            "type": "room"
        },
        "targetTemperature": 25.5,
        "changeReason": "target-temperature-changed",
        "sourceRequestType": "by-admin-app",
        "sourceRequestId": "30a86332-7a75-4e55-8217-1b69c1d6b301"
    }
}
```

## Device health and failure events

### EventType battery-alert

Upozorňuje na změnu stavu baterie zařízení. Stav baterie je vlastností zařízení, proto událost neobsahuje entitu.

Data obsahují:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
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
        "deviceId": "6e748f20-846e-4e89-a831-000000000001",
        "batteryStatus": "low"
    }
}
```

### EventType failure

Upozorňuje na problém týkající se celé entity nebo jednoho jejího zařízení. Povinný objekt `entity`
určuje entitu, které se problém týká. Událost se může týkat libovolného typu entity. U problému termostatické
hlavice je entitou radiátor, na kterém je hlavice namontována; u ostatních zařízení místnost.

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

Pro označení podporovaného selhání za vyřešené slouží endpoint `PUT api/entity/resolve-failure`.

Ukázka selhání konkrétního zařízení:

```json
{
    "protocolVersion": 2,
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000004",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "failure",
    "data": {
        "entity": {
            "id": "3f2b8c1e-6a4d-4f0e-9b7a-000000000021",
            "type": "radiator"
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
`PUT api/entity/resolve-failure`, tato událost se neodesílá.

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
            "id": "3f2b8c1e-6a4d-4f0e-9b7a-000000000021",
            "type": "radiator"
        },
        "deviceId": "6e748f20-846e-4e89-a831-000000000001",
        "type": "generic-device-error"
    }
}
```
