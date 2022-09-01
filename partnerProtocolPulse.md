# Specifikace předávání událostí pro zařízení Pulse

Dokument popisuje způsob předávání událostí ze systému NETLIA do aplikace partnera.

**Způsob komunikace je mezi NETLIA a partnerem dohodnut a na straně NETLIA nastaven.**

# Způsoby předávání dat

Data jsou předávána komunikačním protokolem ve formátu JSON a popisují události, které vznikají na zařízení Pulse.

## Předávání událostí přes HTTP callback

Partner může specifikovat URL endpointů na které jsou události zasílány formou HTTP(S) POST requestů. Requesty mají kódování UTF-8 a Content-Type “application/json”. Systém se snaží předat události v režimu `at-least-once`, může tedy nastat situace že je událost doručena víckrát. Tuto situaci lze ošetři s využitím položky `EventId`, která obsahuje identifikátor události.

### URL HTTP requestu

Do URL je možné vložit zástupné parametry, které budou nahrazeny odpovídající hodnotou.

| Parametr          | Popis                         |
| :-----------------|:------------------------------|
| ProtocolVersion   | verze komunikačního protokolu |
| DeviceId          | identifikátor senzoru         |
| MessageType       | typ zprávy                    |
| EventType         | typ události                  |

Příklad URL (doporučené nastavení):

`https://nejakaadresa.cz/event/v{ProtocolVersion}/{DeviceId}/{MessageType}/{EventType}/`

Pro uvedenou URL budou volány např. tyto requesty:

https://nejakaadresa.cz/event/v1/abc123/pulse/measured/

https://nejakaadresa.cz/event/v1/abc123/pulse/restart/

### Hlavičky HTTP requestu
Partner může specifikovat další konfiguraci přidáním HTTP hlavičy (klíč a hodnota). Tím lze např. vyřešit autorizaci.

### Odpověď na HTTP request

Systém očekává v odpovědi HTTP status 200-299, kterým partner potvrdí přijetí události. Jiná odpověď je vyhodnocena jako nedoručení.

### Chování v případě nedoručení události
 V případě, že se nepodaří událost předat, systém pokus 10x opakuje s 5s prodlevami. Následně je událost zahozena.

# Zařízení Pulse

Měří počty pulzů z připojeného externího zařízení.

Zařízení sčítá pulzy během stanoveného intervalu a odesílá je v události typu `measured`.

Všechny zprávy mají společnou část která obsahuje: 
* `MessageType` - určuje typ zprávy. Pro zařízení pulse je hodnota vždy "pulse".
* `ProtocolVersion` - určuje verzi protokolu. Aktuální verze protokolu je 1.
* `DeviceId` - řetězec který určuje unikátní identifikátor zařízení.
* `EventTime` - čas vzniku zprávy na zařízení. Odesílá se v UTC ve formátu ISO 8601.
* `EventId` - řetězec který unikátně identifikuje zprávu.
* `EventType` - určuje typ události která nastala. Následující tabulka ukazuje jakých hodnot může toto pole nabívat. Podrobný popis je pak v následujících odstavcích


| EventType                                   | Popis |
|:--------------------------------------------|:------|
| [restart](#eventtype-restart)               | Restart zařízení. |
| [alive](#eventtype-alive)                   | Nastává v pravidelném intervalu, potvrzuje funkčnost zařízení. |
| [transport](#eventtype-transport)           | Přechod do transportního režimu - neaktivní stav s minimální spotřebou. |
| [measured](#eventtype-measured)             | Naměřené veličiny. |

## EventType restart
Nastává při restartu zařízení. K restartu může dojít stiskem restartovacího tlačítka na zařízení, při hardwarové chybě, nebo vynucení restartu příkazem ze serveru.

Ukázka zaslané události:
```yaml
{
    "ProtocolVersion": 1,
    "DeviceId": "abc123",
    "MessageType": "pulse",
    "EventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "EventTime": "2021-05-03T14:25:31.8437511Z",
    "EventType": "restart"
}
```

## EventType alive
Nastává v pravidelném intervalu, potvrzuje funkčnost zařízení.

Ukázka zaslané události:
```yaml
{
    "ProtocolVersion": 1,
    "DeviceId": "abc123",
    "MessageType": "pulse",
    "EventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "EventTime": "2021-05-03T14:25:31.8437511Z",
    "EventType": "alive"
}
```

## EventType transport
Nastává při přechodu zařízení do transportního režimu, ke kterému dojde po vložení nové baterie.

Zařízení v transportním režimu má velmi nízkou spotřebu a neposílá žádné zprávy. Pro probuzení čidla z transportního režimu je třeba stisknout RESET tlačítko umístěné na zařízení.

Ukázka zaslané události:
```yaml
{
    "ProtocolVersion": 1,
    "DeviceId": "abc123",
    "MessageType": "pulse",
    "EventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "EventTime": "2021-05-03T14:25:31.8437511Z",
    "EventType": "transport"
}
```

## EventType measured
Nastává při odeslání naměřené hodnoty.

Dodatečné předávané parametry:
| Parametr          | Typ         | Povinný | Popis
| :-----------------|:------------|:--------|:-----
| MeasuredValues    | PulseValue  | ano     | naměřená hodnota

Objekt PulseValue:

| Parametr          | Typ     | Povinný | Popis
| :-----------------|:--------|:--------|:-----
| Time              | string  | ano     | čas měření v UTC ve formátu ISO 8601
| Value             | integer | ano     | počet pulzů

Ukázka zaslané události:
```yaml
{
    "ProtocolVersion": 1,
    "DeviceId": "abc123",
    "MessageType": "pulse",
    "EventId": "c4056fc4-d433-4d2c-bb7f-23a691fd3dac",
    "EventTime": "2021-05-03T14:25:31.8437511Z",
    "EventType": "measured",
    "MeasuredValues": [
        {
        "time": "2022-08-31T09:28:21.626Z",
        "value": 10
        },
        {
        "time": "2022-08-31T09:38:21.626Z",
        "value": 20
        }
  ]
}
```