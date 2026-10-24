# Il prompt di ritiro di una spec della serie v1

> Generato il 2026-10-24 da 08_CREA_RITIRO_SPEC. Chi lo usa dopo un ritiro aggiorna i precedenti e le
> misure e lascia il corpo com'è; il ritiro dell'ultima spec porta via questo file con la cartella.
>
> Uso: si copia il blocco qui sotto in una sessione nuova, compilando SOLO la prima riga. Ogni
> misura è scritta come il comando che la prende (`grep`, `wc`) oltre al valore del 2026-10-24,
> perché i valori invecchiano a ogni ritiro; le misure sul codice (cancelli, guide) sono solo
> comandi, perché al 2026-10-24 il repo non ha ancora una riga di codice (indice § 7.6, D-9).

```text
SPEC DA RITIRARE: doc/v1/NN-nome.md      <- l'unico punto da compilare

Ritira la spec scritta qui sopra. Nel resto di questo prompt <SPEC> è quel percorso, <NN>
il suo numero e <SLUG> il resto del nome senza «.md»: ricavali tu, non chiedermeli. Se il
file non esiste, FERMATI e dimmelo prima di fare qualsiasi cosa.

PREREQUISITO, da misurare e non da assumere: la spec è IN ESERCIZIO, cioè (indice § 7.5):
per 01, 02, 03, 04, 05a e 05b, che vivono prima del deploy, PR mergiata su `main` (`gh pr
view <N> --json state,mergedAt`, oppure `git log --first-parent main --format="%h %ad %s"
--date=short`) E il collaudo di § 7 della spec annotato in `CLAUDE.md § Current Status →
Latest` (D-37); per 06 e 07, PR mergiata su `main` E deploy di produzione su Vercel verde E
la misura di chiusura di § 8 della spec rifatta in produzione (D-56, 2026-10-24: per la 06
il cron lanciato a mano lascia in `cron_runs` una riga con `finished_at` non null; per la 07
«Invia notifica di prova» arriva su entrambi i telefoni e `push_subscriptions.last_success_at`
è valorizzato), letta nella SQL console di Turso e mai «a occhio»; gli esiti post-merge e
«in esercizio il <data>» si scrivono come primo passo della sessione successiva (D-38; per la
07 nel commit di ritiro stesso, 07 § 8: li misuri e li scrivi tu prima di ogni altra cosa).
La 06 rifà in produzione le misure di chiusura di 03–05b (06 § 7 D e F). Il primo sabato di
cron e la prima push «Revisione pronta» NON sono segnali (D-56). Il
caso reale NON è un prerequisito: dove è a portata di giorni lo si cerca e lo si scrive,
ma non lo si aspetta mai — ciò che l'uso vero insegnerà diventa feedback e poi una spec
nuova, e quando il caso arriva si annota nella riga di stato della spec ritirata, come
storia. Il collaudo della sessione di implementazione vale come prova della spec, non
come segnale. DDL: le migrazioni Drizzle sono file committati in `drizzle/` (`ls
drizzle/*.sql`: `0000_*` dalla 03, `0001_reviews.sql` dalla 04, `0002_push.sql` dalla 07;
`git log --format="%h %ad %s" --date=short -- drizzle/` dice quando sono nate); 01, 02, 05a
e 05b non portano DDL. Prima della 06 (ritiri di 03 e 04) il DDL è speso quando `pnpm
db:migrate` lo ha applicato a `local.db` e `src/lib/db/migrate.test.ts` passa con
`createMemoryDb()` sulle migrazioni reali (03 A1). Dalla 06 in produzione lo applica l'owner
dal PC con `pnpm db:migrate:remote` PRIMA del deploy che lo usa (06 § 3 a, 07 § 7 H), mai nel
build di Vercel; la prova è `SELECT COUNT(*) FROM __drizzle_migrations` nella SQL console di
Turso: 2 dopo la 06 (06 A4), 3 dopo la 07. Nessuna migrazione distruttiva in v1: se un
ritiro ne trova una, FERMATI (andrebbe spesa da sola, dopo il deploy, e nessuna spec la
scrive). Se un criterio non regge, un segnale manca o uno script non è stato
speso, FERMATI e dillo: una spec si ritira intera, mai a metà, e non si ritira una spec
la cui implementazione ha lasciato un «da decidere».

LETTURE OLTRE AL PROMPT STANDARD (WORKFLOW.md, AGENTS.md, CLAUDE.md, COMMENTS.md,
DEVELOPMENT_GUIDELINES.md, la guida del dominio): README § Documentazione · l'indice
della serie per intero · la spec INTERA, compresa l'intestazione con lo stato scritto
dalla sessione che l'ha implementata e le divergenze dichiarate lì · `git log
--format="%h %s" -- <SPEC>` · i precedenti da imitare: nessun ritiro in questo repo al
2026-10-24 (`git log --grep=ritir -i --format="%h %ad %s" --date=short` è vuoto: è la prima
serie). Da imitare, nel repo gemello `net-worth-tracker` accanto a questo (`git -C
../net-worth-tracker show --stat <hash>`): `ba9198c5` (2026-10-12, ritiro di MOB-04 con la
serie ancora viva: 205 criteri letti contro il codice, quattro difetti a runtime corretti
con il test visto rosso prima, le prove E2E promesse aggiunte), `f3dac84a` (2026-10-16,
MOB-07: la misura di chiusura era rossa sul codice mergiato senza modifiche, allargata e
scritta con il perché), `de8bafc9` (2026-10-17, ritiro di MOB-09 E della cartella
`doc/mobile/` con l'indice e il suo prompt di ritiro: 753 rimandi riscritti in 175 file, la
storia della serie in README). Il primo ritiro di hevy-coach diventa il precedente locale:
chi lo fa sostituisce queste righe con il suo hash e una riga sul perché. Ogni riferimento
nel codice a un documento ritirato si riscrive in loco perché il perché resti in piedi
da solo, tenendo le date.

ORDINE DI LAVORO

1) VERIFICA CRITERIO PER CRITERIO, prima di cancellare qualsiasi cosa: per ogni criterio
   di accettazione, esiste nel codice? e la prova è quella che il criterio nomina (il test
   con la sua prova di portanza, la misura del collaudo)? Le decisioni sono state
   rispettate una per una, e le scelte «ribaltabili» sono rimaste com'erano o il
   ribaltamento è scritto? Le divergenze dichiarate sono ancora vere? La documentazione
   della spec è stata fatta (guide, README, CLAUDE.md, i totali rimisurati)? E ogni
   RINVIO («la spec successiva lo eredita», «fuori ambito, dichiarato») ha un erede: il
   codice, una spec viva per esteso, l'indice o CLAUDE.md § Not yet present.
   ⚠️ La verifica si fa CON UN WORKFLOW, in sola lettura e tutto Opus: un agente per
   gruppo di criteri, uno per le decisioni e le scelte, uno per la documentazione e i
   rinvii, uno per i contratti condivisi; a ciascuno la lista e la richiesta di
   percorso:riga per ogni voce, mai una conferma; poi un verificatore per segnalazione.
   Sotto i dieci agenti, nessuno scrive file. I punti che reggono li rileggi in proprio
   prima di correggere; correzione, travaso e tutto ciò che segue li fai tu.
   ⚠️ Una divergenza si CORREGGE o si SCRIVE, mai si accetta in silenzio: è l'ultima
   occasione, dopo il testo sparisce.
   ⚠️ Un difetto trovato: prima la sua CLASSE con un grep su tutto il repo; ogni fix
   vuole il test che lo fissa, provato portante rimettendo il difetto.
   ⚠️ I CONTRATTI CONDIVISI: ciò che questa spec doveva creare per le successive esiste
   con QUEL nome e quella firma? Se no, si aggiornano l'indice e OGNI spec viva che lo
   cita (testo e prompt in fondo).

2) MISURA I RIFERIMENTI, non stimarli: `grep -rn --exclude-dir=node_modules
   --exclude-dir=.next -E "doc/v1|spec 0?<NN>|<SLUG>|<titolo a parole>" .` (il titolo a
   parole come sta nel `# ` della spec e nella colonna Titolo di § 5 dell'indice) e, per
   gli orfani, `grep -rn --exclude-dir=node_modules -E "\b(D|T|C)-[0-9]+\b" src scripts
   tests drizzle` (AGENTS.md § 2.2 e indice § 7.1: le etichette non entrano MAI nel
   codice, nei commenti né nei messaggi di commit, quindi questo grep deve essere vuoto
   sin dal primo giorno; se non lo è, è un difetto da correggere in loco scrivendo il
   perché per esteso). Le etichette D-N, T-N, C-N restano vive nei documenti (`doc/v1/` e,
   dove le citano, AGENTS.md, CLAUDE.md, WORKFLOW.md: `grep -n -E "\b(D|T|C)-[0-9]+\b"
   AGENTS.md CLAUDE.md WORKFLOW.md README.md`) finché vive l'indice; i contratti condivisi
   C-1…C-18 vivono nell'indice § 6 e si travasano nelle guide delle aree al ritiro
   dell'ultima spec. Tre categorie, tre trattamenti: CODICE,
   TEST e COMMENTI → riscritti in loco, veri e autosufficienti senza il documento, con
   le date; SPEC ANCORA VIVE → ripuntate al codice (percorso e nome verificati adesso),
   alla guida o alla regola dove la decisione ora vive; SPEC che la citano come
   appuntamento («dopo la NN») → lasciate.

3) TRAVASA ciò che è ancora operativo, prima di cancellare: il perché di una scelta,
   una decisione precedente RIBALTATA con la data, un limite dichiarato, una trappola
   misurata al collaudo. Un pezzo in UN posto solo: regole → AGENTS.md (§ 3 Hevy, § 4 Plan
   data, § 5 Database, § 6 Claude, § 7 Auth and app, § 8 Deploy and PWA, § 9 Testing, § 11
   Gotchas con la data; lo stub «(from spec NN — stub…)» dell'area diventa regole con il
   rimando alla guida); narrativa di dominio → `doc/guide/piano.md` (02), `sync.md` (03),
   `revisione.md` (04), `app.md` (05a/05b), `deploy.md` (06), `pwa.md` (07); design →
   DESIGN.md e PRODUCT.md (dalla 05a, scritti solo dalla skill impeccable, mai a mano);
   regole di sessione e comandi → WORKFLOW.md § 3; stato e «Not yet present» → CLAUDE.md;
   per l'utente → README.md e SETUP.md (dalla 06); il perché locale → commento di design in
   testa al modulo o why-comment sulla riga (COMMENTS.md); un'invariante provata → il test
   che la fissa (`*.test.ts` accanto al modulo, AGENTS.md § 9). Sostituire, mai accumulare.
   In CLAUDE.md una riga di modulo resta la domanda e il rimando: la data, il tipo nuovo,
   il dettaglio vanno nella guida, mai in coda alla riga (nell'Hub, 2026-10-12 → 2026-10-17,
   così CLAUDE.md è tornato da 20k a 27k caratteri).
   Il prompt, il modello e l'effort in fondo alla spec NON si travasano.

4) CANCELLA la spec (`git rm <SPEC>`): testo, criteri, stato e divergenze restano in
   `git log`, che si cita come archivio. L'indice RESTA finché vive un'altra spec della
   serie.
   ⚠️ SE È L'ULTIMA SPEC DELLA SERIE, si ritira la CARTELLA INTERA — l'indice, il README,
   i file di riferimento, questo stesso prompt — e PRIMA di cancellarla tutto ciò che vi
   vive ancora operativo si travasa nella documentazione del repo con le regole del
   passo 3 e del prompt di fine sessione: le decisioni e le regole della serie che
   valgono ancora → AGENTS.md (regola + data + dove è fissata) o la guida del dominio;
   una regola di sessione nata nella serie → WORKFLOW.md; lo stato → CLAUDE.md; la
   storia della serie, con le date, i master e che cosa ha corretto ogni ritiro → README
   § Documentazione (come per le serie precedenti); i contratti condivisi → le guide
   delle aree che li usano, per nome. Quel giorno le etichette della serie citate nel
   codice si riscrivono in loco con uno script di sostituzioni esatte che rispetta i
   fine riga per file, e una revisione a campione prima di fidarsi; `git log -- doc/v1/`
   resta l'archivio, e ogni rimando alla cartella nel repo si riscrive verso di lui.

5) AGGIORNA indici e stato — lo stato vive in PIÙ punti che possono divergere e si
   chiude con un grep, non con una riga: la serie è `doc/v1/`, l'indice
   `doc/v1/00-INDICE.md`; la riga di stato ha la forma `> **Stato**: scritta <data> · in
   implementazione il <data>, branch · implementata il <data>, PR #N · in esercizio il
   <data> · ritirata il <data>` (indice § 7.4) e vive nell'intestazione della spec, nella
   tabella § 5 dell'indice, in `CLAUDE.md § Current Status → Latest`, nella riga `> **Status
   (<data>)**` in testa a README.md (README § Documentation elenca solo l'indice) e, come
   appuntamento («Pronta per l'implementazione dopo la NN»), nell'intestazione delle spec
   sorelle. Ritirata la spec, la sua riga di stato sopravvive in § 5 dell'indice con
   «ritirata il <data>» e in `git log`. Se era l'ultima spec, dillo
   esplicitamente e chiudi la serie come l'ultimo precedente.
   `grep -n "Stato" doc/v1/*.md` (intestazioni e § 5 dell'indice), `grep -n "Latest"
   CLAUDE.md` e `grep -n "Status" README.md` (la riga in testa) devono dire la stessa cosa
   ovunque: quale spec è ritirata e da quando, quale viene dopo, che cosa è in esercizio.

CANCELLI: `pnpm typecheck && pnpm lint && pnpm test && pnpm build` (WORKFLOW.md § 3,
dalla 01), poi `pnpm test:e2e` (dalla 05a), poi `pnpm test:e2e:pwa` (dalla 07, su build di
produzione); il build di verifica è `next build --webpack` (Serwist, T-4), mai `next build`
liscio; i test girano con `TZ=UTC`. Un cancello rosso ferma il ritiro. Nessuna chiamata
reale a Hevy né ad Anthropic in un ritiro (indice § 7.8), salvo nel collaudo dichiarato con
effetto (lettura o scrittura su Hevy, che è permanente) e costo (Anthropic) dopo l'OK.
LIMITI: CLAUDE.md < 40.000 caratteri, obiettivo 20.000, misura `node -e
"console.log([...require('fs').readFileSync('CLAUDE.md','utf8').replace(/\r/g,'')].length)"`
(il 2026-10-24: 7.277); AGENTS.md ≤ 1000 righe, obiettivo 600, misura `wc -l AGENTS.md` (il
2026-10-24: 149); `doc/guide/<tema>.md` ≤ 500 righe per l'indice § 7.3, ma ogni spec dichiara
in § 9 un tetto più stretto per la sua guida: piano.md 200, sync.md 200, revisione.md 300,
app.md 300 (la 05b la estende restando sotto 300), deploy.md 200, pwa.md 250; misura `wc -l
doc/guide/*.md` (il 2026-10-24 nessuna guida esiste: quale sia la più vicina al tetto lo dice
il comando il giorno del ritiro). Se sfori, NON tagliare in chiusura: dimmelo e proponi lo
scorporo come sessione a parte (prompt 05/06 di claude-config).

⚠️ NON: riassumere una spec dentro un documento che resta · lasciare un rimando a un
file che non esiste più · cancellare date · ritirare senza i segnali (e senza il DDL
speso) · ritirare l'indice finché una spec vive · cominciare la spec successiva prima di
aver ritirato questa · lasciare un dubbio: ogni domanda si fa SUBITO con AskUserQuestion.

Alla fine: riassumi il diff; le occorrenze riscritte per categoria; che cosa hai
travasato e dove; che cosa ha trovato la verifica; l'esito del grep dello stato; qual è
la prossima spec e se i suoi prerequisiti sono in piedi. Se hai corretto codice,
scrivilo anche in CLAUDE.md § Latest: i segnali passati valgono per il codice di PRIMA,
e sul nuovo sono da rifare dopo il deploy. Poi chiedimi l'OK per il commit: un branch
dal branch di integrazione, un solo commit.
```
