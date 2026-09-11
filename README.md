# CI for Spring Boot med GitHub Actions

I denne øvingen skal du:

* Generere et nytt **Spring Boot**-prosjekt med **Spring Initializr**
* Opprette et **eget GitHub-repo** og pushe koden dit
* Sette opp **branch protection** på `main`
* **Invitere en medstudent** som collaborator
* Øve på **Pull Request-flyten** (feature branch → PR → review → merge)
* Sette opp en **GitHub Actions workflow** som kjører **unit-tester** på hver PR og på hver push til `main`

Målet er ikke å lære Spring Boot i dybden – vi bruker det kun som et konkret prosjekt å bygge og teste. Fokuset er på **arbeidsflyten** rundt kode: branching, code review, og kontinuerlig integrasjon (CI).

## Læringsmål

Etter å ha fullført øvingen skal du kunne:

* Forklare hva **CI (Continuous Integration)** er og hvorfor det er nyttig
* Sette opp **branch protection rules** for å hindre direkte push til `main`
* Lage en **Pull Request**, be om review, og merge etter godkjenning
* Skrive en enkel **GitHub Actions workflow** som bygger og tester et Maven-prosjekt

## Viktig: Vi jobber med to Git-repoer i denne øvingen — hold dem adskilt!

Dette er en klassisk felle som *kommer* til å forvirre deg om du ikke er obs på det fra start. I løpet av øvingen jobber du med **to helt separate GitHub-repoer**, og de skal ligge i **to sidestilte mapper** — ikke den ene inne i den andre:

```
/workspaces/
├── ci-spring-boot/     ← Repo 1: din fork (bare for README + devcontainer)
└── ci-demo/            ← Repo 2: ditt nye Spring Boot-prosjekt
```

| # | Repo | Mappe | Hva ligger her? |
|---|------|-------|-----------------|
| 1 | **Din fork** (`ci-spring-boot`) | `/workspaces/ci-spring-boot/` | README-en du leser nå, `.devcontainer/`, og eksempel-workflowen. **Ikke push endringer hit.** |
| 2 | **Ditt nye Spring Boot-repo** (`ci-spring-boot-<initialer>`) | `/workspaces/ci-demo/` | Selve Spring Boot-prosjektet du lager i Del 1. **Det er her all koden din, PR-ene, branch protection og CI skal leve.** |

### Hvorfor må de være sidestilte, ikke nøstet?

Hvis du pakker ut Spring Boot-prosjektet **inne i** fork-en (f.eks. `/workspaces/ci-spring-boot/ci-demo/`) havner du i en felle:

* Git ser oppover i mappetreet etter en `.git/`. Hvis du glemmer `git init` i `ci-demo/`, vil `git status`, `git add`, `git commit`, `git remote -v` osv. treffe **fork-ens** git-repo — uten at du merker det. Du kan ende med å commite Spring Boot-koden inn i fork-en.
* `gh repo create --source=. --push` fra feil katalog vil legge til remote og pushe til feil sted.
* Selv med `git init` i undermappa er dette et nøstet git-repo, som er et minefelt (ekskluderinger, submodule-forvirring, feilklikk i IDE).

**Regelen:** hold repo 2 utenfor repo 1. Del 1 forteller deg eksakt hvor du skal legge det (`/workspaces/ci-demo/`).

### Sjekk hvor du er før du gjør noe med Git

```shell
pwd                # Hvilken mappe står jeg i?
git remote -v      # Hvilket GitHub-repo peker denne mappa på?
```

Ser du `github.com:<lærer>/ci-spring-boot` i output, er du i fork-en. Ser du `github.com:<deg>/ci-spring-boot-<initialer>`, er du i ditt eget repo. Ser du **ingenting** i `git remote -v` fra `ci-demo/` før du har lagt til remote — det er som forventet.

## Lag en fork

### Hva er en fork?

En **fork** er en personlig kopi av et GitHub-repo som ligger på **din** GitHub-konto. Den er en fullverdig, uavhengig kopi — du kan pushe til den, opprette branches, lage Pull Requests og gi andre tilgang, uten at det påvirker det opprinnelige (upstream) repoet på noen måte.

* En **fork** er noe GitHub lager for deg med ett klikk. Den lever på GitHub-serverne.
* En **klone** er en lokal kopi på din maskin (eller i et Codespace), laget med `git clone`. Klonen henter innhold fra ett bestemt repo — enten upstream eller din fork.

Vi bruker en fork her fordi du trenger et repo du **eier**, slik at du kan starte en Codespace fra det, endre filer, og få tilgang til devcontaineren i denne øvingen — uten å påvirke lærerens original.

### Slik gjør du det

Gå til dette repoet på GitHub og klikk **Fork**-knappen øverst til høyre. Velg din egen konto som destinasjon. Etter noen sekunder har du en identisk kopi liggende under `github.com/<ditt-brukernavn>/ci-spring-boot`.

## Start et Codespace

I din fork, velg den grønne knappen "<> Code", og "Create codespace on main".

Dette repoet har en **devcontainer** (se `.devcontainer/devcontainer.json`) som gjør at Codespaces automatisk installerer alt du trenger:

* **JDK 21** (Temurin) – for å kompilere og kjøre Java-koden
* **Maven** – byggeverktøyet vi bruker for å bygge Spring Boot-prosjektet
* **GitHub CLI (`gh`)** – kommandolinjeverktøy for GitHub. Vi bruker det til å autentisere Git mot GitHub og til å opprette repo direkte fra terminalen (se Del 2).

Første gang Codespacet startes tar det noen minutter å bygge miljøet.

### Verifiser at verktøyene er tilgjengelige

Sjekk at alt er på plass ved å kjøre følgende i terminalen:

```shell
java -version
mvn -version
gh --version
```

Du skal få et versjonsnummer tilbake for alle tre. Får du "command not found" er devcontaineren ikke ferdig bygget, eller bygget feilet – sjekk loggen fra "Codespaces: View Creation Log".

## Del 1 – Generer prosjektet med Spring Initializr

Gå til **Spring Initializr**: [https://start.spring.io](https://start.spring.io) og velg:

* **Project**: Maven
* **Language**: Java
* **Spring Boot**: siste stabile versjon (unngå SNAPSHOT/RC)
* **Group**: `no.kristiania`
* **Artifact**: `ci-demo`
* **Packaging**: Jar
* **Java**: 21 (eller den versjonen som er tilgjengelig i Codespaces)
* **Dependencies**: `Spring Web`

Klikk **Generate** og last ned zip-fila.

### Legg prosjektet ved siden av fork-en (ikke inni!)

Se advarselen om to git-repoer over. Prosjektet skal legges i `/workspaces/ci-demo/`, **sidestilt** med `/workspaces/ci-spring-boot/` — ikke inne i fork-en.

I Codespaces-terminalen:

```shell
cd /workspaces
curl https://start.spring.io/starter.zip \
  -d type=maven-project -d language=java \
  -d groupId=no.kristiania.cidemo -d artifactId=ci-demo \
  -d name=ci-demo -d packageName=no.kristiania.cidemo \
  -d javaVersion=21 -d dependencies=web \
  -o ci-demo.zip
unzip ci-demo.zip -d ci-demo
cd ci-demo
```

> Uten `bootVersion` bruker Initializr default (siste stabile). Hvis du vil pinne versjon, se `https://start.spring.io/metadata/client` for aktuelle valg.

Har du lastet ned zip-fila via nettleseren i stedet, drag'n'drop den inn i Codespace-vinduet — eller last opp til `/workspaces/` og pakk ut der. **Ikke** la den havne i `/workspaces/ci-spring-boot/`.

### Verifiser at du står i riktig mappe

```shell
pwd
# skal si: /workspaces/ci-demo

ls -la
# skal vise pom.xml, mvnw, src/  — men INGEN .git/ (kommer i Del 2)

git rev-parse --show-toplevel 2>&1
# skal si: "fatal: not a git repository" — det er riktig, vi lager repoet i Del 2.
# Sier den derimot "/workspaces/ci-spring-boot", har du havnet inne i fork-en. Flytt prosjektet ut før du fortsetter.
```

### Verifiser at prosjektet bygger

```shell
./mvnw test
```

Du skal få en grønn build med minst én test (`contextLoads`) som passerer.

## Del 2 – Opprett et nytt GitHub-repo

### Autentisering mot GitHub (viktig!)

Før du kan pushe kode, må Git kunne bevise til GitHub at det er *deg* som pusher. I et helt ferskt Codespace har du **ikke** SSH-nøkler satt opp, og passord-innlogging over HTTPS ble deaktivert av GitHub for flere år siden. Derfor må du bruke ett av alternativene under.

**Anbefalt (Codespaces og lokalt): bruk `gh` CLI**

`gh` (GitHub CLI) er forhåndsinstallert i Codespaces og på de fleste utviklermaskiner.

```shell
gh auth login
```

Velg:
* **GitHub.com**
* **HTTPS** som protokoll
* **Yes** når den spør om å bruke `gh` som Git credential helper
* **Login with a web browser** — kopier engangskoden og lim inn i nettleseren

Etter dette har Git en credential helper som automatisk sender en gyldig token ved hver `git push` — du slipper å taste noe mer.

**Alternativ: Personal Access Token (PAT)**

Hvis du foretrekker å ikke bruke `gh`:

1. Gå til [github.com/settings/tokens](https://github.com/settings/tokens) → **Generate new token (classic)**.
2. Gi den scope `repo`.
3. Kopier tokenet (du får se det bare én gang).
4. Ved neste `git push` bruker du **brukernavnet ditt** og **tokenet i stedet for passord**.
5. Slå på credential-cache så du slipper å taste den hver gang:

   ```shell
   git config --global credential.helper store   # lagrer i klartekst i ~/.git-credentials
   # eller på Mac:
   git config --global credential.helper osxkeychain
   ```

> **NB:** SSH (`git@github.com:...`) fungerer også — men krever at du har generert et SSH-nøkkelpar (`ssh-keygen`) og lagt den offentlige nøkkelen inn på GitHub-kontoen din under `Settings → SSH and GPG keys`. Dette er ikke satt opp i et ferskt Codespace.

### Sett git-identiteten din (bare første gang)

```shell
git config --global user.name  "Ola Nordmann"
git config --global user.email "ola@example.com"
```

### Sørg for at Git bruker HTTPS (viktig!)

Codespaces har **ingen SSH-nøkkel** installert. Hvis du havner på en `git@github.com:...`-URL vil `git push` feile med `Permission denied (publickey)`. For å unngå dette:

```shell
gh config set git_protocol https
```

Dette gjør at `gh repo create` (og andre `gh`-kommandoer som setter opp remote) alltid bruker HTTPS-URLer. Kjør kommandoen én gang før du oppretter repoet under.

### Opprett repoet og push

Du har to måter å gjøre dette på. **Alternativ A** er raskest hvis du bruker `gh`.

**Alternativ A — la `gh` opprette og pushe i én kommando:**

Fra rot-mappa av Spring-prosjektet:

```shell
git init
git add .
git commit -m "Initial commit from Spring Initializr"
git branch -M main
gh repo create ci-spring-boot-<initialer> --public --source=. --push
```

Kommandoen oppretter repoet på GitHub-kontoen din, setter det som `origin` (med HTTPS-URL siden vi konfigurerte det over), og pusher `main` — alt i én sving.

**Alternativ B — opprett manuelt via web:**

1. Gå til [github.com/new](https://github.com/new) og opprett et **tomt** repo (ikke huk av for README, .gitignore eller lisens – prosjektet fra Initializr har allerede dette).
2. Kall det `ci-spring-boot-<dine initialer>`.
3. På repo-siden, **kopier `HTTPS`-URLen** (ikke SSH). Den starter med `https://github.com/...`.
4. Pushe eksisterende prosjekt:

```shell
git init
git add .
git commit -m "Initial commit from Spring Initializr"
git branch -M main
git remote add origin https://github.com/<ditt-brukernavn>/ci-spring-boot-<initialer>.git
git push -u origin main
```

Første `push` vil trigge credential helperen (`gh` eller keychain). Har du satt opp `gh auth login`, går den rett gjennom.

## Del 3 – Sett opp branch protection på `main`

Vi vil hindre at noen (inkludert deg selv) pusher direkte til `main` uten en Pull Request og en godkjent review.

1. Gå til `Settings` → `Branches` i repoet ditt.
2. Under **Branch protection rules**, klikk **Add rule** (eller **Add branch ruleset** i nyere UI).
3. Sett **Branch name pattern** til `main`.
4. Huk av for følgende:
   * **Require a pull request before merging**
     * **Require approvals** – minimum 1
   * **Require status checks to pass before merging** (vi legger til selve sjekken i Del 5)
   * **Do not allow bypassing the above settings** (valgfritt, men anbefalt)
5. Lagre.

Prøv å pushe direkte til `main` etterpå – det skal feile med en melding om at branchen er beskyttet.

## Del 4 – Inviter en medstudent og øv på Pull Requests

### Inviter en collaborator

1. `Settings` → `Collaborators` → **Add people**.
2. Skriv inn GitHub-brukernavnet til medstudenten din.
3. Medstudenten må akseptere invitasjonen fra e-post eller varsler på GitHub.

Gjør det samme andre veien – la medstudenten invitere deg til sitt repo.

### Lag din første Pull Request

1. Opprett en feature branch:

   ```shell
   git checkout -b feature/hello-endpoint
   ```

2. Legg til en enkel REST-endpoint i prosjektet. For eksempel en ny klasse `HelloController.java`:

   ```java
   package no.kristiania.cidemo;

   import org.springframework.web.bind.annotation.GetMapping;
   import org.springframework.web.bind.annotation.RestController;

   @RestController
   public class HelloController {
       @GetMapping("/hello")
       public String hello() {
           return "Hei fra CI-øvingen!";
       }
   }
   ```

3. Skriv en enkel test for controlleren. Legg fila her: `src/test/java/no/kristiania/cidemo/HelloControllerTest.java`:

   ```java
   package no.kristiania.cidemo;

   import org.junit.jupiter.api.Test;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
   import org.springframework.test.web.servlet.MockMvc;

   import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
   import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
   import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

   @WebMvcTest(HelloController.class)
   class HelloControllerTest {

       @Autowired
       MockMvc mockMvc;

       @Test
       void helloReturnsGreeting() throws Exception {
           mockMvc.perform(get("/hello"))
                  .andExpect(status().isOk())
                  .andExpect(content().string("Hei fra CI-øvingen!"));
       }
   }
   ```

   **Hva er `@WebMvcTest`?**

   Spring Boot har flere «test-slicer» — annotasjoner som starter opp *bare den delen* av applikasjonen du trenger for en gitt test. `@WebMvcTest` er slicen for web-laget:

   * Den starter **ikke** hele Spring Boot-appen (som `@SpringBootTest` gjør). Ingen database, ingen service-beans, ingen fullversjons-context.
   * Den laster kun controlleren du peker på (`HelloController.class`), pluss Spring MVC-infrastrukturen rundt (routing, JSON-serialisering, filter osv.).
   * Du får inn en **`MockMvc`** som lar deg sende falske HTTP-requests og gjøre assertions på responsen — uten å faktisk starte en Tomcat-server på en port.
   * Resultat: testen er **rask** (starter på ms, ikke sekunder) og fokusert — du tester controlleren, ikke hele appen.

   Kjør testen lokalt før du pusher:

   ```shell
   ./mvnw --batch-mode test
   ```

4. Commit og push branchen:

   ```shell
   git add .
   git commit -m "Add hello endpoint"
   git push -u origin feature/hello-endpoint
   ```

5. Gå til GitHub og opprett en **Pull Request** fra `feature/hello-endpoint` mot `main`.
6. Be medstudenten din om å reviewe PR-en (bruk **Reviewers**-feltet).
7. Medstudenten gjør en review og godkjenner den.
8. Merge PR-en.

Bytt roller og gjør det samme på medstudentens repo.

## Del 5 – Sett opp GitHub Actions workflow for CI

Vi skal lage en workflow som kjører **unit-tester** ved:

* hver **push til `main`**
* hver **Pull Request** som peker mot `main`

### Opprett workflow-fil

Lag fila `.github/workflows/ci.yml` i repoet:

```yaml
name: CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    steps:
      - name: Sjekk ut repo
        uses: actions/checkout@v4

      - name: Sett opp JDK 21
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven

      - name: Kjør tester
        run: ./mvnw --batch-mode test
```

### Test workflowen

1. Lag en ny feature branch, gjør en liten endring, og opprett en Pull Request.
2. Gå til fanen **Actions** i repoet og se at workflowen kjører.
3. På PR-en skal du nå se en grønn (eller rød) status-check ved siden av commit-en.

### Koble CI-sjekken til branch protection

Gå tilbake til `Settings` → `Branches` → rulen for `main`:

* Under **Require status checks to pass before merging**, søk opp og velg **`build-and-test`**.
  * Det du søker etter her er **navnet på jobben** i workflowen — altså nøkkelen under `jobs:` i `ci.yml` (i vårt tilfelle `build-and-test`). Det er **ikke** navnet på workflowen (`name: CI`) eller navnet på et enkeltsteg.
  * Merk: søkefeltet finner bare status-checks som GitHub har sett minst én gang. Har workflowen aldri kjørt mot dette repoet, får du ingen treff. Kjør PR-en fra forrige seksjon først.
  * *(Beklager på GitHub sine vegne at denne UX-en er litt dårlig — det er ikke opplagt at det er jobb-navnet du skal søke etter, og feltet gir null hint hvis workflowen ikke har kjørt ennå.)*
* Lagre.

Nå kan ingen PR merges før testene har passert.

### Oppgave: Få testene til å feile med vilje

* Endre en test slik at den feiler, push til en feature branch, og opprett en PR.
* Bekreft at:
  * Actions-workflowen slår rødt
  * Merge-knappen er deaktivert på PR-en
* Rett opp testen og bekreft at PR-en nå kan merges.

## Bonusoppgave: Legg til flere sjekker

Utvid `ci.yml` med noe av følgende:

* Kjør `./mvnw --batch-mode verify` i stedet for `test` (kjører også integrasjonstester)
* Legg til et steg som feiler bygget hvis kodedekningen er for lav (f.eks. med **JaCoCo**)
* Legg til et lint-steg med **Checkstyle** eller **Spotless**
* Kjør workflowen på flere Java-versjoner samtidig med en `matrix`-strategi

## Bonusoppgave: Krev "Conversation resolution" og "Signed commits"

I branch protection kan du kreve at alle review-kommentarer er løst, og at commits er signert. Slå på disse og se hva som skjer når du prøver å merge en PR med uløste kommentarer.

## Referanser

* [Spring Initializr](https://start.spring.io)
* [GitHub Actions – dokumentasjon](https://docs.github.com/en/actions)
* [About protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
* [setup-java action](https://github.com/actions/setup-java)
* [GitHub CLI (`gh`) – manual](https://cli.github.com/manual/)
* [`gh auth login`](https://cli.github.com/manual/gh_auth_login)
