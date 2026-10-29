# DouyinLiquidGlass

Ispirato all'estetica "glass" (effetto vetro) di [winaviation-tweaks/liquidass](https://github.com/winaviation-tweaks/liquidass) — caratterizzata da sfocatura dello sfondo in tempo reale, riflessi diagonali, angoli arrotondati e completa configurabilità — questo plugin è progettato specificamente per l'**interfaccia utente interna dell'app Douyin** (pulsanti, barre di navigazione inferiori, barre di ricerca, ecc.).

- Artefatti di compilazione: `.dylib` (per l'iniezione nell'app) e `.deb` (per l'installazione tramite Sileo).
- Agisce esclusivamente sul processo Douyin (bundle ID: `com.ss.iphone.ugc.Aweme`); compatibile con DYKiller (nessun conflitto).
- Supporta iOS 14.0 e versioni successive.

## Architettura (v0.5.6, versione stabile definitiva)

**Non si affida a `%hook` di CydiaSubstrate o ElleKit**; utilizza invece la tecnica di *method swizzling* del runtime di Objective-C. Caratteristiche principali:

1. **Capsule fluttuanti (non interferiscono con i contenitori nativi)**: Il contenitore originale della barra di navigazione inferiore mantiene la sua posizione e dimensione nativa. All'interno di questo contenitore viene renderizzata una capsula con effetto "vetro", che accoglie naturalmente gli elementi nativi della barra. Poiché il contenitore non viene spostato, non si verificano problemi con le barre di avanzamento né conflitti con l'AutoLayout; inoltre, l'effetto vetro rimane visibile anche durante il passaggio da una scheda all'altra.
2. **Hooking mirato**: Il metodo `layoutSubviews` viene intercettato solo per le classi a cui è stato assegnato l'effetto vetro, garantendo prestazioni elevate senza rallentare l'app Douyin.
3. **Effetto "vetro smerigliato" stabile**: Sfrutta `UIVisualEffectView` (stabile all'interno del processo dell'app), combinato con riflessi diagonali disegnati ad hoc e un bordo superiore luminoso. Gli elementi di forma quadrata — come pulsanti o avatar posizionati sulla destra — vengono trasformati automaticamente in **capsule dagli angoli arrotondati**.
4. **Rimozione selettiva dello sfondo e gestione corretta dei livelli**: Vengono nascoste solo le viste che occupano l'intera area e che fungono da sfondo (ad esempio, quelle il cui nome di classe contiene il termine "Blur" e che sono prive di icone o testo). **Qualsiasi sotto-vista (subview) contenente un `UIImageView` o un `UILabel` viene rigorosamente preservata** (garantendo che avatar e icone non vengano mai nascosti accidentalmente). Se l'elemento "glass" (effetto vetro) viene rimosso a causa dei ricalcoli del layout nativo, esso viene automaticamente riagganciato al livello inferiore. **Stato "Off" predefinito per la diagnostica e migrazione con autoriparazione**: le impostazioni `Debug`/`ShowClassTag` non leggono più il vecchio file `.plist` (per evitare che valori residui causino problemi). Al loro posto viene utilizzata una **nuova chiave, `DiagnosticMode`** (impostata di default su "Off"; da attivare su "YES" solo quando è necessario rilevare i nomi effettivi delle classi). A partire dalla versione 0.5.8, il tweak **esegue una pulizia automatica** di eventuali chiavi di configurazione obsolete e problematiche al primo avvio, forzando il parametro `Enabled=YES`. In questo modo, anche qualora sul dispositivo rimanessero file `.plist` "sporchi", non si verificheranno più anomalie come riquadri rossi, etichette di classe persistenti o il glitch che rende invisibile l'intera interfaccia utente ("all-glass").

**Ripristino dell'euristica geometrica (v0.5.9)**: Poiché il rilevamento dei nomi delle classi per i pulsanti situati sul lato destro è diventato inaffidabile (a causa dei frequenti cambiamenti nelle versioni 40.x), è stato ripristinato il **metodo basato sull'euristica geometrica per il lato destro**. Tale metodo funge ora da meccanismo di riserva (fallback), identificando gli elementi in base a criteri specifici: forme piccole e tendenzialmente quadrate, posizionate a una distanza di 30–80pt dal bordo destro, collocate nella sezione centrale verticale e contenenti icone. Per gli elementi posizionati nella parte superiore o inferiore dello schermo, il sistema continua ad affidarsi a una whitelist precisa di nomi di classe, evitando così stime geometriche approssimative.

Lo stesso file `.dylib` è **compatibile con entrambi i metodi di installazione**:
- Installazione tramite **pacchetto `.deb` via Dopamine (Sileo)**, con iniezione nel processo TikTok gestita da ElleKit.
- Iniezione diretta nel file `.ipa` di TikTok tramite **TrollStore / TrollFools** (senza necessità di un ambiente jailbreak attivo). ## Configurazione (`/var/mobile/Library/Preferences/com.dy.liquidglass.plist`)

| Chiave | Predefinito | Descrizione |
|---|---|---|
| `Enabled` | YES (forzato al primo avvio) | Interruttore principale |
| `DiagnosticMode` | NO | Nuova modalità diagnostica: impostare su YES per visualizzare etichette rosse con i nomi delle classi (utile per identificare i nomi reali delle classi tramite screenshot); reimpostare su NO una volta terminato. |
| `Floating` | YES | Attiva/disattiva la capsula fluttuante nella barra inferiore |
| `SideHeuristic` | YES | Meccanismo geometrico di fallback per i pulsanti sul lato destro (rileva piccoli quadrati, elementi posizionati centralmente e componenti che contengono icone lungo il bordo destro); può essere impostato su NO una volta identificati i nomi specifici delle classi. |
| `FloatMargin` | 16 | Margine orizzontale per la capsula (in pt) |
| `FloatCornerRadius` | -1 | Raggio di curvatura degli angoli della capsula (-1 = forma a "pillola", ovvero pari a metà dell'altezza) |
| `BlurStyle` | 59 | Stile dell'effetto sfocatura (59 = `SystemThinMaterialDark`) |
| `Gloss` | 0.7 | Intensità del riflesso speculare (0–1) |
| `TargetClasses` | Predefinito | Lista consentita (whitelist) di nomi di classi specifici (**unione**: le classi predefinite, come `AWEFeedTopBarContainer` e `AWENormalModeTabBar`, sono sempre attive; le voci personalizzate vengono aggiunte a queste). |
| `TargetSubstrings` | Predefinito | Sottostringhe utilizzate per dedurre i nomi delle classi dei pulsanti sul lato destro (**unione**; efficace solo se l'elemento è di piccole dimensioni e contiene un'icona). |
| `ExcludedClasses` | Predefinito | Lista di esclusione (**unione**: `AWEFeedViewCell` e `AWEGradientView` sono sempre escluse). |

## Ambito e limitazioni (Importante: leggere attentamente)

| Funzionalità | Supportata | Note |
|---|---|---|
| Effetto vetro smerigliato (sfocatura dello sfondo in tempo reale) | ✅ | Utilizza `UIVisualEffectView` (stabile all'interno delle applicazioni) |
| Gradienti speculari / Angoli arrotondati / Base in vetro colorato / Elemento flottante | ✅ | Riflessi speculari renderizzati ad hoc + effetti di illuminazione sui bordi | + Ricostruzione dell'interfaccia utente (UI) flottante |
| Rifrazione / Dispersione (effetto distintivo "Liquid Glass" di iOS 26) | ❌ | Questo si basa su shader Metal personalizzati all'interno del servizio di rendering `backboardd`; non è accessibile dall'interno del processo di un'applicazione. |

In sintesi: il risultato è uno stile "vetro liquido smerigliato" (*frosted liquid glass*), anziché una rifrazione o dispersione calcolata per singolo pixel. Si tratta del massimo effetto visivo ottenibile dall'interno del processo di un'applicazione.

## Struttura delle directory

```
DouyinLiquidGlass/
├── Tweak.x                  # Punto di ingresso principale: rilevamento leggero tramite `didMoveToWindow` + corrispondenza dei percorsi (senza dipendenze da Substrate)
├── DYGlassView.h/.m         # Vista in vetro (`UIVisualEffectView` con finitura smerigliata, riflessi disegnati su misura e base colorata)
├── DYGlassInjector.h/.m     # Iniezione, rimozione dello sfondo, ricostruzione dell'interfaccia flottante, hooking mirato, etichettatura delle classi e logica di rimozione
├── DYPrefsSupport.h/.m      # Caricamento delle preferenze, memorizzazione nella cache dei risultati e ricaricamento dinamico (*hot-reloading*)
├── Makefile / control / DouyinLiquidGlass.plist
└── Prefs/                   # Pannello delle impostazioni (tramite PreferenceLoader; compilazione attualmente rimandata)
```

## Compilazione (Basata su cloud tramite GitHub Actions; non è richiesto un Mac)

Questo progetto utilizza il file `.github/workflows/build.yml` per la compilazione sui runner macOS di GitHub.
Genera automaticamente pacchetti `.deb` sia in modalità "rootful" che "rootless" (con supporto per le architetture `arm64` e l'ABI `arm64e`, compatibili con Dopamine).

Una volta effettuato il push o il caricamento del codice nel repository, GitHub Actions avvierà la compilazione. Gli artefatti risultanti possono essere scaricati in fondo alla pagina di esecuzione del workflow, sotto la voce **Artifacts → deb**.

## Installazione (Scegliere uno dei seguenti metodi)

**Metodo A: Installazione tramite Dopamine (Sileo)**
1. Scaricare `deb.zip` ed estrarre il file `.deb`. 2. Trasferisci il file sul tuo dispositivo, individualo tramite Filza e toccalo per installarlo (usando Sileo).
3. Esegui un "respring" del dispositivo, chiudi forzatamente l'app Douyin e riavviala. **Metodo B: Iniezione di un file dylib tramite TrollStore / TrollFools**
1. Ottieni il file `DouyinLiquidGlass.dylib` (estraendolo dal pacchetto `.deb` o recuperandolo come artefatto di build autonomo dalla CI).
2. Utilizza TrollFools per iniettare il file dylib nell'app Douyin già installata.
3. Chiudi forzatamente l'app Douyin e riavviala.

> Questo plugin non dipende da Substrate; pertanto, il Metodo B (esecuzione senza jailbreak) è pienamente supportato.

## Logica di corrispondenza (pronta all'uso e configurabile)

1. **Elenco di esclusione** (`ExcludedClasses`) — Ha la priorità massima: se viene rilevata una corrispondenza, l'elemento viene ignorato.
2. **Nomi di classe esatti** (`TargetClasses`) — Applica l'effetto "glass" (vetro) in caso di corrispondenza esatta.
3. **Sottostringhe dei nomi di classe** (`TargetSubstrings`) — Include valori predefiniti come `TabBar`, `TabBars`, `BottomBar`, `BottomNav`, `MainTabBar`, `ToolBar` e `BottomTab`; l'effetto viene applicato se viene rilevata una corrispondenza.
4. **Euristiche per la barra inferiore** (`Heuristics`) — Identifica automaticamente gli elementi con una larghezza superiore a 200pt, un rapporto larghezza-altezza maggiore di 2:1 e posizionati entro 60pt dal bordo inferiore dello schermo.

La valutazione dei nomi di classe si avvale di un **sistema di caching a tre livelli**, che garantisce prestazioni fluide anche quando le migliaia di viste (view) di Douyin sono soggette a frequenti aggiornamenti del layout.

## Configurazione (non richiede ricompilazione)

Sebbene non sia ancora disponibile un'interfaccia dedicata per le impostazioni, tutti i parametri possono essere regolati tramite un file `plist`, che supporta il ricaricamento in tempo reale (hot-reloading). Modifica tramite Filza:

```
/var/mobile/Library/Preferences/com.dy.liquidglass.plist
```

Chiavi comuni:
- `Enabled` (BOOL, predefinito: YES)
- `Debug` (BOOL, predefinito: YES; registra `[DouyinLiquidGlass] 套玻璃: [class name]` quando viene applicato l'effetto vetro)
- `FilterType` (predefinito: `systemMaterialBlur`; su iOS 15 viene utilizzato come fallback `gaussianBlur`)
- `Blur` (raggio gaussiano; 0 = valore predefinito di 25)
- `Gloss` (intensità del riflesso, 0–1; predefinito: 0,7)
- `Quality` (qualità di cattura; predefinito: 1)
- `CornerRadius` (raggio degli angoli; <0 segue il raggio dell'elemento di destinazione)
- `Heuristics` (rilevamento basato su euristiche; predefinito: YES)
- `TargetClasses` / `TargetSubstrings` / `ExcludedClasses` (array)

Le modifiche diventano effettive immediatamente dopo il salvataggio (oppure eseguendo `notifyutil -p com.dy.liquidglass/Reload`); non è necessario riavviare l'app Douyin.

## FAQ

- **Un elemento non presenta l'effetto vetro:** Controlla i log cercando `[DouyinLiquidGlass] 套玻璃: XXX`. Se non è presente, significa che il nome della classe non è stato rilevato; aggiungi il nome della classe a `TargetSubstrings`.
- **Lo sfondo di un elemento è stato rimosso erroneamente o coperto dall'effetto vetro:** Aggiungi il nome della relativa classe a `ExcludedClasses`.
- **Servono nomi di classi più precisi per i pulsanti:** Usa `class-dump` per estrarre i nomi delle classi dalla versione di Douyin attualmente installata, quindi aggiungili a `TargetSubstrings`.
- **Vuoi ripristinare il pannello delle impostazioni:** Rimuovi il commento da `#SUBPROJECTS += Prefs` nel `Makefile` principale ed effettua nuovamente la compilazione.