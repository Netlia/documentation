## Jen poznamky

### Dalsi poznamky

@startuml
[*] --> TransportMode : Vložení baterky
TransportMode --> SendRestartMessage : Restart tlačítko zmáčknuto

SendRestartMessage --> WaitForRestartConfirmation : Čekání na potvrzení restartu
WaitForRestartConfirmation --> SendRestartMessage : Potvrzení nedorazilo
WaitForRestartConfirmation --> SendTestMessage : Potvrzení dorazilo

SendTestMessage --> WaitForTestConfirmation : Čekání na potvrzení
WaitForTestConfirmation --> SendTestMessage : Potvrzení nedorazilo
note right of SendTestMessage
Test zpráva slouží primárně k
přijetí aktuálního nastavení
ze serveru. Nastavení je
vždy nastaveno na default po restartu
end note

WaitForTestConfirmation --> ErrorCheckMode : Potvrzení dorazilo
WaitForTestConfirmation --> SendTestMessage : Potvrzení dorazilo a je potřeba odeslat další test
Note on link
Počet test zpráv se liší podle typu zařízení
endNote

ErrorCheckMode --> ActiveMode : Errory odeslány nebo neni potřeba odeslat
ErrorCheckMode --> ErrorCheckMode : Error zpráva odeslána
note left of ErrorCheckMode
Pokud bylo zařízení restartováno
kvůli fatálnímu erroru tak je potřeba
odeslat error zprávu s podrobnostmi o chybě
end note

ActiveMode --> EventStartSent : Pošli event start

WaitForConfirmation --> ActiveMode : Potvrzení přijato
WaitForConfirmation --> WaitForConfirmation : Timeout, pošli zprávu znovu

ActiveMode --> WaitForConfirmation : Pošli alive zprávu
note left of ActiveMode
Zařízení posílá - error, measure, event, alive
error - HW chyba na zařízení
measure - hodnota naměřena (např. teplota, vlhkost)
event - nastala událost (např. překročení limitu, zaznamenání pohybu)
alive - kontrolní zpráva
end note

ActiveMode --> ActiveMode : Pošli measure/error/alive zprávu
ActiveMode --> WaitForConfirmation : Pošli measure/error zprávu s potvrzením
ActiveMode --> EventContinueMode : Pošli event start bez potvrzení

EventStartSent --> WaitForEventConfirmation : Čekej na potvrzení
WaitForEventConfirmation --> EventStartSent : Potvrzení nedorazilo
WaitForEventConfirmation --> EventContinueMode : Potvrzení přijato

EventContinueMode --> WaitForConfirmationForContinue : Pošli alive zprávu
note left of EventContinueMode
EventContinueMode je identicky s ActiveModem
Jediny rozdil je že když nastane event tak
se odesílá event continue namísto event start
end note

EventContinueMode --> EventContinueMode : Pošli measure/error/eventContinue/alive zprávu
EventContinueMode --> WaitForConfirmationForContinue : Pošli measure/error/eventContinue zprávu s potvrzením
EventContinueMode --> ActiveMode : Událost skončila
note on link
Konec události nastane pokud zařízení
nějakou dobu nezaznamená událost
end note

WaitForConfirmationForContinue --> EventContinueMode : Potvrzení přijato
WaitForConfirmationForContinue --> WaitForConfirmationForContinue : Timeout, pošli zprávu znovu
@enduml

title Nb Iot communication

Device->Server: Send message with confirmation request
Device<-Server: Downlink
Device->Server: Confirmation
Server->Server: Mark downlink as sent

title NBIot retry

Device->Server: Send message with confirmation request
Device<-Server: Downlink
Device-->Server: Confirmation (delivery failed)
Device->Server: Send message with confirmation request
Device<-Server: Downlink
Device->Server: Confirmation
Server->Server: Mark downlink as sent

title Lora communication

Server->LoraServer: Send downlink message
Device->LoraServer: Send uplink message
LoraServer->Device: Send downlink
LoraServer->Server: Send uplink
Device->LoraServer: Send downlink confirmation
LoraServer->Server: Send downlink confirmation

title Lora "retry"

Server->LoraServer: Send downlink message
Device->LoraServer: Send uplink message
LoraServer->Device: Send downlink
LoraServer->Server: Send uplink
Server->LoraServer: Send Downlink again because it was not confirmed
Device->LoraServer: Send downlink confirmation
LoraServer->Device: Send second downlink
LoraServer->Server: Send downlink confirmation
Server->Server:Mark downlink ass sucessfuly sent
Device->LoraServer:Send second downlink confirmation
LoraServer->Server:Send second downlink confirmation
Server->Server: Ignore confirmation because downlink was already confirmed

title Lora "retry" join

Server->LoraServer: Send downlink message
Device->LoraServer: Send Join which deletes all messages
LoraServer->LoraServer: All waiting downlinks deleted
Device->LoraServer: Send uplink
LoraServer->Server: Send uplink

Server->LoraServer: Send downlink again because it was not confirmed
Device->LoraServer: Send uplink
LoraServer->Device: Send second downlink
LoraServer->Server: Send uplink
Server->LoraServer: Send downlink again because it was not confirmed
Device->LoraServer:Send second downlink confirmation
LoraServer->Device:Send third downlink
LoraServer->Server:Send second downlink confirmation
Server->Server:Mark downlink ass sucessfuly sent

Device->LoraServer:Send third downlink confirmation
LoraServer->Server:Send third downlink confirmation
Server->Server: Ignore confirmation because downlink was already confirmed

title Lora "Message Marker"

Server->LoraServer: Send downlink message
Server->Server: Mark downlink as sent
Device->LoraServer: Send uplink message
LoraServer->Device: Send downlink
LoraServer->Server: Send uplink
Server->Server:Mark downlink as toBeConfirmedInNextMessage
Device->LoraServer: Send downlink confirmation
LoraServer->Server: Send downlink confirmation
Server->Server:Mark downlink as sucessfuly sent

title Lora "Message Marker" join

Server->LoraServer: Send downlink message
Server->Server: Mark downlink as sent
Device->LoraServer: Send Join which deletes all messages
LoraServer->LoraServer: All waiting downlinks deleted
Device->LoraServer: Send uplink
LoraServer->Server: Send uplink
Server->Server:Mark downlink as toBeConfirmedInNextMessage
Device->LoraServer: Send uplink
LoraServer->Server: Send uplink
Server->Server:Set downlink to initial state

Server->LoraServer: Send downlink message again
Server->Server: Mark downlink as sent
Device->LoraServer: Send uplink message
LoraServer->Device: Send downlink
LoraServer->Server: Send uplink
Server->Server:Mark downlink as toBeConfirmedInNextMessage
Device->LoraServer: Send downlink confirmation
LoraServer->Server: Send downlink confirmation
Server->Server:Mark downlink as sucessfuly sent

title Lora "Timeout" (current implementation)

Server->LoraServer: Send downlink message
Device->LoraServer: Send uplink message
LoraServer->Device: Send downlink
LoraServer->Server: Send uplink
Server->Server: Start timer which resends downlink on timeout

Device->LoraServer: Send downlink confirmation
LoraServer->Server: Send downlink confirmation
Server->Server:Stop timer

title Lora "Timeout" join

Server->LoraServer: Send downlink message
Device->LoraServer: Send Join which deletes all messages
LoraServer->LoraServer: All waiting downlinks deleted
Device->LoraServer: Send uplink
LoraServer->Server: Send uplink
Server->Server: Start timer which resends downlink on timeout
Server->Server: Timer timeout
Server->LoraServer: Send downlink message
Device->LoraServer: Send uplink message
LoraServer->Device: Send downlink
LoraServer->Server: Send uplink
Server->Server: Start timer which resends downlink on timeout

Device->LoraServer: Send downlink confirmation
LoraServer->Server: Send downlink confirmation
Server->Server:Stop timer
