# Peer review indipendente — Mac mini M2 Pro come workstation remota per sviluppo e agenti AI

**Versione:** 0.1 — 12 novembre 2026  
**Stato:** PROPOSTA DA REVISIONARE. Nessuna impostazione è stata applicata o verificata sul Mac mini reale.  
**Destinatario:** Claude (revisore tecnico indipendente).

## 0. MANDATO AL REVISORE (LEGGERE PRIMA DI TUTTO)

Agisci come **senior macOS administrator, security engineer e sviluppatore esperto di Claude Code, OpenAI Codex, SSH, Tailscale e workflow di sviluppo remoto**. Esegui una **peer review critica, rigorosa e indipendente** della proposta seguente. NON limitarti a confermare il piano; cerca errori, omissioni, false assunzioni, vulnerabilità, alternative migliori, istruzioni obsolete, difficoltà operative e incompatibilità fra strumenti. La configurazione deve funzionare davvero su Mac mini M2 Pro e smartphone Android, non solo sulla carta.

**Regole:**
1. **Non eseguire installazioni, modifiche o comandi distruttivi.** La prima revisione è solo di analisi. Se hai accesso al Mac su cui gira Claude Code, puoi proporre comandi di diagnostica **esplicitamente read-only**, da eseguire soltanto previa autorizzazione. Non leggere segreti, token o chiavi private, e non stampare seriali o dati sensibili.
2. Usa documentazione **ufficiale e aggiornata a novembre 2026**, distinguendo fatti verificati, deduzioni e aspetti non accertabili senza prova sul dispositivo. Riporta URL ufficiali per ogni punto tecnico controverso; evita di inventare funzioni o parametri.
3. Valuta **sicurezza, affidabilità, semplicità d'uso, manutenzione e costi**. Non proporre un'architettura da datacenter se una configurazione più semplice soddisfa gli obiettivi.
4. **Contesta esplicitamente anche l'architettura di ChatGPT** dove ritieni opportuno. Se cambieresti un componente (Tailscale, Termius, tmux, accesso remoto nativo, app macOS, VPN sul router, git worktrees), motivane il motivo e il trade-off.
5. Distingui **MVP sicuro da configurare subito** e **miglioramenti opzionali**. Evita di ridurre la protezione FileVault o di attivare login automatico come scorciatoia senza analisi del rischio e consenso esplicito.
6. Segnala criticità con severità **A = blocca configurazione o rischio grave**, **B = correggere prima dell'uso esterno**, **C = miglioramento non bloccante**; per ognuna: punto interessato, rischio/scenario, fonte, modifica minima e test.
7. Il risultato richiesto è un **runbook definitivo e riproducibile**: fasi in ordine, prerequisiti, passaggi GUI, comandi esatti solo quando verificati, risultati attesi, test GO/NO-GO, rollback, piano di recupero e cose da NON fare.

## 1. CONTEXTO, HARDWARE E OBIETTIVO

**Mac mini:** Apple M2 Pro, CPU 10 core, GPU 16 core, Neural Engine 16 core, 32 GB memoria unificata, SSD 1 TB. Deve rimanere acceso sulla rete domestica o dell'ufficio. **Versione macOS NON ancora rilevata**, così come FileVault, stato login e disponibilità di monitor/utente presente.

**Client:** MacBook Pro 16 pollici (2023) e smartphone Google Pixel (Android). L'utente sviluppa Next.js/TypeScript/Cloudflare, WordPress, Python, PostgreSQL, progetti GitHub e vuole usare Claude Code e OpenAI Codex su repository locali. Alcuni progetti appartengono a clienti: elevata importanza di sicurezza e separazione dei dati.

**Risultato desiderato:**
- Dal MacBook, operare sul Mac mini come macchina di sviluppo remota, possibilmente con VS Code Remote SSH, terminale ed eventualmente desktop grafico.
- Dal Pixel, da rete mobile 4G/5G fuori casa, aprire il terminale del Mac mini per lanciare comandi e riprendere le sessioni attive; poter usare Claude Code/Codex senza ridurre il telefono a un remoto desktop scomodo.
- Poter avviare o riprendere attività da app Claude/ChatGPT quando la relativa funzione remota è disponibile sul piano, avendo sempre SSH come piano B.
- I processi avviati in una sessione persistente devono restare attivi dopo una disconnessione SSH; niente promessa di sopravvivenza automatica a reboot, crash o esaurimento quote AI.
- Minimizzare esposizione di rete, costi aggiuntivi, complessità e rischio di modifiche incontrollate.

**Informazioni ANCORA SCONOSCIUTE:** versione macOS; quale account sul Mac mini sarà utilizzato; già installati Homebrew, SSH, tmux, Claude Code, Codex, app desktop; sottoscrizioni e disponibilità feature remote; FileVault; rete Ethernet/Wi-Fi, router, ISP e possibilità di intervento fisico; backup; volume dei progetti; eventuali vincoli aziendali/licenze Tailscale.

## 2. ARCHITETTURA CANDIDATA DA METTERE IN DISCUSSIONE

```text
                         Internet
                            |
        +-------------------+------------------+
        |                                      |
  MacBook Pro                               Pixel Android
  Tailscale                                 Tailscale
  SSH / VS Code Remote SSH                  Termius o altro SSH client
  eventuale Screen Sharing                  Claude app / ChatGPT app
        |                                      |
        +---------------+----------------------+
                        |
                RETE PRIVATA TAILSCALE
                        |
                  Mac mini M2 Pro
             macOS Login remoto / OpenSSH
              shell, tmux, git worktrees
             Claude Code CLI / Codex CLI
          Claude Remote Control (opzionale)
          Codex app Remote (opzionale)
          Docker/Node/Python/PostgreSQL
```

**Base preferita:** client Tailscale **Standalone** macOS e Android/macOS; connessione SSH **nativa di macOS** sull'indirizzo Tailscale, con chiavi SSH distinte per MacBook e Pixel; `tmux` per riprendere le sessioni. Le funzioni Remote Control dei prodotti AI sono *interfacce complementari*; non sostituiscono SSH per gestione generale e recupero.

**Attenzione:** NON confondere «SSH attraverso Tailscale» con «Tailscale SSH». Nelle varianti macOS Standalone e App Store il componente server Tailscale SSH non è disponibile; la variante open-source `tailscaled` ha caratteristiche diverse e può essere eseguita prima del login, ma aggiunge complessità. Verificare se e quando valga la pena adottarla.

## 3. RUNBOOK CANDIDATO (DA CORREGGERE IN PEER REVIEW)

### Fase 0 — Diagnostica read-only e inventario

Prima di modificare il Mac mini, raccogliere **senza segreti**:

```bash
sw_vers
uname -m
id -un
df -h /
command -v brew || true
command -v tmux || true
command -v claude || true
command -v codex || true
command -v tailscale || true
pmset -g custom
fdesetup status
```

Verificare a mano: stato di Login remoto e Condivisione schermo, FileVault, sistema aggiornato, connessione Ethernet, piano backup, login dopo reboot, utenti autorizzati, VPN già installate e presenza di progetti o servizi critici da non interrompere. Non pubblicare hostname interni, IP pubblici, e-mail, token, chiavi e seriali nel rapporto.

**Checkpoint:** inventario disponibile; nessuna modifica; versione macOS nota. Se la versione di macOS non è nota, NON dare per certi percorsi UI e funzionalità FileVault.

### Fase 1 — Backup e confini del progetto

- Prima di cambiare impostazioni, verificare backup ripristinabile (es. Time Machine su supporto dedicato) e repository Git con remoto funzionante. GitHub NON sostituisce il backup di database, file non tracciati e configurazioni.
- Segreti solo in password manager/Keychain o `.env` protetti e fuori Git. Nessuna copia incauta di chiavi SSH/API fra dispositivi.
- Valutare un utente dedicato allo sviluppo con privilegi minimi, purché non complichi inutilmente accesso al software e ai progetti già esistenti.
- Eseguire aggiornamenti macOS importanti solo quando l'accesso locale è garantito e le sessioni sono state salvate.

### Fase 2 — Energia e condivisione macOS

- In `Impostazioni di Sistema > Energia` (percorsi da confermare sulla versione installata): abilitare **«Impedisci al Mac di andare automaticamente in standby quando lo schermo è spento»**; lasciare possibile il blocco dello schermo.
- Considerare «Attiva per l'accesso alla rete» solo dopo aver verificato se serve; non equivale necessariamente a garantire lavoro 24/7.
- `Impostazioni di Sistema > Generali > Condivisione > Login remoto`: abilitare SSH e limitarlo a **«Solo questi utenti»**; NON attivare indiscriminatamente «accesso completo al disco per tutti gli utenti».
- `Condivisione schermo` soltanto se serve, con accessi ristretti; NON esporre VNC direttamente a Internet.
- Valutare Ethernet e un piccolo UPS, se la disponibilità remota è importante.

### Fase 3 — Rete privata, accesso e autenticazione

- Installare una sola variante Tailscale per dispositivo; per macOS considerare **Standalone** suggerita dal fornitore; attivare 2FA sull'identity provider.
- Collegare Mac mini, MacBook e Pixel allo stesso tailnet, verificare MagicDNS o usare l'IP Tailscale del Mac mini; applicare policy di accesso minime, se utili.
- Sul MacBook creare/utilizzare chiave SSH dedicata (preferibilmente Ed25519, protetta da passphrase). Sul Pixel creare **un'altra** chiave tramite client SSH affidabile e protezione dell'app, senza trasferire in chiaro quella privata del MacBook.
- Aggiungere **solo le chiavi pubbliche** appropriate a `~/.ssh/authorized_keys` dell'utente autorizzato del Mac mini, permessi corretti; NON disabilitare autenticazione password a livello `sshd` prima di avere verificato accesso con chiavi da entrambi i client e percorso di recupero.
- Test: SSH da MacBook nella stessa rete, poi SSH dal Pixel usando **rete cellulare e Wi-Fi disattivato**. Eseguire solo comandi innocui (`pwd`, `whoami`, `uname -m`).

**Checkpoint obbligatorio:** il Pixel entra via SSH dal cellulare e il MacBook entra via SSH senza aperture di porte sul router. Se fallisce, fermarsi qui.

### Fase 4 — Sviluppo e sessioni persistenti

- Verificare Homebrew (`/opt/homebrew` è il prefisso usuale Apple Silicon), poi installare `tmux` solo se assente; aggiornare gli strumenti richiesti compatibilmente con progetti esistenti.
- `tmux` gestisce sessioni di shell sganciate dal client SSH; **non sopravvive automaticamente al riavvio del Mac**.
- Esempio (solo dopo aver confermato che la cartella esiste):

```bash
cd ~/Progetti/edtech      # Percorso SEGNAPOSTO, NON presumere esista
tmux new -s edtech
# Ora, dentro tmux, avviare uno degli strumenti:
claude
# oppure, in una sessione separata e appropriata:
# codex
```

Dal Pixel in una nuova connessione SSH:

```bash
tmux ls
tmux attach -t edtech
```

- Confermare la sequenza corretta per detach/reconnect anche da terminale Android e MacBook, evitando la disconnessione dell'agente con `exit` involontario o chiusura del processo.
- Attenzione ai progetti modificati simultaneamente da due agenti. Privilegiare branch o **git worktrees isolati**, con integrazione solo dopo code review, test e commit controllato.

### Fase 5 — Claude Code: normale SSH e Remote Control

- Verificare l'installazione con `claude --version` e autenticazione sul Mac mini. Non dare per scontata disponibilità o costo della funzione remota: verificare il piano Claude effettivo (Pro/Max/Ultra/Team/Enterprise) e le versioni CLI.
- Per proseguire un agente nel **terminale** del Pixel: sessione Claude Code dentro `tmux` + SSH.
- Per un'interfaccia più comoda sul **telefono**: valutare `claude --remote-control` per sessione interattiva già usata dal terminale, o `claude remote-control --name "EdTech"` per server remoto; i comportamenti sono differenti. Avviare dentro `tmux` se appropriato, così il processo resta in vita a una disconnessione SSH.
- In server mode, valutare `--spawn worktree` per sessioni multiple anziché la condivisione dello stesso checkout; testare l'esperienza reale da Android.
- Una sessione Remote Control necessita del processo Claude locale in esecuzione: **non equivale a un agente cloud e non riparte da sola dopo un reboot**.
- Mai impostare bypass globale delle approvazioni per semplificare l'uso da smartphone; limitare accessi a filesystem, shell, GitHub, siti e tool esterni.

### Fase 6 — Codex CLI e Codex Remote

- Verificare Codex CLI installato e autenticazione: `codex --version`. Accesso **universale** da Android via SSH/tmux (CLI sul Mac mini).
- Valutare **separatamente** le attività Codex nella nuova app desktop ChatGPT/Codex e la scheda `Remote` su ChatGPT Android. Richiede disponibilità funzionalità per account/area, accoppiamento, host attivo e app appropriate; NON assumere che una vecchia sessione `codex` in tmux sia automaticamente visibile o controllabile come chat Codex Remote.
- Distinguere **Codex Cloud** (ambiente remoto gestito da OpenAI e indipendente dal Mac acceso) da **Codex Remote** (controllo del Mac locale).
- Anche qui: Git worktrees, limiti di autorizzazione, revisione diff e controllo umano prima del deploy.

### Fase 7 — Editor, preview e desktop remoto (facoltativo)

- MacBook: VS Code Remote SSH per usare editor locale con filesystem, test e runtime del Mac mini.
- Pixel: terminale SSH prima scelta; desktop remoto solo per attività che richiedono GUI e quando è utilizzabile sullo schermo piccolo.
- Anteprime Next.js/servizi: preferire tunnel SSH (`ssh -L ...`) o accesso privato Tailscale; evitare porte di sviluppo aperte sul router, `0.0.0.0` non necessario senza motivo, Funnel/public URL non richiesti.
- Valutare se un browser privato raggiunge correttamente la preview e come controllare le sessioni server dopo la disconnessione.

### Fase 8 — Continuità, failure mode e ripristino

**QUESTA È LA PARTE PIÙ IMPORTANTE DELLA PEER REVIEW.**

- Le versioni **Tailscale macOS Standalone e App Store** funzionano nella sessione utente e **non sono normalmente raggiungibili prima del login dopo riavvio**. La variante open-source `tailscaled` può comportarsi diversamente, ma ha più complessità e compromessi. È necessario decidere che livello di recupero unattended è realisticamente raggiungibile.
- La documentazione Apple per Mac Apple Silicon con **macOS 26 o successivo** contempla lo **sblocco FileVault via SSH dopo il riavvio**, a certe condizioni di rete. Valutare in concreto se è raggiungibile *da fuori casa* senza una connessione VPN attiva prima del login, e quali componenti sul router/LAN sarebbero necessari. Non confondere **SSH preboot FileVault** con lo SSH standard dopo il login.
- Valutare se il Mac deve restare con sessione utente aperta e **schermo bloccato**, anziché fare logout. Non suggerire login automatico o disattivazione FileVault come default.
- Controllare disponibilità dopo restart di macOS, power loss, connessione Internet interrotta, token Tailscale scaduto, account AI disconnesso, agente che attende approvazione, app desktop chiusa, tmux terminato, disco esaurito.
- Per servizi che devono davvero avviarsi al boot valutare `launchd` **solo dove necessario e sicuro**; non automatizzare la partenza di agenti con permessi estesi.
- Definire una procedura «se il Mac mini è irraggiungibile» con fallback fisico esplicito. In particolare chiarire **quali problemi sono risolvibili senza presenza in sede e quali no**.

## 4. MATRICE DI TEST DA COMPRENDERE NEL RUNBOOK FINALE

| ID | Test | Risultato minimo richiesto |
|---|---|---|
| T01 | SSH MacBook in LAN con Tailscale | Autenticazione con chiave, shell corretta |
| T02 | SSH Pixel su 4G/5G, Wi-Fi spento | Connessione e comandi read-only |
| T03 | Avvio agente in `tmux`, disconnessione telefono, riattacco | Sessione ancora attiva, nessuna perdita di stato |
| T04 | Passaggio MacBook → Pixel → MacBook | Stessa sessione/checkout con stato coerente |
| T05 | Claude Remote Control dall'app Android | Sessione locale accessibile e autorizzazioni sensate |
| T06 | Codex Remote da ChatGPT Android | Feature disponibile e host controllabile, o fallback documentato |
| T07 | Mac mini con schermo bloccato | SSH e servizi richiesti ancora accessibili |
| T08 | Logout dall'utente macOS | Misurare comportamento reale Tailscale/SSH/app GUI |
| T09 | Standby / display spento | Niente sospensione indesiderata dei lavori |
| T10 | Riavvio controllato con FileVault | Accesso e percorso di recupero documentati, senza ipotesi |
| T11 | Router/internet assente e poi ripristinato | Esito e modalità di recupero |
| T12 | Collisione tra due agenti sullo stesso repository | Nessun conflitto, mediante worktree/branch e workflow adeguato |
| T13 | Revoca dispositivo mobile smarrito | Nessuna possibilità residua di accesso remoto |
| T14 | Backup e ripristino di un file/progetto non critico | Recupero dimostrato |
| T15 | Preview web raggiunta solo in privato | Nessuna esposizione pubblica involontaria |
| T16 | Agente bloccato per approvazione/quota/API | Rilevabile, con intervento remoto possibile |

**Importante:** i test con logout, reboot, FileVault e cambio firewall vanno eseguiti **solo con presenza fisica e piano di recupero**; non usare test distruttivi o simulare interruzioni di corrente in modo rischioso.

## 5. DOMANDE SPECIFICHE A CUI DEVI RISPONDERE

1. L'architettura Tailscale Standalone + OpenSSH macOS + Termius/altro client Android + tmux è la scelta più robusta, semplice e sicura per questo caso? Quali alternative sono superiori e in quali condizioni?
2. Quali sono gli errori, i dettagli imprecisi o le omissioni nella procedura proposta?
3. È corretto evitare Tailscale SSH server su macOS Standalone? Serve davvero la variante `tailscaled` per l'obiettivo indicato?
4. Come gestire in sicurezza l'accesso SSH con chiavi separate, Keychain/biometria su telefono, privilegi minimi, autorizzazioni e blocco/smarrimento del Pixel?
5. Quale comportamento reale hanno Tailscale, sshd, CLI, `tmux` e app desktop quando il Mac è **bloccato**, l'utente fa **logout**, il Mac **va in sleep**, **si riavvia**, o torna la corrente? Distingui ogni scenario.
6. FileVault + macOS 26+ + sblocco SSH preboot: come funziona, quale rete serve e perché potrebbe NON essere praticabile dall'esterno usando solo Tailscale normale? Come testarlo senza perdere accesso?
7. È utile una VPN sul router o un dispositivo LAN sempre acceso per l'accesso d'emergenza? È eccessivo per il caso d'uso? Quando passare a quella soluzione?
8. Meglio `tmux`, `screen`, `mosh`, SSH standard, app Remote Control nativa, o un mix? Trade-off su Android, reti mobili instabili e processo in esecuzione.
9. Corretto usare `claude --remote-control` vs `claude remote-control`? Quale scegliere se si vuole partire da MacBook e continuare su Pixel? Come evitare sessioni multiple che modificano gli stessi file?
10. Quali condizioni di piano/versione devono essere soddisfatte per Claude Code Remote Control e Codex Remote su Android a novembre 2026?
11. Codex CLI in tmux può essere proseguito da Android solo via SSH? La scheda Remote dell'app ChatGPT riguarda le chat dell'app desktop? Distinguere con precisione.
12. Quali strumenti per sviluppo web, database, versioning, segreti, ambienti Python/Node, processi persistenti e anteprime sono davvero necessari subito?
13. Come rendere disponibili le preview Next.js al telefono senza esporle al web? Tunnel SSH, Tailscale Serve o alternative; test e sicurezza.
14. Quali pericoli ci sono con Claude Code e Codex sulla stessa cartella o sullo stesso branch? Proporre workflow pratico con branch/worktree e approvazioni.
15. Come impostare backup e ripristino, patching, spazio SSD, logging, stima consumi, aggiornamenti e controllo costi API/abbonamenti?
16. Quali sono i **5 test bloccanti** prima di dichiarare la configurazione pronta all'uso fuori casa?
17. Come costruire una configurazione incrementale reversibile, riducendo al minimo privilegi, nuove app e punti di guasto?
18. Qual è la migliore esperienza reale per l'utente: SSH/Termius, interfacce remote native, o entrambi? Non dare per scontato che tutto si faccia comodamente da 6 pollici di schermo.

## 6. FORMATO OBBLIGATORIO DELLA RISPOSTA

Restituisci le seguenti sezioni, **nell'ordine**:

**A. Verdetto sintetico:** GO / CONDITIONAL GO / NO-GO e motivazione; giudizio sulla scelta architetturale.

**B. Tabella dei rilievi:** ID, gravità A/B/C, sezione, scenario concreto, evidenza/link ufficiale, correzione minima, test di validazione. Distingui fonte da inferenza.

**C. Architettura finale proposta:** diagramma testuale e componenti **obbligatori, opzionali e sconsigliati**, con motivazioni e costi/licenze da verificare.

**D. Runbook definitivo eseguibile:** fasi ordinate, prerequisiti, percorsi UI macOS a seconda della versione, comandi esatti verificati, output attesi, checklist GO/NO-GO a fine fase, rollback. Specifica quali azioni richiedono presenza fisica.

**E. Matrice di collaudo:** priorità, test pratici, expected result, blocchi e procedure di recupero. Includi test su rete mobile, blocco schermo, logout, reboot, FileVault, doppio agente, smarrimento telefono.

**F. Incident playbook:** «non raggiungo il Mac mini», «Claude/Codex non rispondono», «riavvio imprevisto», «chiave SSH compromessa», «collisione Git», «backup fallito». Per ogni caso: diagnosi sicura e percorso di ripristino.

**G. Informazioni mancanti:** elenca **solo le informazioni che impediscono una decisione corretta** e indicami come raccoglierle con comandi read-only.

**H. Fonti ufficiali:** URL, titolo, data di verifica o aggiornamento quando disponibile; marca come *non verificata* qualsiasi affermazione importante non sostenuta.

**Non modificare niente sul Mac: questa è la revisione di progetto e procedura, non l'esecuzione.**

## 7. FONTI UFFICIALI DI PARTENZA (VERIFICATE DA CHATGPT, DA RICONTROLLARE)

- Apple — Impostazioni standby/energia Mac: https://support.apple.com/it-it/guide/mac-help/-mchle41a6ccd/mac
- Apple — Login remoto via SSH: https://support.apple.com/it-it/guide/mac-help/mchlp1066/mac
- Apple — FileVault, inclusi Mac Apple Silicon macOS 26+: https://support.apple.com/it-it/guide/deployment/dep82064ec40/web
- Tailscale — Client macOS: https://tailscale.com/docs/install/mac
- Tailscale — Varianti macOS e compatibilità Tailscale SSH: https://tailscale.com/docs/concepts/macos-variants
- Tailscale — SSH e limitazioni macOS: https://tailscale.com/docs/features/tailscale-ssh
- Tailscale — Accesso senza login utente: https://tailscale.com/docs/how-to/run-unattended
- Anthropic — Claude Code Remote Control: https://code.claude.com/docs/en/remote-control
- Anthropic — Claude Code power user tips: https://support.claude.com/en/articles/14554000-claude-code-power-user-tips
- OpenAI — ChatGPT Work e Codex / remote mobile: https://help.openai.com/it-it/articles/20001275-chatgpt-work-and-codex
- OpenAI — Codex Remote e utilizzo da telefono: https://developers.openai.com/blog/mastering-codex-remote-for-engineering
- OpenAI — Configurazione Remote da mobile: https://developers.openai.com/training/walkthroughs/working-on-your-phone
- VS Code — Remote SSH: https://code.visualstudio.com/docs/remote/ssh
- Homebrew — Installazione Apple Silicon: https://docs.brew.sh/Installation.html

---
**Istruzione finale a Claude:** tratta questa proposta come un progetto da verificare e migliorare, NON come specifica già approvata. Le feature remote e i limiti delle versioni cambiano: verifica, correggi e metti in evidenza tutto ciò che potrebbe rendere il Mac mini irraggiungibile da fuori casa. Non formulare un «GO» finché la strategia di recupero dopo il reboot non è chiara e sono definiti i test bloccanti.