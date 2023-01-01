## Jen poznamky

### Dalsi poznamky

@startuml
[*] --> TransportMode : Battery inserted
TransportMode --> SendRestartMessage : Restart button pushed

SendRestartMessage --> WaitForRestartConfirmation : Waiting for restart confirmation
WaitForRestartConfirmation --> SendRestartMessage : Confirmation not received
WaitForRestartConfirmation --> SendTestMessage : Confirmation not received

SendTestMessage --> WaitForTestConfirmation : Waiting for confirmation
WaitForTestConfirmation --> SendTestMessage : Confirmation not received
note right of SendTestMessage
Test message primarily serves 
for downloading device settings 
from server. Settings are always 
restored to default after restart.
end note

WaitForTestConfirmation --> ErrorCheckMode : Confirmation received
WaitForTestConfirmation --> SendTestMessage : Confirmation received and another test message must be sent
Note on link
Number of test messages differs by device type
endNote

ErrorCheckMode --> ActiveMode : Errors have been sent or don't need to be sent
ErrorCheckMode --> ErrorCheckMode : Error message has been sent
note left of ErrorCheckMode
If the device has been restarted
due to a fatal error error message
must be sent with error details
end note

ActiveMode --> EventStartSent : Send Event start

WaitForConfirmation --> ActiveMode : Confirmation received
WaitForConfirmation --> WaitForConfirmation : Timeout, resend the message

ActiveMode --> WaitForConfirmation : Send alive message
note left of ActiveMode
Device sends - error, measure, event, alive
error - HW error in the device
measure - measured values (e.g. temperature, humidity)
event - event registered (e.g. treshold crossed, motion detected)
alive - control message
end note

ActiveMode --> ActiveMode : Send measure/error/alive message
ActiveMode --> WaitForConfirmation : Send measure/error message with confirmation
ActiveMode --> EventContinueMode : Send event start without confirmation

EventStartSent --> WaitForEventConfirmation : Wait for confirmation
WaitForEventConfirmation --> EventStartSent : Confirmation not received
WaitForEventConfirmation --> EventContinueMode : Confirmation received

EventContinueMode --> WaitForConfirmationForContinue : Send alive message
note left of EventContinueMode
EventContinueMode is identical to ActiveModem
The only difference is that if an event occurs
when in EventContinueMode, event continue is 
sent instead of event start.

EventContinueMode --> EventContinueMode : Send measure/error/eventContinue/alive message
EventContinueMode --> WaitForConfirmationForContinue : Send measure/error/eventContinue message with confirmation
EventContinueMode --> ActiveMode : Event end
note on link
Event ends when no input events 
are detected by the device for 
some time.
end note

WaitForConfirmationForContinue --> EventContinueMode : Confirmation received
WaitForConfirmationForContinue --> WaitForConfirmationForContinue : Timeout, resend the message
@enduml




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
