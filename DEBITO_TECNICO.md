# Debito tecnico

Problemi noti, lasciati volutamente aperti. Non bloccano l'uso quotidiano:
il server è il nostro, il collegamento è un cavo USB, e nel peggiore dei
casi crasha il client. Questo file serve a ritrovarli con riga e commit
quando (e se) si decide di sistemarli.

Riferimenti alle righe: branch `develop` dopo il commit `3416498`
(`android: log why a stream session ended or crashed`). Se le righe
slittano, cercare le stringhe citate.

Da quel commit ogni caso qui sotto lascia una traccia in logcat, quindi
un crash si diagnostica con:

```
adb logcat -s TethrLink StreamDecoder
```

---

## 1. Lunghezza frame dal server usata senza tetto

- **File**: `android/app/src/main/java/com/tethrlink/MainActivityV2.kt:768`
  (`if (readBuf.size < frameSize) readBuf = ByteArray(frameSize)`)
- **Introdotto in**: `cb515b6` (2026-04-23)
- **Problema**: il server manda un intero a 4 byte come lunghezza del
  frame; il client lo usa direttamente per allocare. Un server bacato
  che manda una lunghezza assurda (fino a 2 GB) provoca
  `OutOfMemoryError`, che è un `Error` e non viene gestito: il processo
  muore.
- **Stato attuale**: logga `Suspicious frame size` sopra 16 MB
  (`READ_BUF_SIZE * 16`), poi alloca comunque. Il crash viene loggato
  dal `catch (t: Throwable)` con `last frame size`.
- **Fix futuro**: scartare il frame o chiudere il socket sopra una soglia
  ragionevole (es. 16 MB per JPEG a risoluzioni alte), lasciando che il
  path di riconnessione nel `finally` riparta.

## 2. Decode JPEG senza protezione da `OutOfMemoryError`

- **File**: `android/app/src/main/java/com/tethrlink/StreamDecoder.kt:246`
  (`BitmapFactory.decodeByteArray(data, 0, data.size)`)
- **Introdotto in**: `cb515b6` (2026-04-23); riga toccata in `3416498`
- **Problema**: un payload JPEG enorme o corrotto può far lanciare
  `OutOfMemoryError` a `BitmapFactory`. Stessa classe del punto 1: non
  catturato, crash.
- **Stato attuale**: un decode che restituisce `null` (JPEG corrotto)
  viene loggato e saltato. L'OOM arriva al `catch (t: Throwable)` di
  `startStreaming`, viene loggato e rilanciato.
- **Fix futuro**: `try/catch (Throwable)` attorno al decode, frame
  scartato, eventualmente `inSampleSize` se la dimensione supera la
  view.

## 3. `catch (e: Exception)` in `startStreaming` lascia passare ogni `Error`

- **File**: `android/app/src/main/java/com/tethrlink/MainActivityV2.kt:780`
  (`} catch (e: Exception) {`)
- **Introdotto in**: `cb515b6` (2026-04-23)
- **Problema**: qualsiasi `Error` (OOM, `NoSuchMethodError`, ...) salta
  il blocco, esce dalla coroutine e uccide l'app. È il meccanismo che ha
  reso invisibile il crash sui tablet Android 7.1.1 e 10 (fix in
  `244e53f`).
- **Stato attuale**: aggiunto `catch (t: Throwable)` che logga con stack
  trace e **rilancia**. Comportamento identico, ma la causa è in logcat.
- **Fix futuro**: decidere se un `Error` nel path di streaming deve
  chiudere la sessione e tornare a `Scanning` invece di uccidere il
  processo. Attenzione: inghiottire un OOM può lasciare l'app in uno
  stato inconsistente; meglio chiudere il socket e ripartire.

## 4. `waitingForSps` è una variabile globale, non per istanza

- **File**: `android/app/src/main/java/com/tethrlink/StreamDecoder.kt:17`
  (`@Volatile private var waitingForSps = true`, fuori dalla classe)
- **Introdotto in**: `17c237f` (2026-05-12)
- **Problema**: resta `false` dopo la prima sessione. Un nuovo
  `StreamDecoder` alla riconnessione accetta slice video prima di aver
  ricevuto SPS/PPS. Oggi non rompe perché il server manda sempre
  SPS/PPS nel primo frame; se un giorno manda prima un P-frame, il
  decoder hardware va in errore non transiente e il video resta nero.
- **Fix futuro**: spostare il campo dentro la classe, resettato a `true`
  nel costruttore.

## 5. `nalQueue.put()` bloccante se il decoder smette di consumare

- **File**: `android/app/src/main/java/com/tethrlink/StreamDecoder.kt:183`
  (`nalQueue.put(videoData)`, e la `put` di config poco sopra)
- **Introdotto in**: `17c237f` (2026-05-12)
- **Problema**: coda da 60 elementi. Se MediaCodec resta vivo ma non
  rilascia più input buffer (stallo hardware), dopo 60 frame la `put`
  blocca per sempre il thread IO della coroutine. `streamJob.cancel()`
  non interrompe una `put` bloccata: il pulsante "End" non funziona e il
  socket non viene chiuso. Serve uccidere l'app.
- **Stato attuale**: nessun log, non ancora osservato dal vivo.
- **Fix futuro**: `offer(data, timeout)` con log e scarto del frame, o
  chiudere la sessione dopo N secondi senza input buffer disponibili.

## 6. `isUsbTetherIp()` apre in caso di eccezione

- **File**: `android/app/src/main/java/com/tethrlink/MainActivityV2.kt:445`
  (`} catch (_: Exception) { true }`)
- **Introdotto in**: `cb515b6` (2026-04-23)
- **Problema**: se l'enumerazione delle interfacce fallisce, il filtro
  "il server è sulla subnet USB" ritorna `true` e accetta qualunque IP
  dal beacon di discovery. Il server rifà il controllo dal suo lato, per
  cui l'impatto reale è nullo.
- **Fix futuro**: `return false` e log dell'eccezione.

## 7. API 23 usate con `minSdk 21`

- **File**:
  - `android/app/src/main/java/com/tethrlink/SettingsActivity.kt:108` e
    `:113` (`resources.getColor(id, theme)`)
  - `android/app/src/main/res/values/themes.xml:14`
    (`android:windowLightStatusBar`)
- **Introdotto in**: `cb515b6` (2026-04-23), `7329288` (2026-04-03)
- **Problema**: `getColor(int, Theme)` non esiste sotto API 23: su
  Android 5.0/5.1 la schermata Settings crasha con `NoSuchMethodError`.
  `windowLightStatusBar` viene solo ignorato, nessun crash.
- **Stato attuale**: lint `NewApi` li segnala ad ogni build
  (`./gradlew :app:lintDebug`). Nessun device Android 5 su cui provare,
  quindi lasciati stare.
- **Fix futuro**: `ContextCompat.getColor(this, id)` (già fatto nelle
  pill di scala in `6d04919`), e spostare `windowLightStatusBar` in
  `values-v23/themes.xml`. In alternativa alzare `minSdk` a 23.

## 8. `allowBackup="true"` nel manifest

- **File**: `android/app/src/main/AndroidManifest.xml:10`
- **Introdotto in**: `42175e0` (commit iniziale)
- **Problema**: il backup Android include le SharedPreferences, quindi
  il device id (16 byte casuali, usato dal server solo per riconoscere
  il tablet). Non è un segreto, impatto trascurabile.
- **Fix futuro**: `android:allowBackup="false"`, oppure un
  `fullBackupContent` che escluda `tethrlink_device.xml`.

---

## Note sul lint

`./gradlew :app:lintDebug` fallisce per i 3 errori del punto 7 e
basta. Finché restano, il lint non è utilizzabile come gate di build.
Quando si sistema il punto 7, aggiungere in `android/app/build.gradle.kts`:

```kotlin
lint {
    abortOnError = true
}
```

e farlo girare prima di ogni release: avrebbe intercettato il crash sui
tablet Android 7.1.1 / 10 prima della 2.0.1.
