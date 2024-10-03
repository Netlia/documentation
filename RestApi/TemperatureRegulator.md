# API

Dokument popisuje API pro komunikaci se zařízením temperature regulator. Swagger k těmto endpointům je
dostupný [zde](https://public-api.netlia.com/swagger/temperature-regulator/index.html). Přístupové údaje ke swaggeru je
možné získat od zástupce Netlia.

## Zabezpečení

Komunikace je zabezpečna bearer tokenem ve formátu JWT. Token se posílá ve standartním headeru s klíčem Authorization.
Příklad headeru:

| Header Key    | Header Value                                                |
|:--------------|:------------------------------------------------------------|
| Authorization | Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJodHRwOi8v... | 

Token je možné získat u zástupce Netlia.

## Chybové odpovědi

### Chyby 5xx

Chyby 5xx jsou způsobené chybou serveru a neměly by nastavávat. Obvykle jsou způsobeny dočasným výpadkem serveru, nebo chybou v kódu.
Pokud se klientovi vrátí tato chyba, měl by zkusit zopakovat request. Jak správně zopakovat request je popsáno [níže](#requestid-a-opakované-volání-endpointů).

Pokud chyba přetrvává tak by měl být kontaktován zástupce firmy Netlia.

### Chyby 4xx

Chyby 4xx jsou vráceny, pokud klient provedl neplatný/nevalidní request. Nejčastější chyby na které klient narazí jsou:

* 400 Bad Request - Tělo requestu nobsahuje validní formát.
* 404 Not Found - Url je naplatná a neodpovídá žádnému endpointu.
* 422 Unprocessable Content - Request obsahuje validní formát, ale nemohl být proveden. Například pokud klient požaduje
  vytvoření zařízení, které již existuje.

### Tělo chybových odpovědí

Všechny chybové stavové kódy (4xx a 5xx) obsahují
standardní [problem details](https://datatracker.ietf.org/doc/html/rfc7807) body, které má následující formát:

```yaml
{
  "type": string,
  "title": string,
  "status": int,
  "traceId": string,
  "errorId": int,
  "errors": {
    string: [string, string, ...],
    string: [string, string, ...],
    string: [string, string, ...],
    ...
  }
}
```

* Type - odkazuje na podrobné vysvětlení erroru. Pokud server nemá žádné specifické detaily k danému erroru, tak
  obsahuje pouze odkaz na vysvětlení http kódu.
* Title - obsahuje textový popis chyby. V případě, že server nemá více informací je zde uveden popis stavového kódu. V  případě  specifických chyb obsahuje konkrétní informace (příklad v ukázce).
* Status - duplikuje stavový kód odpovědi. Toto pole je v body obsaženo pouze pro zjedodušení práce partnera (např.
  pokud loguje body a neloguje vrácený http kód).
* TraceId - slouží k jednoznačné identifikaci konkrétní chyby (typicky použito při nahlášení chybného chování
  partnerem).
* ErrorId - číselný identifikátor typu chyby. Každý druh chyby má svůj identifikátor, který může být použit partnerem při programovém zpracování chyby.
* Errors - je nepovinné pole, které obsahují pouze odpovědi vracející více než jednu chybu. Obsahuje slovník, kde klíčem
  je řetězec, který logicky spojuje pole chyb, které následuje za ním. Viz. příklad č. 3.

> **Pokud zpracováváte konkrétní chybu na klientovi, nespoléhejte na hodnotu v Title. Namísto toho vždy použijte
ErrorId.**

Příklady chybových responses:

1. Server vrátil chybu 500. Pro vyřešení problému zkuste retry, nebo kontaktujte zástupce Netlia.

```json
{
  "type": "https://httpstatuses.io/500",
  "title": "Internal Server Error",
  "status": 500,
  "errorId": 500,
  "traceId": "00-1ecf9c21495b20af2e8aac4c71653a57-7c4329e49f308038-00"
}
```

2. Server vrátil 404. Klient se v tomto případě snaží pracovat s neexistujícím device.

```json
{
  "type": "https://httpstatuses.io/400",
  "title": "Device not found",
  "status": 422,
  "errorId": 1,
  "traceId": "00-eab978ed39bb58b120c99c08ef42a6a2-aca7bff9ea0a470c-00"
}
```

3. Server vrátil chybu 400. Klient se snaží odeslat nevalidní JSON (errorId s hodnotou 400 je vrácen vždy když klient
   odešle JSON, který není syntakticky správně):

```json
{
  "errors": {
    "serial": [
      "Unexpected end when deserializing object. Path 'serial', line 3, position 21."
    ],
    "newDevice": [
      "The newDevice field is required."
    ]
  },
  "type": "https://httpstatuses.io/400",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errorId": 400,
  "traceId": "00-3f89119d4e33d8d706194838c4b8dc50-558f2c77a8cf08c5-00"
}
```

4. Server vrátil chybu 404. Klient se snaží zavolat neexistující URL.

```json
{
  "type": "https://httpstatuses.io/404",
  "title": "Url is incorrect. Page not found.",
  "status": 404,
  "errorId": 404,
  "traceId": "00-eab978ed39bb58b120c99c08ef42a6a2-aca7bff9ea0a470c-00"
}
```

## RequestId a opakované volání endpointů

Mnoho problémů s nefungujícím API může být jednoduše vyřešeno opakovaným zavoláním requestu. Endpointy ale často
nemohou být volány opakovaně, jelikož to může vést k nežádoucím výsledkům - typicky uváděným problémem je dvojité vytvoření platby klientem.

GET volání tímto problémem v našem API netrpí, jelikož vždy pouze získávají data a nikdy je nemění. Problém nastává u
volání, které mění stav systému.

Abychom takovým problémům předešli, tak naše API implementuje koncept `requestId`, který zajišťje, že request může být opakován ale bude proveden pouze jednou. Všechny endpointy, které implementují tento koncept používají HTTP metodu PUT nebo
DELETE.

### Jak funguje requestId

Každý request, který mění stav systému musí obsahovat v těle requestu položku `requestId`. Jedná se o jednoznačný
identifikátor.
Pokud klient odešle request s `requestId`, který již byl použit v nedávné době (24h), tak server nezpracovává request
znovu ale pouze
vrátí výsledek předchozího volání.

Vyjímkou jsou stavové kódy 5xx. Pokud server vrátí 5xx a klient volání zopakuje se stejným `requestId` tak se server
pokusí
request zpracovat znovu a vrátí výsledek.

Pokud klient provede více konkurentních requestů se stejným `requestId`, tak server zpracuje pouze jeden z nich a
pro ostatní vrátí stavový kód `409 Conflict`.

### Implementace retry na staraně klienta

Pokud server vrátí stavový kód `4xx`, tak je chyba u klienta a nemá význam request opakovat.

V případě, že nastane jakákoliv jiná chyba, tak by měl klient request zopakovat a neměnit `requestId`.

## Popis endpointů

### PUT api/temperature-regulator/{deviceId}/mode

Změna módu teplotního regulátoru. V zimním módu zařízení udržuje cílovou teplotu podle nastavení. 
V letním režimu zařízení nereguluje teplotu a provádí udržovací operace jako 
je šetření baterie a protočení hlavice jednou za čas aby se zamezilo zatuhnutí hlavice.

Předávané parametry:

| Parametr  | Typ    | Povinný | Popis                               |
|:----------|:-------|:--------|:------------------------------------|
| requestId | string | ano     | Jednoznačný identifikátor requestu. |
| mode      | int    | ano     | Cílový mód zařízení.                |

Cílový mód zařízení může nabývat těchto hodnot:

| Hodnota | Název                      |
|---------|----------------------------|
| basic   | Základní regulace teploty. |
| summer  | Letní režim.               |

Ukázka requestu:

```yaml
{
    "requestId": "b5e5a8e4-d09d-4d0f-8878-5ab24c2647fc",
    "mode": "basic"
}
```

Ukázka response:

```
200 OK, žádné informace v body.
```

### GET api/temperature-regulator/{deviceId}/mode

Zjištění módu teplotního regulátoru.

Ukázka response:

```yaml
{
    "mode": "basic"
}
```

### PUT api/temperature-regulator/{deviceId}/temperature

Nastavení cílové teploty pro regulaci.
Pokud je zařízení v režimu `summer` tak se nic neprovede a vrátí se uspěšná odpověď.

Předávané parametry:

| Parametr          | Typ    | Povinný | Popis                               |
|:------------------|:-------|:--------|:------------------------------------|
| requestId         | string | ano     | Jednoznačný identifikátor requestu. |
| targetTemperature | float  | ano     | Cílová teplota.                     |

Ukázka requestu:

```yaml
{
    "requestId": "b5e5a8e4-d09d-4d0f-8878-5ab24c2647fc",
    "targetTemperature": 21.5
}
```

Ukázka response:

```
200 OK, žádné informace v body.
```

### PUT api/temperature-regulator/temperature

Nastavení cílové teploty pro regulaci na více zařízeních.
Pokud je zařízení v režimu `summer` tak se nic neprovede a vrátí se uspěšná odpověď.

Předávané parametry:

| Parametr           | Typ                 | Povinný | Popis                                    |
|:-------------------|:--------------------|:--------|:-----------------------------------------|
| requestId          | string              | ano     | Jednoznačný identifikátor requestu.      |
| targetTemperatures | targetTemperature[] | ano     | Nastavení cílových teplot na zaříyeních. |

Objekt targetTemperature:

| Parametr          | Typ    | Povinný | Popis                   |
|:------------------|:-------|:--------|:------------------------|
| deviceId          | string | ano     | Identifikátor zařízení. |
| targetTemperature | float  | ano   | Cílová teplota.       |

Ukázka requestu:

```yaml
{
    "requestId": "b5e5a8e4-d09d-4d0f-8878-5ab24c2647fc",
    "targetTemperatures": [
      {
        "targetTemperature": 21.5,
        "deviceId": "b5e5a8e4-d09d-4d0f-8878-5ab24c2247fc"     
      },
      {
        "targetTemperature": 21.5,
        "deviceId": "b5e5a8e4-d09d-4d0f-8878-5ab24c2547fc"
      }
      // ...
    ]
}
```

Ukázka response:

```
200 OK, žádné informace v body.
```

### GET api/temperature-regulator/{deviceId}/temperature

Zjištění cílové teploty pro regulaci.

Ukázka response:

```yaml
{
    "targetTemperature": 21.5 // may be null
}
```

### PUT api/temperature-regulator/{deviceId}/factory-reset

Zruší zařízení, což umožní znovu spárovat jednotlivé fyzické komponenty. Dojde k odstranění provozních dat.

Ukázka requestu:

```yaml
{
    "requestId": "b5e5a8e4-d09d-4d0f-8878-5ab24c2647fc"
}
```

Ukázka response:

```yaml
{
    "releasedPhysicalDeviceIds": ["abc123", "abc456", "abc789"]
}
```

<!--
### PUT api/temperature-regulator/

Vytvoří požadavek na zavedení nového zařízení do systému. Po úspěšném vytvoření zařízení je partner informován eventem `device-created` - viz. dokumentace k předávání událostí.

Předávané parametry:

| Parametr           | Typ         | Povinný | Popis                                                              |
|:-------------------|:------------|:--------|:-------------------------------------------------------------------|
| RequestId          | string      | ano     | Jednoznačný identifikátor requestu.                                |
| DeviceType         | string      | ano     | Typ zařízení.                                                      |
| PhysicalDeviceIds  | string[]    | ano     | Fyzická zařízení / komponenty ze kterých je zařízení složeno.      |
| CustomData         | object      | ne      | Objekt s informacemi specifickými dle partnera.                    |

Ukázka requestu:

```yaml
{
    "RequestId": "b5e5a8e4-d09d-4d0f-8878-5ab24c2647fc",
    "DeviceType": "temperature-regulator",
    "PhysicalDeviceIds": ["abc123", "abc456", "abc789"],
    "CustomData":
        {
            "sample-key-1": "sample-value-1",
            "sample-key-2": 123 
        }
}
```

Ukázka response: 

```yaml
{
    "DeviceId": "7a3945e9-89c6-4464-9feb-6642c97035b2",
    "PhysicalDeviceIds": ["abc123", "abc456", "abc789"]
}
```
-->
