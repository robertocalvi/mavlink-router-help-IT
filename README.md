# Mavlink-router-help-IT

## Comunicare con Raspberry Pi tramite MAVLink

Questa pagina spiega come collegare e configurare un Raspberry Pi (RPi) in modo che sia in grado di comunicare con un controller di volo utilizzando il protocollo MAVLink su una connessione seriale. Questo può essere utilizzato per eseguire attività aggiuntive come il riconoscimento delle immagini che semplicemente non può essere eseguito dal controllore di volo a causa dei requisiti di memoria per la memorizzazione delle immagini.

Collegare la porta TELEM2 del controller di volo ai pin Ground, TX e RX di RPi come mostrato nell'immagine sotto: 

![COLLEGARE RASPBERRY PI AL CONTROLLORE DI VOLO](https://user-images.githubusercontent.com/20637640/216408483-319d26d0-904c-4ac0-80a2-f99df7b0a772.png)

![Raspberry-Pi-4-GPIO-header](https://user-images.githubusercontent.com/20637640/216367155-c025f4dc-06db-48d0-8b89-8802b0524489.jpg)


## SCHEMA-PIN-GPIO-Orbitty-TX2

![TX-RX-GPIO-Orbitty-TX2](https://user-images.githubusercontent.com/20637640/216422011-305aeec8-d9e1-4bbe-851f-d3c58a92760c.jpg)

![SCHEMA-PIN-GPIO-Orbitty-TX2](https://user-images.githubusercontent.com/20637640/216366229-a1e3d27c-5800-47c4-b4b2-6eff906b93d7.png)

---------SX-----------

- 01 +3.3V OUTPUT 
- 03 UART0 TX
- 05 UART1 TX
- 07 GPIO-0
- 09 GPIO-2
- 11 I2C CLK
- 13 RECOVERY
- 15 RESET
- 17 POWER BUTTON 
- 19 GROUND

---------DX-----------

- 02 +5V OUTPUT
- 04 UART0 RX
- 06 UART1 RX
- 08 GPIO-1
- 10 GPIO-3
- 12 I2C SDA
- 14 RTC BAT INPUT 
- 16 GND
- 18 GROUND 
- 20 GROUND

NOTE:
In L4T:
- (03) UART0 TX viene visualizzata come /dev/ttyS0
- (04) UART0 TR viene visualizzata come /dev/ttyS0
- (05) UART1 TX viene visualizzata come /dev/ttyTHS2
- (06) UART1 TR viene visualizzata come /dev/ttyTHS2

L'RPi può essere alimentato collegando la sorgente +5V al pin +5V o da ingresso USB, mentre il TX2 a 12V mediante morsetti.


Collegare al controllore di volo con una stazione di terra (ad es. Mission Planner) e impostare i seguenti parametri:

`SERIAL2_PROTOCOL = 2` (impostazione predefinita) per abilitare MAVLink 2 sulla porta seriale.

`SERIAL2_BAUD = 921` in modo che il controllore di volo possa comunicare con l'RPi a 921600 baud.

`LOG_BACKEND_TYPE = 3` se si utilizza APSync per lo streaming dei file di log flash dei dati in RPi

Configurare la porta seriale (UART)

Se non è già configurato, la porta seriale del Raspberry Pi (UART) dovrà essere abilitata. Usa l'utilità di configurazione Raspberry Pi per questo.

Tipo:

sudo raspi-config

- E nell'utilità, seleziona "Opzioni di interfaccia":
- Utility di configurazione RasPi
- E poi “Seriale”:
- Seleziona no nel “Would you like a login shell to be accessible over serial?”.
- Seleziona yes nel “Would you like the serial port hardware to be enabled?”.
- Riavvia il Raspberry Pi quando hai finito.

La porta seriale del Raspberry Pi sarà ora utilizzabile su /dev/serial0

MAVProxy

MAVProxy può essere usato per inviare comandi al controllore di volo dal Pi. Può anche essere utilizzato per instradare la telemetria ad altri endpoint di rete.

Questo presuppone che sia stata impostata una connessione SSH.

Vedi la documentazione MAVProxy per le istruzioni di installazione (https://ardupilot.org/mavproxy/docs/getting_started/download_and_installation.html#mavproxy-downloadinstalllinux)

Per testare l'RPi e il controller di volo sono in grado di comunicare tra loro prima assicurarsi che l'RPi e il controller di volo siano alimentati, quindi in una console sul tipo RPi:

`python3 mavproxy.py --master=/dev/serial0 --baudrate 921600 --aircraft MyCopter`

Una volta avviato MAVProxy dovresti essere in grado di digitare il seguente comando per visualizzare il valore dei parametri ARMING_CHECK
```
param show ARMING_CHECK
param set ARMING_CHECK 0
arm throttle
```
![RaspberryPi_ArmTestThroughPutty](https://user-images.githubusercontent.com/20637640/216369480-ae923641-2b65-4c9b-920c-9cd43ada322c.png)


Nota

Se ricevi un errore sul fatto di non essere in grado di trovare i file di registro o se questo esempio altrimenti non viene eseguito correttamente, assicurati di non aver assegnato accidentalmente questi file a un altro nome utente, come Root.
Per eseguire MAVProxy come router di telemetria sul Pi, impostalo per essere eseguito come servizio e usa i parametri –daemon e –non-interactive. Ad esempio:

`mavproxy.py --daemon --non-interactive --default-modules='' --continue --master=/dev/serial0 --baudrate 1500000 --out=udp:pro:14550`

Nota

Se il Raspberry PI è pesantemente caricato, mavproxy.py potrebbe non fornire un collegamento affidabile per il routing della telemetria. Questo è più probabile su dispositivi più vecchi/più lenti come il Raspberry PI Zero. Se ciò accade, prendere in considerazione l'utilizzo di mavlink-routerd. Vedi questo post sul forum ArduPilot per una discussione dettagliata: MavLink Routing con software Router.

# Router Mavlink

Mavlink-router viene utilizzato per instradare la telemetria tra la porta seriale dell'RPi e qualsiasi endpoint di rete. Vedere la documentazione per le istruzioni di installazione e di esecuzione.

Dopo l'installazione, modificare il file di configurazione di mavlink-router;

`/etc/mavlink-router/main.conf` 

Sezione UART section a:

```
[UartEndpoint to_fc]
Device = /dev/serial0
Baud = 921600
```

È inoltre necessario aggiungere un endpoint UDP aggiuntivo per consentire ad altre stazioni di terra sulla stessa rete di connettersi al Pi. Modificare il file di configurazione di mavlink-router /etc/mavlink-router/main.conf per includere:

```
[UdpEndpoint to_14550_external]
Mode = eavesdropping
Address = 0.0.0.0
Port = 14550
PortLock = 0
```

## Connessione a Mission Planner

Il controller di volo risponderà ai comandi MAVLink ricevuti attraverso le porte Telemetry 1 e Telemetry 2 (vedi immagine nella parte superiore di questa pagina), il che significa che sia l'RPi che la normale stazione di terra (ad es. Il pianificatore di missione, ecc.) può essere collegato. Inoltre è possibile collegare MissionPlanner all'applicazione MAVProxy in esecuzione su RPi in modo simile a come è fatto per SITL.

Principalmente si tratta di aggiungere un --out <ipaddress>:14550 al comando di avvio di MAVProxy, dove l'indirizzo è quello del PC che esegue il pianificatore di missione. Su Windows si può usare ipconfig per determinare l'indirizzo IP. Sul computer utilizzato per scrivere questa pagina wiki il comando MAVProxy è diventato:

`mavproxy.py --master=/dev/ttyAMA0 --baudrate 57600 --out 192.168.137.1:14550 --aircraft MyCopter`

La connessione con il pianificatore della missione è mostrata di seguito:
  
![RaspberryPi_MissionPlanner](https://user-images.githubusercontent.com/20637640/216369254-01c846ab-6d85-45ad-b21d-f26bb2a28d4e.jpg)

  
