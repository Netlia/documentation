# API pro místnosti a zařízení

Dokument popisuje partnerské REST API pro regulaci teploty v místnostech, plánování vytápění, ovládání termostatických hlavic a výměnu zařízení.
Požadavky i odpovědi s daty používají JSON. Adresu serveru a přístupové údaje poskytne zástupce Netlia.
Všechny uvedené cesty jsou relativní k adrese serveru.

Interaktivní dokumentace je dostupná na `/swagger/partner`, její OpenAPI popis na `/swagger/partner/swagger.json`.
Přístup ke stránce dokumentace vyžaduje přihlášení oprávněného uživatele.
Události zasílané partnerovi popisuje [dokumentace událostí](../EventForwarding/EventForwarding_CZ_room_v2.md).

## Obsah

Obsah zobrazíte následujícím způsobem:

![content](../images/table-of-contents.webp "Content")

## Zabezpečení

Komunikace je zabezpečena bearer tokenem ve formátu JWT. Token získáte u zástupce Netlia a posíláte jej
v hlavičce `Authorization` při každém volání API.

| Header Key | Header Value |
|:-----------|:-------------|
| Authorization | Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9... |
| Content-Type | application/json |

Hlavičku `Content-Type` posílejte u požadavků s JSON tělem.

## Chybové odpovědi

### Chyby 5xx

Chyby 5xx znamenají problém na straně serveru. Mohou být dočasné. Pokud přetrvávají, kontaktujte zástupce Netlia.
Před opakováním požadavku zohledněte pravidla v části [Idempotency-Key a opakované volání endpointů](#idempotency-key-a-opakované-volání-endpointů).

### Chyby 4xx

| HTTP kód | Význam |
|:---------|:-------|
| 400 Bad Request | Neplatné vstupy, chybějící nebo neplatná hlavička `Idempotency-Key`, neexistující místnost nebo místnost bez nakonfigurované regulace. Podrobnosti jsou v těle odpovědi. |
| 401 Unauthorized | Chybějící nebo neplatný bearer token. |
| 403 Forbidden | Klient nemá oprávnění k operaci. |
| 404 Not Found | URL neodpovídá žádnému endpointu. |

Požadavek s chybou 400 je před opakováním potřeba opravit. Pokud obdržíte `429 Too Many Requests`, respektujte
hlavičku `Retry-After`, pokud je uvedena.

### Tělo chybových odpovědí

Chybové odpovědi používají formát Problem Details:

| Parametr | Typ | Popis |
|:---------|:----|:------|
| type | string | Odkaz na popis typu chyby. |
| title | string | Obecný název chyby. |
| status | int | HTTP stavový kód. |
| instance | string | Cesta požadavku, pokud je uvedena. |
| detail | string | Konkrétní popis chyby. |
| errorCode | integer | Číselný identifikátor chyby. U chyb validace vstupu má hodnotu `400`, u serverové chyby `500`. |
| errors | object | Nepovinný slovník validačních chyb: název pole a pole textových zpráv. |
| traceId | string | Nepovinný identifikátor požadavku pro dohledání problému. |

Pro programové rozlišení chyb používejte `errorCode`, nikoli `title`.
Kódy obchodních chyb jsou odvozené od typu chyby a mohou přesahovat rozsah 32bitového čísla.

Příklad validační chyby (text zprávy je ilustrativní):

```json
{
  "title": "Bad Request",
  "status": 400,
  "detail": "One or more validation errors occurred.",
  "errorCode": 400,
  "errors": {
    "Idempotency-Key": ["The Idempotency-Key field is required."]
  }
}
```

## Idempotency-Key a opakované volání endpointů

Všechny endpointy v tomto dokumentu kromě `GET` vyžadují hlavičku `Idempotency-Key`.
Hodnota musí být platné UUID. Chybějící, prázdná nebo neplatná hodnota způsobí odpověď `400 Bad Request`.
Identifikátor se posílá výhradně v hlavičce, nikoli v JSON těle požadavku.

Požadavky `GET` hlavičku nevyžadují a nijak ji nezpracovávají. Čtení dat stav systému nemění, proto je opakované
volání vždy bezpečné.

| Header Key | Header Value |
|:-----------|:-------------|
| Idempotency-Key | b5e5a8e4-d09d-4d0f-8878-5ab24c2647fc |

Příklad celého požadavku:

```http
PUT /api/room/f47ac10b-58cc-4372-a567-0e02b2c3d479/temperature HTTP/1.1
Authorization: Bearer <token>
Idempotency-Key: b5e5a8e4-d09d-4d0f-8878-5ab24c2647fc
Content-Type: application/json

{"targetTemperature":21.5}
```

### Význam identifikátoru

`Idempotency-Key` slouží jako klíč idempotence: umožňuje zopakovat stejný požadavek, aniž by byla jeho změna provedena
vícekrát. Pro každý nový požadavek vytvořte nové UUID. Při opakování stejného požadavku zachovejte stejnou metodu,
URL, tělo i `Idempotency-Key`. Stejný identifikátor nepoužívejte pro jinou operaci ani pro změněné vstupy.

Pokud server požadavek s daným klíčem již úspěšně zpracoval, při opakování se stejným `Idempotency-Key` neprovede
změnu znovu a přehraje původní odpověď: vrátí stejný stavový kód i stejné tělo jako při prvním zpracování.
To chrání například před opakovaným vytvořením plánovaných změn teploty nebo opakovanou výměnou zařízení
po výpadku spojení.

Klíče jsou uchovávány 24 hodin od prvního zpracování požadavku. Po uplynutí této doby již server opakování
nerozpozná a požadavek zpracuje jako nový. Opakované pokusy proto provádějte v rámci tohoto okna.

U okamžitých změn teploty se identifikátor předává do související události `heating-state-changed` jako `sourceRequestId`.
U ostatních operací nelze předpokládat jeho vrácení v události.

### Opakování na straně klienta

Po síťové chybě nebo chybě 5xx zopakujte stejný požadavek se stejným `Idempotency-Key`. Pokud nebyl úspěšně zpracován,
server se jej pokusí zpracovat znovu. Pokud již dokončen byl, přehraje jeho původní odpověď bez opakovaného provedení
změny. Pro opakované pokusy používejte rostoucí prodlevu a omezte jejich počet.

Po chybě 400 opravte vstupy a odešlete nový požadavek s novým `Idempotency-Key`. Při odpovědi `429 Too Many Requests`
respektujte hlavičku `Retry-After`, pokud je uvedena.

## Základní datové typy

### Čas

#### UTC

Absolutní čas se zapisuje v ISO 8601 s označením UTC `Z`, například `2027-10-09T14:12:38Z`.

#### ZonedDateTime

Plánování používá lokální čas společně s časovým pásmem IANA:

```json
{
  "ianaTimeZone": "Europe/Prague",
  "localDateTime": "2027-10-21T15:00:00"
}
```

`localDateTime` je čas na místních hodinách bez přípony `Z` a bez UTC offsetu.
`ianaTimeZone` určuje, ve kterém pásmu se čas vyhodnotí, včetně pravidel letního a zimního času.
Obě položky jsou povinné. Neznámé časové pásmo je odmítnuto.

### UUID

Formát: `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`, například `f47ac10b-58cc-4372-a567-0e02b2c3d479`.

### Teplota

Teploty jsou desetinná čísla ve stupních Celsia. V JSON používejte desetinnou tečku.
Možnost předat `null` je uvedena u konkrétního parametru. Povinná položka musí být v těle přítomna i tehdy,
když její hodnota může být `null`.

## Popis endpointů

Každý endpoint uvádí hlavičky, které je nutné poslat. Všechny je vyžadují ve stejné podobě:
`Authorization` s bearer tokenem, `Content-Type` pro JSON tělo a `Idempotency-Key` s novým UUID pro každý
nový požadavek. Význam `Idempotency-Key` popisuje
[Idempotency-Key a opakované volání endpointů](#idempotency-key-a-opakované-volání-endpointů).
Změny mohou být zpracovávány asynchronně. Úspěšná odpověď neznamená, že již byla v místnosti dosažena požadovaná
teplota nebo že zařízení již provedlo příkaz.

### POST api/room/schedule-temperature

Naplánuje cílové teploty pro jednu nebo více místností, případně s předehříváním.

Hlavičky požadavku:

```http
Authorization: Bearer <token>
Content-Type: application/json
Idempotency-Key: b5e5a8e4-d09d-4d0f-8878-5ab24c2647fc
```

Předávané parametry v těle:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| targetTemperatures | ScheduleTargetTemperature[] | ano | Seznam plánovaných změn. |

Objekt `ScheduleTargetTemperature`:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| roomId | string (UUID) | ano | ID místnosti. |
| targetTemperature | float | ano | Cílová teplota v °C; nesmí být `null`. |
| reachTargetTemperatureByThisTime | ZonedDateTime | ano | Čas dosažení cílové teploty při předehřívání, jinak čas změny cíle. |
| regulationType | string | ano | `standard-with-pre-heating` nebo `standard-without-pre-heating`. |
| scheduleId | string (UUID) | ano | Identifikátor předávaný s položkou plánu. Neslouží jako klíč idempotence požadavku. |

Ukázka requestu:

```json
{
  "targetTemperatures": [
    {
      "roomId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "targetTemperature": 22.5,
      "reachTargetTemperatureByThisTime": {
        "ianaTimeZone": "Europe/Prague",
        "localDateTime": "2027-10-21T15:00:00"
      },
      "regulationType": "standard-with-pre-heating",
      "scheduleId": "c0a80121-5ef8-492f-b3a1-56c65f0dcf19"
    }
  ]
}
```

Ukázka response:

```text
200 OK, žádné informace v body.
```

#### Poznámky

* Čas musí ležet v budoucnosti v uvedeném časovém pásmu.
* Místnosti a časy všech položek se ověří před předáním plánu ke zpracování.
* `standard-without-pre-heating` změní cílovou teplotu až v naplánovaný čas. Dosažení teploty může trvat déle.
* `standard-with-pre-heating` umožňuje začít topit předem, aby bylo cílové teploty dosaženo v naplánovaný čas.
  Skutečný výsledek závisí na podmínkách v místnosti a možnostech vytápění.
* Pro plánované snížení teploty použijte `standard-without-pre-heating`.
* Okamžitá změna cílové teploty může ukončit právě probíhající předehřívání.
* Při opakování stejného plánování zachovejte `Idempotency-Key`. Samotné `scheduleId` nenahrazuje klíč idempotence požadavku.

**Příklad příchodu a odchodu hosta:**

Pro příchod v 11:00 naplánujte 22 °C s `standard-with-pre-heating`.
Pro odchod v 17:00 naplánujte 18 °C s `standard-without-pre-heating`.
Systém se pokusí místnost vyhřát před příchodem a při odchodu sníží cílovou teplotu.

### POST api/room/{roomId}/abort-scheduled-temperatures

Zruší dosud nezpracované změny cílové teploty dané místnosti, jejichž plánovaný čas je **po** zadaném čase.
Záznamy přesně v tomto čase se neruší. Operace vybírá pouze záznamy se stejným časovým pásmem jako v požadavku;
použijte proto stejné `ianaTimeZone` jako při plánování. Již zpracovávané nebo dokončené záznamy se nemění.
`roomId` v URL je UUID místnosti.

Hlavičky požadavku:

```http
Authorization: Bearer <token>
Content-Type: application/json
Idempotency-Key: b5e5a8e4-d09d-4d0f-8878-5ab24c2647fc
```

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| abortAfterTime | ZonedDateTime | ano | Dolní časová hranice pro zrušení nezpracovaných změn. |

Ukázka requestu:

```json
{
  "abortAfterTime": {
    "ianaTimeZone": "Europe/Prague",
    "localDateTime": "2027-10-21T12:00:00"
  }
}
```

Ukázka response (`200 OK`):

```json
{"abortedCount":2}
```

`abortedCount` udává počet zrušených záznamů; může být `0`.

### PUT api/room/temperature

Předá okamžité změny cílové teploty pro více místností. Každá místnost může mít jinou cílovou teplotu.

Hlavičky požadavku:

```http
Authorization: Bearer <token>
Content-Type: application/json
Idempotency-Key: b5e5a8e4-d09d-4d0f-8878-5ab24c2647fc
```

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| targetTemperatures | TargetTemperature[] | ano | Seznam změn. |

Objekt `TargetTemperature`:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| roomId | string (UUID) | ano | ID místnosti. |
| targetTemperature | float nebo null | ano | Cílová teplota v °C. |

Ukázka requestu:

```json
{
  "targetTemperatures": [
    {"roomId":"f47ac10b-58cc-4372-a567-0e02b2c3d479","targetTemperature":21.5},
    {"roomId":"d65f1ffb-aa60-4eff-9666-78a93a048b17","targetTemperature":19.0}
  ]
}
```

Ukázka response:

```text
200 OK, žádné informace v body.
```

Pokud některá místnost neexistuje nebo nemá nakonfigurovanou regulaci, požadavek je odmítnut před předáním změn.
To neznamená, že zařízení provedou všechny změny současně.

### POST api/entity/replace-device

Vymění zařízení přiřazené k entitě. Entitu určuje objekt `entity` v těle požadavku.
Náhradní zařízení musí být registrované v Netlia, dostupné pro instalaci a vhodného typu pro výměnu.
Vyměňované zařízení musí patřit k uvedené entitě.

Hlavičky požadavku:

```http
Authorization: Bearer <token>
Content-Type: application/json
Idempotency-Key: b5e5a8e4-d09d-4d0f-8878-5ab24c2647fc
```

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| entity | Entity | ano | Entita, ve které je zařízení nainstalováno. |
| replacedDeviceId | string (UUID) | ano | ID vyměňovaného zařízení. |
| replacementDeviceId | string (UUID) | ano | ID náhradního zařízení. |

Objekt `Entity` má stejnou podobu jako `entity` v zasílaných událostech:

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| id | string (UUID) | ano | ID entity. Pro `type: "room"` jde o ID místnosti. |
| type | string | ano | Typ entity: `room`, `riser` nebo `building-side`. |

Ukázka requestu:

```json
{
  "entity": {
    "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "type": "room"
  },
  "replacedDeviceId": "6e748f20-846e-4e89-a831-000000000001",
  "replacementDeviceId": "6e748f20-846e-4e89-a831-000000000002"
}
```

Ukázka response:

```text
200 OK, žádné informace v body.
```

O výměně je partner informován událostí `device-replaced`.

### PUT api/thermo-heads/turn-off-regulation

Vypne regulaci vybraných termostatických hlavic, případně do zadaného času. Před vypnutím lze nastavit jejich polohu.

Hlavičky požadavku:

```http
Authorization: Bearer <token>
Content-Type: application/json
Idempotency-Key: b5e5a8e4-d09d-4d0f-8878-5ab24c2647fc
```

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| deviceIds | string (UUID)[] | ano | ID hlavic. Seznam musí obsahovat alespoň jednu hlavici. |
| turnedOffUntil | string (UTC čas) nebo null | ne | Čas automatického obnovení regulace; musí být v budoucnosti. Bez hodnoty zůstane regulace vypnutá do opětovného zapnutí. |
| positionBeforeTurnOff | integer nebo null | ne | Poloha hlavic před vypnutím, od 1 do 99. |

Ukázka requestu:

```json
{
  "deviceIds": ["6e748f20-846e-4e89-a831-000000000004"],
  "turnedOffUntil": "2027-10-21T16:00:00Z",
  "positionBeforeTurnOff": 50
}
```

Ukázka response:

```text
200 OK, žádné informace v body.
```

Všechna ID musí označovat existující hlavice. Duplicitní ID se zpracuje pouze jednou.
Čekající nebo nepotvrzené příkazy změny polohy se zruší; pokud je uvedeno `positionBeforeTurnOff`, zařadí se nový příkaz.

### PUT api/thermo-heads/turn-on-regulation

Zapne regulaci vybraných termostatických hlavic.

Hlavičky požadavku:

```http
Authorization: Bearer <token>
Content-Type: application/json
Idempotency-Key: b5e5a8e4-d09d-4d0f-8878-5ab24c2647fc
```

| Parametr | Typ | Povinný | Popis |
|:---------|:----|:--------|:------|
| deviceIds | string (UUID)[] | ano | ID hlavic. Seznam musí obsahovat alespoň jednu hlavici. |

Ukázka requestu:

```json
{
  "deviceIds": ["6e748f20-846e-4e89-a831-000000000004"]
}
```

Ukázka response:

```text
200 OK, žádné informace v body.
```

Všechna ID musí označovat existující hlavice. Duplicitní ID se zpracuje pouze jednou.
