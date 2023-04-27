# API
Bude se jednat o REST API, bude k dispozici Swagger.

# Zabezpečení
Komunikace bude zabezpečena bearer tokenem v hlavičce, informace budou doplněny.

Většina endpointů přijímajících data v těle requestu vyžaduje položku `requestId`. Jedná se o jednoznačný identifikátor requestu, který generuje partner (jedná se o UUID) a slouží pro zajištění idempotence, čímž je docíleno toho, že akce je provedena jen jednou pokud dojde o opakovanému zavolání v určitém časovém intervalu. Při opakovaném požadavku se stejným `requestId` je vrácen HTTP status `200 OK`.

# Chybové odpovědi
Chyby 5xx jsou způsobené chybou serveru a neměli by nikdy nastat. Pokud server vratí tuto chybu, měl by být informován zástupce Netlia.

Chyby 4xx jsou vráceny, pokud klient provedl neplatný/nevalidní request. Vysvětlení stavových kódů je možné najít v [HTTP dokumentaci](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status#client_error_responses).

V některých případech 4xx error obsahuje body s dalšími informacemi o chybě. Body má následující formát:

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

### PUT api/temperature-regulator/{DeviceId}/mode

Změna módu teplotního regulátoru.

Předávané parametry:

| Parametr    | Typ         | Povinný | Popis                               |
|:------------|:------------|:--------|:------------------------------------|
| requestId   | string      | ano     | Jednoznačný identifikátor requestu. |
| mode        | string      | ano     | Cílový mód zařízení.                |

Cílový mód zařízení může nabývat těchto hodnot:

| Hodnota            | Název                       |
|--------------------|-----------------------------|
| basic              | Základní regulace teploty.  |
| summer             | Letní režim.                |

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

### GET api/temperature-regulator/{DeviceId}/mode

Zjištění módu teplotního regulátoru.

Ukázka response:

```yaml
{
    "mode": "basic"
}
```

### PUT api/temperature-regulator/{DeviceId}/temperature

Nastavení cílové teploty pro regulaci.
Podporováno pouze, pokud mód regulátoru je `basic`.

Předávané parametry:

| Parametr           | Typ         | Povinný | Popis                               |
|:-------------------|:------------|:--------|:------------------------------------|
| requestId          | string      | ano     | Jednoznačný identifikátor requestu. |
| targetTemperature  | float       | ano     | Cílová teplota.                     |

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

### GET api/temperature-regulator/{DeviceId}/temperature

Zjištění cílové teploty pro regulaci.
Podporováno pouze, pokud mód regulátoru je `basic-regulator`.

Ukázka response:

```yaml
{
    "targetTemperature": 21.5
}
```

<!--
### PUT api/temperature-regulator/{DeviceId}/factory-reset

Zruší zařízení, což umožní znovu spárovat jednotlivé fyzické komponenty. Dojde k výmazu provozních dat.

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

### PUT api/temperature-regulator/

Vytvoří požadavek na zavedení nového zařízení do systému. Po úspěšném vytvoření zařízení je partner informován eventem `device-created` - viz. dokumentace k předávání událostí.

Předávané parametry:

| Parametr           | Typ         | Povinný | Popis                                                              |
|:-------------------|:------------|:--------|:-------------------------------------------------------------------|
| requestId          | string      | ano     | Jednoznačný identifikátor requestu.                                |
| deviceType         | string      | ano     | Typ zařízení.                                                      |
| physicalDeviceIds  | string[]    | ano     | Fyzická zařízení / komponenty ze kterých je zařízení složeno.      |
| customData         | object      | ne      | Objekt s informacemi specifickými dle partnera.                    |

Ukázka requestu:

```yaml
{
    "requestId": "b5e5a8e4-d09d-4d0f-8878-5ab24c2647fc",
    "deviceType": "temperature-regulator",
    "physicalDeviceIds": ["abc123", "abc456", "abc789"],
    "customData":
        {
            "sample-key-1": "sample-value-1",
            "sample-key-2": 123 
        }
}
```

Ukázka response: 

```yaml
{
    "deviceId": "7a3945e9-89c6-4464-9feb-6642c97035b2",
    "physicalDeviceIds": ["abc123", "abc456", "abc789"]
}
```
-->