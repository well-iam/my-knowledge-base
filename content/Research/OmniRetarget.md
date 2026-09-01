Paper: OmniRetarget: Interaction-Preserving Data Generation for Humanoid Whole-Body Loco-Manipulation and Scene Interaction
arXiv: https://arxiv.org/abs/2509.26633 (Amazon FAR / MIT / UC Berkeley / Stanford / CMU)
Contesto: materiale di lettura richiesto per il colloquio Research Engineer @ KTH RPL (Quantao Yang / Anna Deichler)

## Research Problem
Come generare dati di riferimento (kinematic reference motions) di alta qualità per addestrare policy RL su humanoid robot capaci di loco-manipolazione complessa — muoversi e allo stesso tempo interagire con oggetti e terreno (portare scatole, arrampicarsi, superare pendii) — partendo da dati di movimento umano, senza introdurre artefatti fisicamente implausibili (piedi che scivolano, compenetrazioni con oggetti/terreno).

## Existing Research
Due filoni principali:
- **Teleoperazione** (HumanPlus, OmniH2O, TWIST): un operatore umano controlla il robot online. Feedback in tempo reale, ma labor-intensive, non scala, affaticamento dell'operatore, mancanza di feedback aptico.
- **Retargeting offline** (PHC, GMR, VideoMimic): adatta cinematicamente il movimento umano al robot senza operatore in loop. Più scalabile, ma i metodi esistenti si basano su semplice *keypoint matching* con ottimizzazione non vincolata o penalità soft — risultato: foot-skating, compenetrazioni, nessuna consapevolezza esplicita di oggetti/scena.

## Research Gap
- Nessun metodo esistente preserva esplicitamente le relazioni spaziali/di contatto (corpo-oggetto-terreno) durante il retargeting *mentre* applica vincoli fisici hard (non solo soft penalty).
  [L'unico lavoro concettualmente vicino (IMMA, 2012) preserva relazioni spaziali tra parti del corpo ma non è open-source, ignora limiti cinematici e non considera oggetti/ambiente.]
- Inoltre, generare varianti dello stesso task (oggetti/terreni diversi) richiede oggi nuove demo da zero — costoso, non scalabile.

## Proposed Direction
**OmniRetarget**: costruisce una *interaction mesh* (tetraedralizzazione di Delaunay) i cui vertici includono giunti chiave del corpo **più** punti campionati sull'oggetto manipolato e sul terreno (campionati più densamente per preservare meglio il contatto). Il retargeting minimizza la deformazione Laplaciana di questa mesh tra sorgente (umano) e target (robot), sotto vincoli hard espliciti:
- no compenetrazione (collision avoidance)
- limiti articolari e di velocità
- no foot-skating (vincolo di stance foot fisso)

Risolto frame-by-frame con un solver SQP-style (sequential SOCP), warm-started dal frame precedente, con auto-differenziazione via Drake per gestire correttamente la geometria dei quaternioni.

**Data augmentation "gratuita"**: una volta costruita la interaction mesh, si può perturbare posizione/orientamento/scala dell'oggetto o l'altezza del terreno e ri-risolvere l'ottimizzazione — ottenendo varianti fisicamente plausibili da un'unica demo umana. Per evitare che l'ottimizzazione collassi in una trasformazione rigida banale, si aggiungono termini che ancorano parti del corpo (es. gambe) alla traiettoria nominale, forzando il resto (es. braccia) a scoprire nuove coordinazioni.

## Research Significance
Con reference motion pulite (quasi zero compenetrazione/foot-skating), il training RL a valle diventa drasticamente più semplice: bastano 5 reward term e 4 termini di domain randomization (contro il pesante reward engineering richiesto dai metodi precedenti per compensare gli artefatti), ottenendo comunque zero-shot sim-to-real transfer su hardware reale (Unitree G1) — inclusi task complessi come parkour, arrampicata su piattaforme (0.9m, 70% dell'altezza del robot) e un wall-flip a 15 rad/s. 
In sintesi: sposta il problema "a monte" — invece di curare gli artefatti con reward hacking, li elimina alla fonte con una generazione dati più rigorosa. Risultati quantitativi: +10% success rate RL rispetto ai baseline, con varianza minore (training più stabile).

## Perché i dati sono necessari
Il vero collo di bottiglia della RL su humanoid non è l'algoritmo, è il **reward engineering**: l'RL esplora in modo efficiente solo se il reward è ben progettato, e su humanoid (spazi d'azione ad alta dimensionalità, dinamica complessa) progettare reward a mano per movimenti naturali ed espressivi è difficile e fragile. Imitare movimenti umani offre un reward "gratuito" e denso (position/orientation tracking verso una reference) — ma **solo se la reference è di buona qualità**. Se il retargeting produce artefatti (piedi che affondano nel pavimento, mani che compenetrano l'oggetto), la RL deve o (a) correggere questi artefatti con reward ad-hoc extra (fragile, richiede tuning continuo), o (b) i dati vanno puliti manualmente (costoso, non scala). Da qui il valore di OmniRetarget: elimina il problema alla radice, producendo reference già pulite, così il reward RL resta minimale.
In più, raccogliere fisicamente ogni variante di un task (stessa azione, oggetto diverso, terreno diverso) via teleoperazione o mocap richiederebbe una sessione di raccolta dati per ogni variante — con la data augmentation di OmniRetarget, una singola demo umana genera decine di varianti "gratis", il che è ciò che rende il metodo scalabile.

## Cosa fa effettivamente la policy RL (chiarimento importante)
La policy RL **non impara a fare il retargeting** — il retargeting è un passaggio completamente separato, offline, risolto come ottimizzazione vincolata (non è appreso, è un solver numerico classico, lo stesso per ogni nuova demo).

Quello che la RL impara è il passo successivo: **trasformare una traiettoria cinematica di riferimento (posizioni articolari desiderate nel tempo) in una policy di controllo fisicamente realizzabile** — cioè le azioni (target di posizione articolare / coppie) che il robot deve eseguire, frame dopo frame, per seguire quella traiettoria tenendo conto della sua dinamica reale: inerzia, forze di contatto, attrito, disturbi esterni, limiti degli attuatori. In pratica, colma il gap tra "cinematica pulita" (cosa vorremmo che il robot facesse, geometricamente) e "dinamica fisica" (cosa il robot può effettivamente fare data la sua massa, i suoi motori, il contatto col mondo).

Concretamente, la policy osserva: la reference motion (posizione/velocità articolare di riferimento, errore di posizione/orientamento del pelvis), la propriocezione del robot (velocità lineare/angolare del pelvis, posizione/velocità articolare reale), e l'azione precedente. Il reward premia quanto bene il robot sta tracciando la reference (DeepMimic-style tracking), penalizza cambi bruschi di azione, violazioni dei limiti articolari, e self-collision. Non c'è nessuna informazione esplicita su scena/oggetto nell'osservazione — la policy è "cieca" e deve fidarsi della reference motion, che è per questo che la qualità del retargeting è così critica: se la reference è già corretta e priva di artefatti, alla policy basta imparare il *tracking* dinamico, non deve anche "correggere" errori cinematici a monte.

Questo è anche il motivo per cui serve lo zero-shot sim-to-real transfer: la policy, allenata in simulazione con domain randomization (massa/inerzia/forma dell'oggetto, disturbi random, rumore nei sensori), generalizza a variazioni della dinamica reale senza essere mai stata addestrata direttamente sul robot fisico.

## Confronto con la teleoperazione (per il colloquio)
La teleoperazione dà feedback online e non richiede retargeting automatico, ma è labor-intensive e scala male — un operatore per ogni demo, affaticamento, difficoltà a stabilizzare movimenti estremi (squat). Il retargeting offline (la direzione del gruppo KTH RPL) è più scalabile: adatta l'intera demo — movimento e interazioni con la scena — una volta sola, offline, invece di richiedere un operatore per ogni variante. Il mio lavoro sul G1 a KTH usava IL da teleoperazione VR (dati diretti, costosi da raccogliere); questa posizione si muove verso retargeting da mocap (più scalabile, richiede risolvere il gap cinematico uomo-robot preservando i contatti) — è il punto di congiunzione naturale tra il mio background in WBC/tracking e la loro direzione.

## Collegamento con la ricerca di Anna Deichler
Non c'è overlap metodologico diretto (loco-manipulation via retargeting mocap ≠ comunicazione referenziale), ma c'è un allineamento tematico su due livelli:

1. **Stesso problema di fondo, segnali diversi.** OmniRetarget: come generare movimento robotico plausibile a partire da un segnale umano (mocap), preservando le relazioni spaziali/di contatto che l'adattamento all'embodiment del robot rischierebbe di perdere. Il lavoro di Anna su grounded gesture generation affronta una versione linguistico-spaziale dello stesso problema: generare movimento comunicativo (gesti) a partire da un segnale umano (linguaggio + contesto 3D), preservando il riferimento spaziale corretto. Cambia cosa va preservato (contatto fisico vs. riferimento comunicativo), non la struttura del problema.

2. **Stesso collo di bottiglia sui dati.** La motivazione di OmniRetarget (dati scarsi/costosi per interazioni robot-oggetto-terreno) è lo stesso ostacolo che affligge la gesture generation grounded (dataset linguaggio-gesto-riferimento spaziale annotati sono scarsi). L'approccio di augmentation di OmniRetarget (una demo → molte varianti valide via re-ottimizzazione) è concettualmente il tipo di soluzione che potrebbe generalizzare a quel dominio.

**Punto esplicito fatto da Anna**: "making robotic task execution involve humans is one of the goals of current projects" — probabilmente la traiettoria è un humanoid che esegue task fisici (ciò che abilita OmniRetarget) *e* allo stesso tempo comunica/referenzia con un umano nello spazio condiviso durante l'esecuzione (il suo filone) — es. il robot che indica cosa sta per afferrare, o risponde a un riferimento spaziale umano mentre esegue loco-manipolazione. L'interaction mesh di OmniRetarget potrebbe in linea di principio estendersi per preservare anche vettori comunicativi (sguardo, direzione del gesto verso l'oggetto referenziato), non solo il contatto fisico.

Utile per rispondere in modo genuino a una domanda su "research interest" al colloquio: non un overlap diretto, ma un punto di incontro plausibile tra le due direzioni in un progetto futuro.

## Punti da avere pronti al colloquio
- Confronto critico: dove la teleoperazione resta comunque utile (feedback online, task di precisione) vs dove vince il retargeting (volume di dati, varietà di scenari)
- Come collegherei il mio lavoro WBC/diffusion policy sul G1 con una pipeline stile OmniRetarget: usare le reference motion pulite generate da OmniRetarget come input per il layer di tracking/control che ho già costruito
- Differenza netta tra "retargeting" (ottimizzazione cinematica offline) e "training RL" (apprendimento del tracking dinamico) — non confonderli in risposta
- Se emerge la domanda su research interest/continuare la ricerca: il collegamento tra loco-manipolation (task fisico) e comunicazione referenziale (Anna) come possibile traiettoria futura del gruppo