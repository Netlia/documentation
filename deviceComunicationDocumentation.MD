# Netlia dokumentace zařízení

Netlia nabízí velké množství zařízení které komunikují přes společný protokol. Tento dokument slouží k
popisu chování jednotlivých zařízení a k popisu společného komunikačního protokolu.

## Základní vlastnosti zařízení

 - Zařízení komunikují se serverem pomocí Lora (TODO odkaz) nebo NbIot (TODO odkaz) sítě. Výběr sítě záleží na typu zakoupeného zařízení.
 - Zařízení komunikují pomocí hexadecimálního payloadu. Příklad 020902050098434F0000020000.
 - Zařízení umožňují odesílat (TODO odkaz) i přijmat zprávy (TODO odkaz) ze serveru.
 - Zařízení obsahuje jeden až N senzorů které sledují okolí. Na základě informací z těchto senzorů se zařízení rozhoduje jakou zprávu bude odesílat na server. 

## Obecné chování zařízení

Zařízení odesílají tři hlavní druhy zpráv - event, restart a measure. Event zprávy jsou odeslány
vždy když zařízení zaznamená nějakou událost. Například když Move zařízení zaznamená pohyb 
nebo když water detection senzor přijde do kontaktu s vodou. Restart zprávy informují server že došlo k 
restartování zařízení stisknutím tlačítka nebo vložením baterky (TODO Tohle asi ani neni pravda, pri vlozeni baterky se posle pouze transport). Measure zprávy jsou odesílány v určitém 
časovém intervalu a odesílají naměřená data. Příkladem může být naměřená teplota z teplotního senzoru.
Podrobný popis těchto a dalších zpráv můžete nájít zde (TODO odkaz).

Některé zprávy v závisloti na nastavení a typu zařízení vyžadují potvrzení ze serveru. V těchto
případech zařízení čeká na odpověď ze serveru a pokud ji neobdrží tak zprávu odešle znovu.
V době kdy se zařízení snaží opětovně odeslat zprávu nepřijmá žádné nové informace ze senzorů.
Pokud tedy nastane událost zatím co zařízení čeká na potvrzení ze serveru tak je tato událost ignorována.

Všechny zprávy které nejsou potvrzovány mouhou být ztraceny jelikož se je zařízení nesnaží odeslat znovu.
Server se o těchto zprávách nikdy nedozví. Jedinou informaci kterou může server získat je že se nějaká zpráva ztratila
jelikož neodpovídá počítadlo zpráv.

Zařízení dokáží přijmat zprávy ze serveru. Tyto zprávy můžeme rozdělit na potvrzení, nastavení konfigurace a příkazy.
Potvrzení slouží k potvrzování zpráv ze zařízení. Nastavení konfigurace umožňuje nastavit konfiguraci zařízení a
příkazy umožňují určitým způsobem manipulovat se zařízením. Například přikázat chytrému světlu aby se vyplo nebo zaplo. 
Více o těchto zprávách najdete zde (TODO odkaz).

Příjmání zpráv probíhá vždy poté co zařízení odešle zprávu na server. Server musí zprávu odeslat v určitém
časovém okně jinak ji zařízení nepřijme. Toto chování se také liší podle toho jestli komunikace probíhá 
přes NbIot nebo Lora. Více podrobností zde (TODO odkaz).

## Restart zařízení

Zařízení umožňuje dva druhy restartu - stisknutí tlačítka a vložení baterie. Při vložení baterie se zařízení
přepne do transportního režimu který šetří baterku a slouží převážně k přepravě zařízení. Pro přepnutí do běžného
režimu je potřeba stisknout restart tlačítko.

Při vložení baterie zařízení také restartuje některá nastavení která mohla být přenastavena ze serveru. 
//TODO která?? Restart tlačítkem také může restartovat některá nastavení. //TODO která?? Více o 
nastavení a restartu zařízení můžete najít zde (TODO odkaz pravdepodobne na downlink).


## Stavový diagram zařízení

Následující diagram popisuje stavy do kterých se zařízení může dostat. Stavový diagram popisuje 
všechny stavy do kterých se zařízení teoreticky může dostat. Většina zařízení ale nepodporuje všechny
stavy a poporuje pouze podmnožinu tohoto úplného stavového automatu. Podrobnosti o jednotlivých 
zařízeních najdete zde (TODO).

Stavový automat neobsahuje restart přechod. Restart zařízení může nastat v jakémkoliv stavu.

Stavový automat také nezohledňuje přijmání zpráv ze serveru.

![State machine](/deviceComunicationDocumentation.png)




//TODO nestastne vyndani baterky
// Pri vyndani baterie server nemuze rozeznat zda na zařízení pouze došla baterka nebo zda chce uživatel restartovat
// senzor do tovarniho nastaveni.  Příklad kdy tohle nebude dobře fungovat - server pošle na zarizeni 
// zmenu konfigurace. Zmena zpusobi ze se zarizeni rozbije (spatny downlink nebo bug...). Uzivatel se tedy rozhodne ze
// vynda baterku a restartuje zarizeni do tovarniho nastaveni. Server ale musi znat konfiguraci zarizeni a hned po prijeti
// restart zpravy se bude snazit poslat stejnou konfiguraci na device. V tu chvili se zarizeni znovu rozbije. 
// Jedinym resenim je ze nejak restartujes server a pak az restartujes zarizeni. V tomhle intervalu kdy je 
// server nastaveny jinak ale nemusi prijmat zpravy.
// Dale je tam problem se zarizenima ktere jsou napajene ze zasuvky.

//TODO je potreba alert continue
// Pokud by se potvrzovali i alertEndy tak alert continue nepridava zadnou hodnotu.
// Pokud by se nepridali tak je ta pridana hodnota dost minimalni


// TODO mozna jinak u NBiot a Lora
// Vyzadujici potvrzeni
// Reset
// Test
// Alert start (dalsi se nepotvrzuji)
// Alive
// Kazda 6ta zprava se potvrzuje at je jakakoliv (primarne pro measure na teplomeru, u
// senzoru simple ani snad nemuze nastat)


//Nastavitelne downlinkem:
//Da se vypinat alert zprava potvrzeni, jestli potreba potvrzovat Xtou a globalni vypnuti.

//Da se zjistit podle flagu

//Vodomer ma vyple potvrzovani vseho

## Zprávy ze zařízení na server

Zprávy se odesílají v hexadecimálním tvaru a skládají se ze dvou částí - hlavičky a dat.
Hlavička obsahuje obecné informace a typ zprávy. Typ zprávy pak určuje 
jak bude vypadat datová část zprávy.

## Hlavičkla

Hlavička má 7 bytů a obsahuje obecné informace které jsou společné pro všechny typy zpráv.
Příklad: 0209020. Následující tabulka obsahuje zkrácený popis hlavičky. Celkový popis pro
jednotlivé byty následuje v dalších odstavcích.

| Pozice | Zkrácený popis                                                                                          |
|--------|---------------------------------------------------------------------------------------------------------|
| 0.byte | Počítadlo odslaných zpráv                                                                               |
| 1.byte | Počítadlo přijatých zpráv                                                                               |
| 2.byte | Stav baterie v zazení                                                                                   |
| 3.byte | Teplota procesoru                                                                                       |
| 4.byte | Kvalita připojení                                                                                       |
| 5.byte | 0 bit - Vyžádání potvrzení zprávy,<br/>1 bit - disconect<br/>Zbylé bity - počet pokusů o odeslání zprávy|
| 6.byte | Nepoužitý                                                                                               |
| 7.byte | Typ zprávy                                                                                              |

### 0.byte - počítadlo odeslaných zpráv

Počítadlo nabívající hodnot 0-255. Zařízení po každém odeslání zprávy zvýší počítadlo
o 1. Pokud počítadlo nabyde hodnoty 255 tak se při dalším odeslání zprávy nastaví na 0.

Počítadlo nezvyšuje svou hodnotu pokud se zařízení snaží znovu odeslat zprávu která
nebyla potvrzena.

Příklad:

| Číslo zprávy | Hodnota hlavičky                                                |
|--------------|-----------------------------------------------------------------|
| 1            | 00------------ (zbytek hlavičky je nahrazen - pro jednoduchost) |
| 2            | 01------------                                                  |
| 3            | 02------------                                                  |
| 255          | 254------------                                                 |
| 256          | 255------------                                                 |
| 257          | 00------------ (došlo k přetečení a counter se restartoval)     |

### 1.byte - počítadlo přijatých zpráv

Defautlně je nastaveno na 0. Po přijetí zprávy ze serveru se nastaví na posledí hodnotu 
přijatou ze serveru. Více o této hodnotě zde (TODO odkaz).


### 2.byte - napětí baterie v zařízení

Určuje hodnotu napětí baterie. Pokud je zařízení napájeno ze zásuvky tak obsahuje hodnotu napětí zdroje.

Napětí může nabívat hodnot 1.8V–4.35V. Hodnota 0 odpovídá napětí 1.8V a hodnota 255 odpobvídá napětí 4.35V.

Aktuální napětí můžeme vypočítat podle následujícího vzorce: *naměřená hodnota* / 100 + 1.8

// TODO co kdyz je napeti vyssi nebo nizzsi? nez 4.35 a 1.8.

### 3.byte - teplota procesoru

Určuje teplotu procesoru. Tento byte může nabývat hodnotu 0-160 a 255. Aktuální 
teplotu vypočítáme podle následujícího vzorce: "přijatá hodnota" - 40 = aktuální teplota.
Pokud je teplota vyšší než 120°C tak senzor pošle hodnotu 0xFF (255). Zařízení tedy 
dokáže měřit teploty -40°C až 120°C.

Senzor nikdy nepošle hodnotu 161 až 254. Jelikož by odpovídala hodnotě vyšší než 120°C.

// TODO co se stane když je teplota nižší než -40

### 4.byte - RSSI

Určuje RSSI (TODO odkaz na wiki) zařízení při odesílání zprávy.

### 5.byte - potvrzení, disconnect a počet pokusů o odeslání zprávy
Tento byte obsahuje tři informace. Následující tabulka vysvětluje složení bytu:

| Pozice  | Zkrácený popis                                      |
|---------|-----------------------------------------------------|
| 0.bit   | Žádost o potvrzení zprávy serverem                  |
| 1.bit   | Určuje zda zařízení bylo přepnuto do disconect módu |
| 2-7.bit | Počet pokusů o odeslání zprávy                      |

Pokud je 0.bit nastaven na 1 tak zařízení vyžaduje potvrzení zprávy ze serveru.
Zařízení čeká na potvrzení X minut (TODO jak dlouho??). Pokud potvrzení
nedorazí tak zprávu odešle znovu a zvýší *Počet pokusů o odeslání zprávy* o jedna.

Pokud ani podruhé nedorazí potvrzení tak zařízení opět čeká X minut (TODO jak dlouho?)
a poté se pokusí zprávu odeslat ještě jednou.

Při dalším pokusu se zařízení přepne do disconnect módu (1.bit se nastaví na 1) a
zprávy odesílá se stále větší prodlevou. 

Následující tabulka shrnuje předchozí informace a popisuje chování v disconnect módu:

| Pokus o odeslání     | Hodnota 5.byte | Popis                                                                                                                          |
|----------------------|----------------|--------------------------------------------------------------------------------------------------------------------------------|
| 1. pokus o odeslání  | 0000000**1**   | Zařízení odeslalo zprávu a vyžaduje potvrzení (tučně označný bit).                                                             |
| 2. pokus o odeslání  | **000001**01   | Zařízení čekalo Xmin (TODO zjistit) a nyní se snaží zprávu znovu odeslat. Počítadlo opakování (tučně) bylo zvýšeno o 1.        |
| 3. pokus o odeslání  | 00001001       | Zařízení čekalo Xmin (TODO zjistit) a nyní se snaží zprávu znovu odeslat.                                                      |
| 4. pokus o odeslání  | 000011**1**1   | Zařízení čekalo 15 min (TODO zjistit) a nyní se přeplo do disconnect módu (tučně) a pro další opakování bude čekat delší dobu. |
| 5. pokus o odeslání  | 00010011       | Zařízení čekalo 15 min                                                                                                         |
| 6. pokus o odeslání  | 00010111       | Zařízení čekalo 15 min                                                                                                         |
| 7. pokus o odeslání  | 00011011       | Zařízení čekalo 60 min                                                                                                         |
| 8. pokus o odeslání  | 00011111       | Zařízení čekalo 60 min                                                                                                         |
| 29. pokus o odeslání | 01110111       | Zařízení čekalo 60 min                                                                                                         |
| 30. pokus o odeslání | 01111011       | Zařízení čekalo do doby než mělo odeslat alive zprávu (TODO odkaz)<br/> poté namísto alive zprávy znovu odeslalo tuto zprávu   |
| 31. pokus o odeslání | 01111111       | Zařízení opět čeká do doby než se má odeslat alive zpráva                                                                      |

//TODO co kdyz ma zarizeni nastaveny alive interval kratsi nez 12hod?

Před každým dalším odesláním zařízení čeká dokud není čas odeslat alive zprávu (TODO odkaz). Zařízení se snaží stejnou zprávu odeslat stále dokola 
dokud není restartováno (vybití baterie nebo ruční reset).

Pokud nastane jiná událost zatím co se zařízení pokouší odeslat stávající zprávu tak je nová událost
ignorována.

### 6.byte - nepoužitý

### 7.byte - typ zprávy

Typ zprávy určuje co budou obsahovavt další byte v payloadu. Následující tabulka ukazuje jaké hodnoty může 
tento byte nabívat:


| Hodnota | Název               |
|---------|---------------------|
| 1       | DownlinkaAcknowlege |
| 2       | Restart             |
| 3       | Test                |
| 4       | Error               |
| 5       | Event               |
| 7       | Alive               |
| 8       | Transport           |
| 9       | Measure             |

DownlinkaAcknowlege, Restart, Test, Error, Alive, Transport mají stejný obsah pro všechna zařízení.
Tyto zprávy jsou popsány v dalších odstavcích o datech.

Event a Measure se liší podle typu zařízení a jsou popsány dále v tomto dokumentu u popisu jednotlivých
zařízení. (TODO odkaz)


## Data

Následující odstavce popisují datovou část zprvávy. Všechny tyto zprávy začínají společnou
hlavičkou která je popsaná výše (TODO odkaz). Obsah hlavičky (pro připomenutí):

| Pozice | Zkrácený popis                                                                                          |
|--------|---------------------------------------------------------------------------------------------------------|
| 0.byte | Počítadlo odslaných zpráv                                                                               |
| 1.byte | Počítadlo přijatých zpráv                                                                               |
| 2.byte | Stav baterie v zazení                                                                                   |
| 3.byte | Teplota procesoru                                                                                       |
| 4.byte | Kvalita připojení                                                                                       |
| 5.byte | 0 bit - Vyžádání potvrzení zprávy,<br/>1 bit - disconect<br/>Zbylé bity - počet pokusů o odeslání zprávy|
| 6.byte | Nepoužitý                                                                                               |
| 7.byte | Typ zprávy                                                                                              |
// TODO odkazy do tabulky

### Downlink acknowlage

Downlink acknowlage se odesílá vždy po přijetí příkazu ze serveru jako potvrzení.
V hlavičce je označen hodnotou 1 v 7.bytu (Typ zprávy).

Obsah:

| Byte   | Význam                       |
|--------|------------------------------|
| 0.byte | Nepoužitý vždy obsahuje 0xFF |
| 1.byte | Nepoužitý vždy obsahuje 0x00 |

### Restart

Posílá se vždy po restartu zařízení tlačítkem nebo při přijetí reset zprávy (TODO nazev a odkaz)
ze serveru.
V hlavičce je označen hodnotou 0x02 v 7.bytu (Typ zprávy).

Obsah:

| Byte    | Význam                       |
|---------|------------------------------|
| 0.byte  | Nepoužitý vždy obsahuje 0xFF |
| 1.byte  | Nepoužitý vždy obsahuje 0x12 |
| 2.byte  | Typ zařízení                 |
| 3.byte  | Mód zařízení                 |
| 4.byte  | major verze payloadu         |
| 5.byte  | minor verze payloadu         |
| 6.byte  | major verze firmwaru         |
| 7.byte  | minor verze firmwaru         |
| 8.byte  | patch verze firmwaru         |
| 9.byte  | major verze radia            |
| 10.byte | minor verze radia            |
| 11.byte | patch verze radia            |
| 12.byte | počet restartů               |
| 13.byte | kód restartu                 |

Typ a mód zařízení identifikuje zařízení a jeho interní nastavení. Více o typu a módu se dozvíte 
zde // TODO odkaz.

Major, minor a patch verze vždy udává verzi podle definice semantického verzovani (https://semver.org/).
Tyto údaje slouží převážně k diagnostice zařízení.


Počet restartů udává kolikrát byl senzor restartován od vložení baterky (TODO ověřit).

Kód restartu udává důvod proč resatart nastal. Byte může nabívat následujících hodnot:

| Hodnota | Význam                                      |
|---------|---------------------------------------------|
| 0       | Restart způsoben chybou Freez_RST           | //TODO co je freez rst? 
| 1       | Restart vyvolán přijmutím zprávy ze serveru | //TODO jakou zpravou??
| 2       | Restart způsoben stisknutím tlačítka        |
| 3       | Restart způsoben chybou HARDFAULT           | // TODO co to je??
| 4-7     | Nepoužito                                   |

### Test

Zařízení po odeslání restartu odesílá test zprávu. Test zpráva slouží k přijetí zpráv ze serveru
po restartoví zařízení a také pro kontrolu síly signálu.  Obvykle se v této zprávě odesílají nastavení zařízení 
jelikož restart může způsobit nastavení některých nastavení na defaultní hodnotu //TODO lepsi. 
Nastavení nemůže být přijato po odeslání restart zprávy kvůli omezení Lora protokolu. Více informací Zde. (TODO odkaz)

Zařízení podporují odeslat nula až X testovacích zpráv podle nastavení. 
Defaultní nastavení pro většinu zařízení je jedna testovací zpráva po restartu.

Zařízení vždy čeká určiý čas po odeslání restart zprávy a až poté odešle test zprávu.
Doba čekání se dá nastavit zprávou ze serveru. Defaultně zařízení čeká 2 minuty.

Více informací o nastavení intervalů a počtů testovacích zpráv můžete najít zde. //TODO odkaz

Obsah:

| Byte   | Význam                       |
|--------|------------------------------|
| 0.byte | Nepoužitý vždy obsahuje 0xFF |
| 1.byte | Nepoužitý vždy obsahuje 0x00 |

### Error

Error zpráva je ze zařízení odeslána když nastane hardwarová chyba.

Obsah:

| Byte            | Význam                               |
|-----------------|--------------------------------------|
| 0.byte          | Typ chyby                            |
| 1.byte          | Nepoužitý vždy obsahuje 0x04         |
| 2.byte - 5.byte | Obsahuje hodnotu z chybového registru | 

Errory se dělí na dva druhy chyb - fatální a normální. Normální chyby mají hodnotu 0.bytu 
nastavenou na 0 a fatální mají hodnotu 1.

Pokud v zařízení nastane fatální chyba tak se zařízení restartuje. Poté pošle restart zprávu
a následně odešle errorovou zprávu. 
// TODO ma prednost errorova zprava nebo test zprava?
// TODO po jak dlouhe dobe je Errorová zpráva je odeslná?


Pokud nastane normální chyba tak zařízení pošle errorovou zprávu a nerestartuje se. Pokud 
se chyba opakuje tak je odeslána další errorová zpráva. Pokud je normální errorová
zpráva detekována a odeslána 5* tak se vyvolá fatální chyba. Zařízení se pak restartuje
a odešle errorovou zprávu stejně jako u každé jiné fatální chyby.

// TODO co kdyz se seenzor snaží odeslat zprávu a nemá připojení a v tu chvíli nastane HW chyba

// TODO tohle asi neni potreba popisovat

Chybový registr obsahuje informaci o tom jaká chyba nastala. Následující tabulka popisuje
hodnoty a jejich význam:

| Hodnota v binární podobě            | Význam                                     | Závažnost chyby |
|-------------------------------------|--------------------------------------------|-----------------|
| 00000000 00000000 00000000 00000000 | Chyba CLEAR_TRY - I2C                      | Normální        |
| 00000000 00000000 00000000 00000001 | Chyba CLEAR_TRY - UART                     | Normální        |
| 00000000 00000000 00000000 00000010 | Chyba CLEAR_TRY - EEPROM                   | Normální        |
| 00000000 00000000 00000000 00000100 | Chyba CLEAR_TRY - senzor                   | Normální        |
| 00000000 00000000 00000000 00001000 | Chyba CLEAR_TRY - akční člen               | Normální        |
| 00000000 00000000 00000001 00000000 | Chyba Freeze_RST - radio                   | Fatální         |
| 00000000 00000000 00000010 00000000 | Chyba Freeze_RST - vložení ne plné baterie | Fatální                |
| 00000000 00000000 00000100 00000000 | Chyba Freeze_RST - neočekávaný stav STM    | Fatální                |
| 00000000 00000000 00001000 00000000 | Chyba Freeze_RST - chyba stavu STM_INIT    | Fatální                |
| 00000000 00000000 00010000 00000000 | Normální chyba nastala 5*                  | Fatální                |
| 00000000 00000001 00000000 00000000 | Chyba Change_State - spojení (Sítě)        | Fatální                |
| 00000000 00000010 00000000 00000000 | Chyba Change_State - nízké napětí baterie  | Fatální                |
| 00000000 00000100 00000000 00000000 | Chyba Change_State - event bufferu         | Fatální                |
// TODO lepsi vyznam Tohle je pouze vykopírováno z dokumentace. Dokumentace také vysvetluje Change_State atd.

### Alive

Alive zpráva slouží jako notifikace že zařízení je v pořádku a stále vysílá. Alive zpráva
se posílá pouze pokud zařízení dlouhou dobu nekomunikovalo. Defaultně je alive zpráva odeslána pokud
zařízení nekomunikovalo se serverem 12 hodin. Interval se dá nastavit pomocí příkazu ze serveru (TODO nazev a odkaz).


Obsah:

| Byte   | Význam                       |
|--------|------------------------------|
| 0.byte | Nepoužitý vždy obsahuje 0xFF |
| 1.byte | Nepoužitý vždy obsahuje 0x00 |

### Transport

Transport zpráva je odeslána při přepnutí zařízení do transportního režimu. Transportní režim je 
speciální stav zařízení ve kterém zařízení nekomunikuje a šetří baterii. Obvykle se používá pro 
přepravu nebo skladování zařízení.

Zařízení se do transportního režimu dostane vložením baterie. Ukončen transportního režimu nastane
při stisknutí restart tlačítka. // TODO mam pocit ze pavel rikal ze oziveni pujde i ze serveru

## Zařízení

### Water

### Move

## Downlink

## Komunikace mezi serverem a zařízením

### Lora komunikace

### Nb Iot komunikace

 

## Jen poznamky
Test a Restart
Kdy se odesila restart a test - restart postisknuti reset tlacitka, da se vyvolat i downlinkem
Test se posila pouze po restartu.  Za minutu po restartu. U vsech senzoru.

Join - u lory. Pro navazani konektivyty probiha join (nejaka vymena klicu). Po provedeni 
joinu se smaze fronta downlinku v lore.

Join je vyvolan pokud je senzor restartovan nebo pokud se mu 2* nepodari dorucit zpravu (TODO potreba overit).

U NB iot zadny join neni.

Testovaci zprava je tam kvui downlinku - restart smaze frontu protoze udela join a my na serveru vyhodnotime
ze se udelal join a pote tam vlozime znovu downlink a pri primu testu se nam potvrdi

Test je nastavitelny downlinkem (parametr ktery rika jestli se ma poslat 1 nebo 2 3...) TODO overi u davida

Do transportniho rezimu se senzor prepne pri vlozeni baterky a odesle transport zpravu

Alive se posila jednou za 12hodin (default a u tlacitka je to 6) od poledni zpravy
Interval se da nastavit downlinkem


Event_Type Message - neni pouzito // TODO overit



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
Počet testů se dá nastavit
// TODO da se opravdu počet testů nastavit??
// Pokud nastavím počet testů na 3 a restartuju senzor
// tak se znovu nastaví hodnota na default a stejně se testy neprovedou
endNote

ErrorCheckMode --> ActiveMode : Errory odeslány nebo neni potřeba odeslat
ErrorCheckMode --> ErrorCheckMode : Error zpráva odeslána
note left of ErrorCheckMode
Pokud bylo zařízení restartováno
kvůli fatálnímu erroru tak je potřeba
odeslat error zprávu s podrobnostmi o chybě
//TODO overit jestli error check probíhá před
nebo po odeslani test zpravy
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
