# Události místností – protokol 2.0

## Místnost a fyzická zařízení

Protokol 2.0 popisuje měření a regulaci teploty z pohledu místnosti. Místnost je označena identifikátorem
`roomId`. Fyzická zařízení, například termostatické hlavice a teploměr nebo termostat, jsou přiřazena k místnosti.
Události neobsahují identifikátor teplotního regulátoru ani parametry `deviceId` a `deviceType`.

Události o přiřazení, odpojení a výměně fyzických zařízení používají obecné označení entity:
`entityType` a `entityId`. Aktuálně se odesílají pouze pro místnosti (`entityType: "room"`).

Protokol je prozatím překladovou vrstvou nad stávajícími událostmi. Zachovává jejich význam, čas vzniku a
identifikátor události. Pokud teplotní regulátor v IOTC nemá přiřazenou místnost, jeho události se ve verzi 2
neodesílají. Zasílání verze 1 tím není ovlivněno. Události venkovních teploměrů, teploměrů stoupaček a PIR
čidel nejsou v této verzi podporovány.

Dokumentace původního protokolu je dostupná v [popisu událostí verze 1](EventForwarding_CZ_regulator.md).
Změna protokolu událostí sama o sobě nemění [REST API](../RestApi/TemperatureRegulator.md).

## Obsah

Obsah zobrazíte následujícím způsobem:

![content](../images/table-of-contents.webp "Content")

## Základní datové typy

### Čas

Čas je předáván jako řetězec v UTC, například `2026-09-23T14:12:38Z`.
Může obsahovat také zlomky sekundy, například `2026-09-23T14:12:38.91Z`.
Přípona `Z` označuje časové pásmo UTC (+00:00).

### UUID

Identifikátory místností, entit a událostí mají formát UUID:

`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`

### Identifikátor fyzického zařízení

`physicalDeviceId` je řetězec obsahující sériové číslo fyzického zařízení. U nových zařízení odpovídá jeho ID
ve formátu UUID, u starších zařízení může mít jiný formát, například `abc123`. Příjemce proto nesmí vyžadovat,
aby každý identifikátor fyzického zařízení byl UUID. Stejná pravidla platí pro položky `physicalDeviceIds` a
pro parametry `replacedPhysicalDeviceId` a `replacementPhysicalDeviceId`.

### Nepovinné hodnoty

U nepovinných parametrů v tabulkách může být hodnota `null`. Příjemce by měl akceptovat také jejich vynechání.

## Zasílání událostí a volba verze

Události jsou zasílány metodou HTTP `POST` na nakonfigurovanou adresu partnera. Tělo požadavku obsahuje JSON
s hlavičkou `Content-Type: application/json`. S požadavkem jsou odeslány také hlavičky nastavené pro danou trasu.

Verze protokolu se nastavuje pro každou HTTP trasu samostatně. Jeden partner tak může přijímat verzi 1 na jedné
adrese a verzi 2 na jiné. Stávající trasy i nové trasy bez explicitně zadané verze používají verzi 1.
Před přepnutím trasy musí příjemce podporovat nové názvy událostí a jejich datový formát.

Při vytvoření partnera pomocí `POST /partner` lze u každé položky `routes` nastavit `protocolVersion` na `1`
nebo `2`. Pro změnu existující trasy slouží následující administrátorský endpoint:

```http
PUT /partner/{partnerId}/routes/{routeId}/protocol-version
Content-Type: application/json

{"protocolVersion": 2}
```

Změna se týká nově vznikajících událostí. Již připravené události si ponechávají původní verzi, obsah i cílovou
trasu. Při přechodu tedy mohou být ještě doručovány dříve připravené události verze 1.

V cílové adrese lze použít následující zástupné parametry:

| Parametr | Význam ve verzi 2 |
|:---------|:-----------------|
| `{ProtocolVersion}` | Verze protokolu, tedy `2`. |
| `{RoomId}` | ID místnosti. |
| `{EntityId}` | ID entity, aktuálně ID místnosti. |
| `{EntityType}` | Typ entity, aktuálně `room`. |
| `{EventType}` | Název události, například `heating-state-changed`. |
| `{EventId}` | ID události. |
| `{DeviceId}` | Zpětně kompatibilní zástupný parametr pro ID místnosti. |
| `{DeviceType}` | Zpětně kompatibilní zástupný parametr s hodnotou `room`. |

Příklad cílové adresy:

```text
https://partner.example/events/v{ProtocolVersion}/{EntityType}/{EntityId}/{EventType}
```

Zástupné parametry `{DeviceId}` a `{DeviceType}` jsou pouze součástí konfigurace URL; ve v2 JSON těle tyto
parametry nejsou.

## Základní parametry

Všechny události obsahují následující společné parametry:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| protocolVersion | int | ano | Verze protokolu, vždy `2`. |
| eventId | string (UUID) | ano | ID události, slouží k zajištění idempotence. |
| eventTime | string (UTC čas) | ano | Čas, kdy událost nastala. |
| eventType | string | ano | Typ události. Výčet a příklady jsou uvedeny níže. |

Zpracování událostí by mělo být idempotentní. Při překladu mezi verzemi zůstává `eventId` zachováno, takže stejná
událost doručená ve verzi 1 a 2 má stejné ID. Při souběžném příjmu obou verzí s tím musí příjemce počítat.

### Události místnosti

Měření, stav topení, stav baterie, poruchy a změny údajů místnosti obsahují:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| roomId | string (UUID) | ano | ID místnosti, které se událost týká. |

Běžné události neobsahují ID partnera, budovy ani podlaží. Výjimkou je `room-created`, které předává také
údaje potřebné k zařazení nově vytvořené místnosti do budovy a podlaží.

### Události přiřazení, odpojení a výměny fyzických zařízení

Události `physical-device-attached-to-entity`, `physical-device-detached-from-entity` a
`physical-device-replaced` používají místo `roomId` tyto parametry:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| entityType | string (výčet níže) | ano | Typ entity, ke které fyzické zařízení patří. Aktuálně `room`. |
| entityId | string (UUID) | ano | ID entity. Pro `entityType: "room"` jde o ID místnosti. |

Definované typy entit:

| entityType | Význam | Aktuální podpora |
|:-----------|:-------|:-----------------|
| `room` | Místnost. | Události se odesílají. |
| `riser` | Stoupačka. | Vyhrazeno pro budoucí rozšíření. |
| `building-side` | Strana budovy, například severní nebo východní, pro venkovní teploměry. | Vyhrazeno pro budoucí rozšíření. |

## Eventy

### EventType measured-humidity-temperature-in-a-room

Informuje o změření teploty a relativní vlhkosti v místnosti. `physicalDeviceId` označuje teploměr nebo
termostat, ze kterého měření pochází. Tento typ události se používá pouze pro místnosti; měření stoupaček a
venkovní teploty bude mít v budoucnu vlastní typy událostí.

Dodatečné předávané parametry:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| physicalDeviceId | string | ano | ID fyzického zařízení, které provedlo měření. |
| temperature | float | ano | Naměřená teplota [°C], zaokrouhlená na dvě desetinná místa. |
| humidity | float | ano | Naměřená relativní vlhkost [%], zaokrouhlená na jedno desetinné místo. |

Ukázka zaslané události:

```json
{
    "protocolVersion": 2,
    "roomId": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000001",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "measured-humidity-temperature-in-a-room",
    "physicalDeviceId": "abc789",
    "temperature": 25.5,
    "humidity": 27.5
}
```

### EventType thermo-head-changed-position

Informuje o změně polohy jedné nebo více termostatických hlavic v místnosti.

Dodatečné předávané parametry:

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
| physicalDeviceId | string | ano | ID fyzického zařízení – termostatické hlavice. |
| newPosition | int (0–100) | ano | Nová poloha v procentech. `0` znamená zavřenou hlavici, `100` plně otevřenou. |

V aktuální implementaci se událost odesílá po úspěšném potvrzení příkazu ke změně polohy, pokud se poloha
skutečně změnila. Obsahuje změnu potvrzenou danou hlavicí.

Ukázka zaslané události:

```json
{
    "protocolVersion": 2,
    "roomId": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000002",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "thermo-head-changed-position",
    "positionInformation": {
        "positions": [
            {
                "physicalDeviceId": "abc123",
                "newPosition": 25
            },
            {
                "physicalDeviceId": "abc456",
                "newPosition": 30
            }
        ]
    }
}
```

### EventType battery-alert

Upozorňuje na změnu stavu baterie fyzického zařízení v místnosti.

Dodatečné předávané parametry:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| physicalDeviceId | string | ano | ID fyzického zařízení. |
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
    "roomId": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000003",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "battery-alert",
    "physicalDeviceId": "abc123",
    "batteryStatus": "low"
}
```

### EventType device-failure

Upozorňuje na poruchu fyzického zařízení nebo problém týkající se místnosti. Příkladem může být vložení
vybité baterie nebo hardwarová závada.

Dodatečné předávané parametry:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| physicalDeviceId | string nebo null | ne | ID fyzického zařízení. Pokud je `null`, problém není přiřazen konkrétnímu fyzickému zařízení v místnosti. |
| type | string (výčet níže) | ano | Typ selhání. |
| severity | string | ano | Závažnost: `warning` nebo `error`. |
| localizedDescription | string | ano | Uživatelský popis selhání. |
| isResolvableByPartner | bool | ano | Zda může partner označit problém jako vyřešený. |

`error` označuje problém, který může způsobit nefunkčnost a měl by být vyřešen co nejrychleji.
`warning` označuje upozornění, které nemusí vyžadovat okamžitý zásah.

Pokud je `isResolvableByPartner` nastaveno na `true`, může partnerská aplikace nabídnout uživateli možnost
označit problém jako vyřešený. Hodnota `false` může znamenat, že je nejprve potřeba fyzický zásah, například
výměna zařízení nebo baterie. Závažnost a možnost vyřešení vždy určují hodnoty konkrétní události.

Definované hodnoty `type`:

| type | Popis |
|:-----|:------|
| `inserted-discharged-battery` | Do zařízení byla vložena vybitá baterie. |
| `inserted-partially-discharged-battery` | Do zařízení byla vložena částečně nabitá baterie. |
| `generic-physical-device-error` | Obecná porucha fyzického zařízení. |
| `thermo-head-position-setting-failed` | Nepodařilo se nastavit požadovanou polohu hlavice. |

Pro označení podporovaného selhání za vyřešené slouží stávající endpoint `PUT api/device-failure/resolve`.
Verze 2 nemění pravidla vyhodnocování ani zasílání poruch.

Ukázka zaslané události:

```json
{
    "protocolVersion": 2,
    "roomId": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000004",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "device-failure",
    "physicalDeviceId": "abc123",
    "type": "generic-physical-device-error",
    "severity": "error",
    "localizedDescription": "Došlo k závadě na zařízení, proveďte prosím jeho výměnu.",
    "isResolvableByPartner": false
}
```

### EventType device-failure-resolved

Informuje o vyřešení dříve oznámeného selhání. Doporučujeme nejprve prostudovat událost `device-failure`.
Událost se odesílá při vyřešení detekovaném systémem; při vyřešení partnerem pomocí endpointu
`PUT api/device-failure/resolve` se tato událost neodesílá.

Dodatečné předávané parametry:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| physicalDeviceId | string nebo null | ne | ID fyzického zařízení, jehož problém byl vyřešen. Může být `null` pro problém bez konkrétního fyzického zařízení. |
| type | string | ano | Typ vyřešeného selhání; stejné hodnoty jako u `device-failure`. |

Ukázka zaslané události:

```json
{
    "protocolVersion": 2,
    "roomId": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000005",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "device-failure-resolved",
    "physicalDeviceId": "abc123",
    "type": "inserted-discharged-battery"
}
```

### EventType physical-device-replaced

Informuje o výměně fyzického zařízení přiřazeného k entitě. Zařízení `replacedPhysicalDeviceId` bylo
nahrazeno zařízením `replacementPhysicalDeviceId`, například při výměně porouchané hlavice.
Místnost určuje dvojice `entityType: "room"` a `entityId`.

Dodatečné předávané parametry:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| replacedPhysicalDeviceId | string | ano | ID původního fyzického zařízení. |
| replacementPhysicalDeviceId | string | ano | ID nového fyzického zařízení. |

Ukázka zaslané události:

```json
{
    "protocolVersion": 2,
    "entityType": "room",
    "entityId": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000006",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "physical-device-replaced",
    "replacedPhysicalDeviceId": "abc123",
    "replacementPhysicalDeviceId": "abc124"
}
```

### EventType physical-device-attached-to-entity

Informuje o přiřazení jednoho nebo více fyzických zařízení k entitě. Při první instalaci v místnosti obsahuje
jedna událost všechna přiřazená fyzická zařízení. Při pozdějším přiřazení jednotlivého zařízení má stejný formát
s jednou položkou v poli.

Tato událost nahrazuje původní `temperature-regulator-created` i `physical-device-attached-to-device`.
Neoznamuje vytvoření místnosti; k tomu slouží samostatné `room-created`.

Dodatečné předávané parametry:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| physicalDevices | physicalDevice[] | ano | Pole přiřazených fyzických zařízení. |

Objekt `physicalDevice` má následující formát:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| physicalDeviceId | string | ano | ID přiřazeného fyzického zařízení. |
| physicalDeviceType | string | ano | Typ fyzického zařízení: `thermo-head` nebo `thermometer` pro instalaci topení v místnosti. |

Termostat je pro účely tohoto kontraktu uváděn jako `thermometer`. `physicalDevices` je přímo pole,
neobsahuje obalový objekt `data`. Parametry `note` a `legacyPhysicalDeviceId` se ve verzi 2 nepředávají.

Ukázka zaslané události:

```json
{
    "protocolVersion": 2,
    "entityType": "room",
    "entityId": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000007",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "physical-device-attached-to-entity",
    "physicalDevices": [
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
}
```

### EventType heating-state-changed

Informuje o změně cílové teploty nebo stavu topení v místnosti, například o zahájení nebo ukončení předehřívání.
Změna může vzniknout požadavkem uživatele, partnerské aplikace nebo zpracováním plánu.

Dodatečné předávané parametry:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| targetTemperature | float nebo null | ano | Cílová teplota [°C]. Může být `null`, pokud není pro danou změnu určena. |
| changeReason | string (výčet níže) | ano | Důvod změny. |
| sourceRequestType | string nebo null | ne | Původ požadavku: `by-user` nebo `by-admin-app`. |
| sourceRequestId | string (UUID) nebo null | ne | ID původního požadavku, pokud je k dispozici. |

Hodnoty `changeReason`:

| changeReason | Popis |
|:-------------|:------|
| `pre-heating-started` | Začalo předehřívání místnosti. |
| `pre-heating-stopped` | Předehřívání skončilo. |
| `target-temperature-changed` | Změnila se cílová teplota. |
| `diagnostic-started` | Začal diagnostický režim. |
| `diagnostic-stopped` | Diagnostický režim skončil. |

Pokud změna navazuje na požadavek partnerské aplikace, `sourceRequestType` má hodnotu `by-admin-app` a
`sourceRequestId` může obsahovat `requestId` původního volání API. Při změně vyvolané tlačítkem termostatu
má `sourceRequestType` hodnotu `by-user`. U změn bez tohoto kontextu mohou být oba parametry `null`.

Ukázka zaslané události:

```json
{
    "protocolVersion": 2,
    "roomId": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000008",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "heating-state-changed",
    "targetTemperature": 25.5,
    "changeReason": "target-temperature-changed",
    "sourceRequestType": "by-admin-app",
    "sourceRequestId": "30a86332-7a75-4e55-8217-1b69c1d6b301"
}
```

### EventType physical-device-detached-from-entity

Informuje o odpojení fyzických zařízení od entity, například při odebrání nebo poruše hlavice v místnosti.
Místnost zůstává zachována. Aktuální překlad jednotlivého odpojení předává pole s jedním ID.

Dodatečné předávané parametry:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| physicalDeviceIds | string[] | ano | ID odpojených fyzických zařízení. |

Ukázka zaslané události:

```json
{
    "protocolVersion": 2,
    "entityType": "room",
    "entityId": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000009",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "physical-device-detached-from-entity",
    "physicalDeviceIds": [
        "abc123"
    ]
}
```

### EventType room-created

Informuje o vytvoření místnosti. Místnost může existovat ještě před instalací fyzických zařízení.
Tato událost jako jediná předává také informace o partnerovi, budově a podlaží, aby ji příjemce mohl zařadit
do příslušné hierarchie. Identifikátory určují příslušnost; názvy slouží k zobrazení.

Dodatečné předávané parametry:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| name | string | ano | Název místnosti. |
| partnerId | string (UUID) | ano | ID partnera, kterému budova patří. |
| buildingId | string (UUID) | ano | ID budovy. |
| buildingName | string | ano | Název budovy. |
| floorId | string (UUID) | ano | ID podlaží. |
| floorName | string | ano | Název podlaží. |

Ukázka zaslané události:

```json
{
    "protocolVersion": 2,
    "roomId": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000010",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "room-created",
    "partnerId": "9381f1af-4187-4478-8c80-391c8882f421",
    "buildingId": "fdc2ea0d-d0f9-4d2f-aeca-5116f514093f",
    "buildingName": "Administrativní budova",
    "floorId": "d77c48f1-f7f3-4a02-b075-69b989914463",
    "floorName": "1. patro",
    "name": "Kancelář 101"
}
```

### EventType room-renamed

Informuje o změně názvu místnosti. Identifikátor `roomId` se přejmenováním nemění.

Dodatečné předávané parametry:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| name | string | ano | Nový název místnosti. |

Ukázka zaslané události:

```json
{
    "protocolVersion": 2,
    "roomId": "d65f1ffb-aa60-4eff-9666-78a93a048b17",
    "eventId": "c4056fc4-d433-4d2c-bb7f-000000000011",
    "eventTime": "2026-09-23T14:12:38Z",
    "eventType": "room-renamed",
    "name": "Zasedací místnost"
}
```

## Přechod z verze 1

Přehled hlavních změn:

| Verze 1 | Verze 2 |
|:--------|:--------|
| `protocolVersion: 1` | `protocolVersion: 2`. |
| `deviceId` označuje regulátor. | `roomId` označuje místnost; u přiřazení, odpojení a výměny se používá `entityId` s `entityType`. |
| `deviceType: "temperature-regulator"` | Parametr `deviceType` se v těle události nepředává. |
| `measured-humidity-temperature` | `measured-humidity-temperature-in-a-room`. |
| `temperature-regulator-created` | `physical-device-attached-to-entity` s více fyzickými zařízeními. |
| `physical-device-attached-to-device` | `physical-device-attached-to-entity` s polem fyzických zařízení. |
| `physical-device-detached-from-device` | `physical-device-detached-from-entity` s polem `physicalDeviceIds`. |
| `physicalDevices.data` při vytvoření regulátoru | Přímé pole `physicalDevices` při přiřazení k entitě. |
| `note`, `legacyPhysicalDeviceId` | Nepředávají se. |

Názvy událostí `thermo-head-changed-position`, `battery-alert`, `device-failure`, `device-failure-resolved`,
`physical-device-replaced`, `heating-state-changed`, `room-created` a `room-renamed` zůstávají zachovány.
Jejich identifikační parametry a přesný obsah odpovídají výše uvedeným popisům verze 2.
