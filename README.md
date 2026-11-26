# Notifiche Telegram

App Android (Kotlin nativo) che legge le notifiche del telefono e le inoltra in tempo reale a un bot Telegram.

## Funzioni
- **NotificationListenerService**: legge titolo, testo, app di provenienza e orario di ogni notifica.
- **Configurazione**: token del bot e Chat ID (con pulsante per trovare il Chat ID in automatico) e pulsante "Invia test".
- **Filtri**: esclusione di singole app (con ricerca e app di sistema), opzione per ignorare le notifiche persistenti, salto dei riepiloghi di gruppo e dei duplicati ripetuti entro 60 secondi.
- **Invio**: POST JSON a `sendMessage` usando le coroutine (Dispatchers.IO), in ordine, con nuovi tentativi su errori di rete, 429 e 5xx.
- **Background**: foreground service con notifica permanente (`START_STICKY`), riavvio all'accensione e dopo ogni aggiornamento dell'app, ricollegamento automatico del listener.

Formato del messaggio:
```
📱 WhatsApp
Mario Rossi
Ciao, ci vediamo alle 18?

🕒 26/11/2026 08:58:12
```

## 1. Crea il bot
1. Su Telegram apri **@BotFather** → `/newbot` → scegli nome e username → copia il **token**.
2. Apri il tuo bot e premi **AVVIA** (`/start`). Senza questo passaggio il bot non può scriverti.

## 2. Compila l'APK con GitHub Actions
1. Crea un repository su GitHub e carica tutto il contenuto di questa cartella (compresa `.github/`).
2. Il workflow **Build Notifiche Telegram APK** parte a ogni push su `main`, oppure a mano da *Actions → Run workflow*.
3. A fine build scarica l'artifact **NotificheTelegram-apk** (è uno zip che contiene l'APK).

### Firma (consigliata per poter aggiornare senza disinstallare)
Se non configuri niente, viene creato un APK **debug**: si installa subito, ma ogni build è firmata con una chiave diversa, quindi per aggiornare devi disinstallare (perdendo la configurazione).

Per avere una firma fissa crea un keystore una volta sola:
```bash
keytool -genkeypair -v -keystore notifiche.jks -alias notifiche \
  -keyalg RSA -keysize 2048 -validity 10000
base64 -w0 notifiche.jks > notifiche.jks.b64
```
Poi in *Settings → Secrets and variables → Actions* aggiungi:

| Secret | Valore |
|---|---|
| `NOTIFICHE_KEYSTORE_BASE64` | contenuto di `notifiche.jks.b64` |
| `NOTIFICHE_KEYSTORE_PASSWORD` | password del keystore |
| `NOTIFICHE_KEY_ALIAS` | `notifiche` |
| `NOTIFICHE_KEY_PASSWORD` | password della chiave |

Da quel momento il workflow genera `app-release.apk` firmata. Conserva il file `.jks`: non va caricato nel repository (è già escluso nel `.gitignore`).

## 3. Primo avvio sul telefono
1. Installa l'APK e consenti le notifiche dell'app.
2. Inserisci il **token**, premi **Trova il mio Chat ID automaticamente** (oppure scrivi l'ID a mano), poi **Salva** e **Invia test**.
3. Premi **Concedi accesso alle notifiche** e attiva *Notifiche Telegram*.
   - **Android 13 e successivi**: le app installate da APK hanno questa impostazione bloccata ("Impostazione con restrizioni"). Vai in **Apri Info app → menu ⋮ in alto a destra → Consenti impostazioni con restrizioni**, poi riprova.
4. Premi **Disattiva ottimizzazione batteria**.
5. Su **Xiaomi/Redmi/POCO, Huawei, Oppo, Samsung**: attiva anche l'*Avvio automatico* / togli le restrizioni di batteria dalle impostazioni dell'app, altrimenti il sistema può bloccare il riavvio all'accensione.

## Note
- Le app Telegram sono **escluse di default**, altrimenti i messaggi del bot verrebbero reinoltrati all'infinito. Non toglierle dall'elenco se il bot scrive allo stesso telefono.
- Per un canale pubblico puoi usare `@nomecanale` come Chat ID (il bot deve essere amministratore del canale). Per la chat privata con il bot serve l'ID numerico.
- Il token è salvato solo sul telefono (SharedPreferences privati) e il backup dell'app è disattivato.

## Struttura
```
app/src/main/java/com/emanuel/notifiche/
  MainActivity.kt                  interfaccia di configurazione e stato
  AppFilterActivity.kt             elenco app da escludere
  NotificationForwarderService.kt  lettura notifiche, filtri, inoltro
  TelegramClient.kt                chiamate HTTP alle API di Telegram
  KeepAliveService.kt              foreground service con notifica permanente
  BootReceiver.kt                  avvio all'accensione e dopo un aggiornamento
  Prefs.kt                         impostazioni salvate
```
Requisiti: Android 8.0+ (minSdk 26), target Android 15 (SDK 35).
