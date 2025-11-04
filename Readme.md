## Formål med datakonkurransen for Avinor
- Lage en predikasjonsmodell som analyserer når AFIS-fullmektig opplever mest trafikk/arebidsbelastning.
- En modell som predikerer sannsynlighet for samtidighet. Slik at AFIS-fullmektig kan kommunisere med flere fly samtidig. 
- Viktig for Avinor å kunne ha en forklarbar modell av regulatoriske hensyn. 
- Samtidighet per flyplass per time i perioden 01.10.25-31.10.25

## Loggføring
- Understand the data
- Build a baseline
- Build the first ML model
- Validate and compare 
- Package report

1. Begreper
- airport_group, date, hour skal være unike i både trening og predikasjonsmodellene.
        - Må sjekkes for duplikater
        - Samme format for alle dato
        - Levere 1 rad per nøkkel. 

### Hva er nøkkelen i datasettene, og er den unik?
- Nøklene for observasjonene er unik i både trenings og malfilen. Dette betyr at hver kombinasjon av gruppe dato og time representerer en og en observasjon.

Dette er viktig, hvis de ikke er unike så må man velge ut hvilke rad som gjelder, og det kan forvrenge både treningsdata og predikasjonsformat.

### Hvilke kolonner er kun i train, og hvorfor?
- Kolonnen target finnes kun i trenings datasettet fordi den representerer fasiten om den faktisk var samtidighet i den timen.

Den mangler i interference fordi det er data som vi får for å kunne predikere sannsynligheten for.

### Hvorfor trenger vi preds_mal.csv?
- Det er det vi skal levere inn. Den fungerer som en mal av nøykatig kombinasjoner av kolonnene. Samt fyller inn kolonnen pred i denne malen.


### 2. Lage baseline
- En baseline er et enkelt, logisk system som du kan lage predikasjoner uten maskinlæring.
- Den forteller om det er et signal i dataen, som for eksempel faste mønstre per time/dag, eller om du kan forbedre resultatene med mer avanserte metoder.

Mentalt bilde:
- Du skal spå om fremtidige flyplasser har samtidige flybevegelser, men er ikke det bare å se på hvordan det har vært tidligere?

- Det er en bra baseline, du har stabilitet over tid, men nå skal du prøve å slå denne baselinen.

## Baseline 1 – Group-Hour
- Beregner gjennomsnittlig target per (airport_group, hour)
- Bruker historikk (train) for å spå framtid (inference)
- Evalueres på siste måned i treningsdata
- Metrikker: AUC, Brier score
- Fallback: hvis manglende kombinasjon i infer → bruk global mean(target)

## Baseline 2 – Group-Weekday-Hour
- Samme som over, men inkluderer dag i uken (dow)
- Tester om trafikkrytme i uka påvirker samtidighet

## Før jeg lager kode uteifra Pseudo-code
- Hva gjør baseline A og B?
    - svar: Baseline A bruker tidligere historikk gjennomsnittet av target per (airport_group, hour) for å kunne lage en sannsynlighet.

    Baseline B utvider baseline A ved også å bruke ukedager, altså gjennomsnittet per (airport_group, dow, hour) for å få mer nøyaktig svar. 

- Hvor ville det ha vært lekkasje?
    - svar: Å blande den siste måneden (valideringsperioden) inn i rate gjennomsnittene eller å bruke informasjon fra inference-settet når modellparametere beregnes. 

- Definere fall-back hieraki(g, d, h -> g, h -> global)
    - svar: Først (group,dow, hour), deretter (group, hour) og til slutt global gjennomsnittsrate for å sikre prediksjon i alle noder. 
    
- Jeg vet hvilke metrikker jeg må rapportere (AUC, Brier, per-gruppe AUC.)
    - svar: total AUC, total Brier score og per-gruppe AUC for robusthetskontroll.

- Jeg skriver først funksjoner for rate-trening og rate-oppslag.
    - svar: en for å trene rater fra hisotrikk og en for å slå opp rater med fallback-logiken.


## 3. Lage maskinlæringsmodellen
Nå som preds baseline modellen er laget så skal jeg lage en feature plan og trenings og validerings plan. Vi skal lage en plan slik at maskinlæringsmodellen til å predikere målet bedre enn baslinenen.

### No cheating! No data leakage. 

- Jeg skal starte med logistikk regresjon. 
    - Etter det starte med tree-based models som HistGradingBoosting, LightGBM eller XGBoost for ikke-linære mønstre.

- Bruke samme split logikk som for baseline.

- Lage en feature list:
Feature name: Description: Why it's safe:

Hour: Timen av dagen - Komemr fra timestamp uten å lekke. 

Dow: Day of the week. - Fra data, no leakage

is_weekend: 1 hvis lørdag elelr søndag - fra dow

feat_sched_flights_cnt: Planned number of flights- Comes from schedule, not target

feat_sched_concurrence:	Planned concurrency level - Also from schedule

feat_sched_flights_cnt_lag1: 	Previous hour’s planned flights (within same group) - Uses only past values

feat_sched_concurrence_lag1: Previous hour’s planned concurrency - Same reason

rate_group_hour: Average target for that group+hour (from baseline table) - Computed from older data only

rate_group_dow_hour: Average target for that group+dow+hour (from baseline table) - Same


Data preparation:
- pd.to_datetime må alltid være hour er integer.
- Sortere airport_group, date og hour før lage lag features.
- Fill na

Train and validate:
- Train LR on training set

- Evaluate on validation set → record AUC and Brier

- Train HGB on training set

- Evaluate on validation set → record AUC and Brier

Calibrate the tree model

- Calibration just makes the probabilities more realistic.

Inference (Future prediction)
Når du har testet nok og stoler på modellen: 
- Lage samme features for inference_data_oct2025.csv.

- Predikere med den endelige modelen. 

- Clip to [0,1], round to 3 decimals.

- Merge into preds_mal.csv → pred column.

- Save as your final submission.


### Rapport
Outline:
- Problem + mål

- Data (kort)

- Features (med anti-leakage-notat)

- Validering (spesifikk: hvilke måneder hvor)

- Resultater (AUC/Brier totalt + per gruppe, 2–3 plott)

- Kalibrering (før/etter, 1 plott)

- Feilanalyse (3 viktigste innsikter + tiltak)

- Neste steg (konkret 3–5 punkter)


