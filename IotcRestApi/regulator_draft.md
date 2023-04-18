# API
Bude se jednat o REST API, bude k dispozici Swagger.

# Zabezpečení
Komunikace bude zabezpečena bearer tokenem v hlavičce, informace budou doplněny.

Většina endpointů přijímajících data v těle requestu vyžaduje položku `requestId`. Jedná se o jednoznačný identifikátor requestu, který generuje partner (jedná se o GUID) a slouží pro zajištění idempotence, čímž je docíleno toho, že akce je provedena jen jednou pokud dojde o opakovanému zavolání v určitém časovém intervalu.

# Chybové odpovědi
Chyby 5xx jsou způsobené chybou serveru a neměli by nikdy nastat. Pokud server vratí tuto chybu tak
by měl být okamžitě informován zástupce z firmy Netlia.

Chyby 4xx jsou vráceny, pokud klient provedl neplatný/nevalidní request. Vysvětlení stavových kódu je možné najít
v HTTP dokumentaci - https://developer.mozilla.org/en-US/docs/Web/HTTP/Status#client_error_responses. Podporované stavové kódy budou doplněny.

V některých případech 4xx error obsahuje body s dalšimy informacemi o chybě. Body má následující formát:

```yaml
{
    "errors" : [
        {
            "errorCode":int
            "description":string
            "additionalInformation":string
            "traceId":string
        }
    ]
}
```

# Zařízení a podporované endpointy

## Teplotní regulátor

### POST api/temperature-regulator/{DeviceId}/mode

Změna módu teplotního regulátoru.

Předávané parametry:

| Parametr    | Typ         | Povinný | Popis                               |
|:------------|:------------|:--------|:------------------------------------|
| requestId   | string-guid | ano     | Jednoznačný identifikátor requestu. |
| mode        | string      | ano     | Cílový mód zařízení.                |

Cílový mód zařízení může nabývat těchto hodnot:

| Hodnota            | Název                       |
|--------------------|-----------------------------|
| basic-regulator    | Základní regulace teploty.  |
| manual             | Manuální regulace.          |
| summer             | Letní režim.                |

Ukázka requestu:

```yaml
{
    "requestId": "b5e5a8e4-d09d-4d0f-8878-5ab24c2647fc",
    "mode": "basic-regulator"
}
```

### GET api/temperature-regulator/{DeviceId}/mode

Zjištění módu teplotního regulátoru.

Ukázka response:

```yaml
{
    "mode": "basic-regulator"
}
```

### POST api/temperature-regulator/{DeviceId}/temperature

Nastavení cílové teploty pro regulaci.
Podporováno pouze, pokud mód regulátoru je `basic-regulator`.

Předávané parametry:

| Parametr           | Typ         | Povinný | Popis                               |
|:-------------------|:------------|:--------|:------------------------------------|
| requestId          | string-guid | ano     | Jednoznačný identifikátor requestu. |
| target-temperature | float       | ano     | Cílová teplota.                     |

Ukázka requestu:

```yaml
{
    "requestId": "b5e5a8e4-d09d-4d0f-8878-5ab24c2647fc",
    "target-temperature": 21.5
}
```

### GET api/temperature-regulator/{DeviceId}/temperature

Zjištění cílové teploty pro regulaci.
Podporováno pouze, pokud mód regulátoru je `basic-regulator`.

Ukázka response:

```yaml
{
    "target-temperature": 21.5
}
```

### POST api/temperature-regulator/{DeviceId}/position

Nastavení cílové polohy termohlavic.
Podporováno pouze, pokud mód regulátoru je `manual`.

Předávané parametry:

| Parametr           | Typ         | Povinný | Popis                               |
|:-------------------|:------------|:--------|:------------------------------------|
| requestId          | string-guid | ano     | Jednoznačný identifikátor requestu. |
| target-temperature | float       | ano     | Cílová teplota.                     |

Ukázka requestu:

```yaml
{
    "requestId": "b5e5a8e4-d09d-4d0f-8878-5ab24c2647fc",
    "position": 50
}
```

### GET api/temperature-regulator/{DeviceId}/position

Zjištění cílové polohy termohlavic.
Podporováno pouze, pokud mód regulátoru je `manual`.

Ukázka response:

```yaml
{
    "position": 50
}
```

### POST api/temperature-regulator/{DeviceId}/factory-reset

Zruší zařízení, což umožní znovu spárovat jednotlivé fyzické komponenty. Dojde k výmazu provozních dat.