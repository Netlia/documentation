# API
Bude se jednat o REST API, bude k dispozici Swagger.

# Zabezpečení
Komunikace bude zabezpečena bearer tokenem v hlavičce, informace budou doplněny.

Většina endpointů přijímajících data v těle requestu vyžaduje položku `RequestId`. Jedná se o jednoznačný identifikátor requestu, který generuje partner (jedná se o UUID) a slouží pro zajištění idempotence, čímž je docíleno toho, že akce je provedena jen jednou pokud dojde o opakovanému zavolání v určitém časovém intervalu. Při opakovaném požadavku se stejným `RequestId` je vrácen HTTP status `409 Conflict`.

# Chybové odpovědi
Chyby 5xx jsou způsobené chybou serveru a neměli by nikdy nastat. Pokud server vratí tuto chybu, měl by být informován zástupce Netlia.

Chyby 4xx jsou vráceny, pokud klient provedl neplatný/nevalidní request.

Všechny chybové stavové kódy (4xx a 5xx) obsahují standardní [problem details body](https://datatracker.ietf.org/doc/html/rfc7807), které má následující formát:

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
* Type - odkazuje na podrobné vysvětlení erroru. Pokud netlia nemá žádné specifické detaily k danému erroru, tak obsahuje pouze odkaz na vysvětlení http kódu.
* Title - obsahuje textový popis chyby. V případě obecných chyb popis odpovídá popisu stavového kódu. V případě specifických chyb obsahuje konkrétní informace (příklad v ukázce).
* Status - duplikuje stavový kód odpovědi. Toto pole je v body obsaženo pouze pro zjedodušení práce partnera (např. pokud loguje body a neloguje vrácený http kód).
* TraceId - slouží k jednoznačné identifikaci konkrétní chyby (typicky při nahlášení chybného chování partnerem).
* ErrorId - číselný identifikátor chyby. Každá chyba má svůj identifikátor, který může být použit partnerem při programovém zpracování chyby. Pro obecné chyby ErrorId odpovídá http statusu.
* Errors - je nepovinné pole, které obsahují pouze odpovědi vracející více než jednu chybu. Obsahuje slovník, kde klíčem je řetězec který logicky spojuje pole chyb, které následuje za ním. Viz. příklad č. 3.

> **Pokud zpracováváte konkrétní chybu na klientovi tak nespoléhejte na hodnotu v Title. Namísto toho vždy použijte ErrorId.**

Příklady chybových responses:

1. Server vrátil chybu 500. Je potřeba informovat firmu Netlia aby chybu opravila:
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
  "status": 404,
  "errorId": 1,
  "traceId": "00-eab978ed39bb58b120c99c08ef42a6a2-aca7bff9ea0a470c-00"
}
```

3. Server vrátil chybu 400. Klient se snaží odeslat nevalidní JSON (errorId s hodnotou 400 je vrácen vždy když klient odešle JSON který není syntakticky správně):

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


# Zařízení a podporované endpointy

## Teplotní regulátor

### PUT api/temperature-regulator/{DeviceId}/mode

Změna módu teplotního regulátoru.

Předávané parametry:

| Parametr    | Typ         | Povinný | Popis                               |
|:------------|:------------|:--------|:------------------------------------|
| RequestId   | string      | ano     | Jednoznačný identifikátor requestu. |
| Mode        | string      | ano     | Cílový mód zařízení.                |

Cílový mód zařízení může nabývat těchto hodnot:

| Hodnota            | Název                       |
|--------------------|-----------------------------|
| 0              | Základní regulace teploty.  |
| 1              | Letní režim.                |

Ukázka requestu:

```yaml
{
    "RequestId": "b5e5a8e4-d09d-4d0f-8878-5ab24c2647fc",
    "Mode": 0
}
```

Ukázka response: 

```
200 OK, žádné informace v body.
```

### GET api/temperature-regulator/{DeviceId}/mode

Zjištění módu teplotního regulátoru.

Ukázka response:

```yaml
{
    "Mode": 0
}
```

### PUT api/temperature-regulator/{DeviceId}/temperature

Nastavení cílové teploty pro regulaci.
Podporováno pouze, pokud mód regulátoru je `basic`.

Předávané parametry:

| Parametr           | Typ         | Povinný | Popis                               |
|:-------------------|:------------|:--------|:------------------------------------|
| RequestId          | string      | ano     | Jednoznačný identifikátor requestu. |
| TargetTemperature  | float       | ano     | Cílová teplota.                     |

Ukázka requestu:

```yaml
{
    "RequestId": "b5e5a8e4-d09d-4d0f-8878-5ab24c2647fc",
    "TargetTemperature": 21.5
}
```

Ukázka response: 

```
200 OK, žádné informace v body.
```

### GET api/temperature-regulator/{DeviceId}/temperature

Zjištění cílové teploty pro regulaci.
Podporováno pouze, pokud mód regulátoru je `basic`.

Ukázka response:

```yaml
{
    "TargetTemperature": 21.5
}
```


### PUT api/temperature-regulator/{DeviceId}/factory-reset

Zruší zařízení, což umožní znovu spárovat jednotlivé fyzické komponenty. Dojde k odstranění provozních dat.

Ukázka requestu:

```yaml
{
    "RequestId": "b5e5a8e4-d09d-4d0f-8878-5ab24c2647fc"
}
```

Ukázka response:

```yaml
{
    "ReleasedPhysicalDeviceIds": ["abc123", "abc456", "abc789"]
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
