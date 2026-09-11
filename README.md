# Kontinuerlig integrasjon med Spring Boot og GitHub Actions

I denne øvingen skal du:

* Generere et nytt **Spring Boot**-prosjekt med **Spring Initializr**
* Opprette et **eget GitHub-repo** og pushe koden dit
* Sette opp **branch protection** på `main`
* **Invitere en medstudent** som collaborator
* Øve på **Pull Request-flyten** (feature branch → PR → review → merge)
* Sette opp en **GitHub Actions workflow** som kjører **unit-tester** på hver PR og på hver push til `main`

Målet er ikke å lære Spring Boot i dybden – vi bruker det kun som et konkret prosjekt å bygge og teste. Fokuset er på **arbeidsflyten** rundt kode: branching, code review, og kontinuerlig integrasjon (CI).

Underveis kommer vi til å bruke **GitHub CLI (`gh`)** til å autentisere Git og opprette repoer fra terminalen, og noen litt mer avanserte **Codespaces**-triks — blant annet å legge til flere rot-mapper i workspacet så du kan jobbe med to prosjekter side om side i samme editor.

## Læringsmål

Etter å ha fullført øvingen skal du kunne:

* Forklare hva **CI (Continuous Integration)** er og hvorfor det er nyttig
* Sette opp **branch protection rules** for å hindre direkte push til `main`
* Lage en **Pull Request**, be om review, og merge etter godkjenning
* Skrive en enkel **GitHub Actions workflow** som bygger og tester et Maven-prosjekt
* Bruke **GitHub CLI (`gh`)** til å logge inn og opprette et nytt repo fra kommandolinjen (`gh auth login`, `gh repo create`)
* Bruke **Codespaces multi-root workspaces** (`code -a`) for å jobbe med flere mapper samtidig i samme editor-vindu
* Forstå hva **Maven Wrapper** (`./mvnw`) er, og hvorfor det gjør builds mer reproduserbare på tvers av maskiner og CI-systemer

## Viktig: Vi jobber med to Git-repoer i denne øvingen — hold dem adskilt!

Dette er en klassisk felle som *kommer* til å forvirre deg om du ikke er obs på det fra start. I løpet av øvingen jobber du med **to helt separate GitHub-repoer**, og de skal ligge i **to sidestilte mapper** — ikke den ene inne i den andre:

```
/workspaces/
├── ci-spring-boot/     ← Repo 1: din fork (bare for README + devcontainer)
└── ci-demo/            ← Repo 2: ditt nye Spring Boot-prosjekt
```

| # | Repo | Mappe | Hva ligger her? |
|---|------|-------|-----------------|
| 1 | **Din fork** (`ci-spring-boot`) | `/workspaces/ci-spring-boot/` | README-en du leser nå og `.devcontainer/`. **Ikke push endringer hit.** |
| 2 | **Ditt nye Spring Boot-repo** (f.eks. `ci-spring-boot-ola`) | `/workspaces/ci-demo/` | Selve Spring Boot-prosjektet du lager i Del 1. **Det er her all koden din, PR-ene, branch protection og CI skal leve.** |

Legger du Spring Boot-prosjektet **inne i** fork-en, går git-kommandoene lett til feil repo — git leter oppover i mappetreet og treffer fork-ens `.git/` uten at du merker det. Hold repo 2 utenfor repo 1. Del 1 forteller deg hvor du skal legge det (`/workspaces/ci-demo/`).

### Sjekk hvor du er før du gjør noe med Git

```shell
pwd                # Hvilken mappe står jeg i?
git remote -v      # Hvilket GitHub-repo peker denne mappa på?
```

Ser du en URL med `glennbechdevops/ci-spring-boot` (lærerens repo), er du i fork-en. Ser du en URL med **ditt eget** brukernavn og repo-navnet du valgte, er du i ditt eget repo. Ser du **ingenting** i `git remote -v` fra `ci-demo/` før du har lagt til remote — det er som forventet.

## Lag en fork

En **fork** er din egen kopi av et repo på GitHub. Du trenger en for å kunne starte en Codespace og endre filer uten å påvirke originalen.

Klikk **Fork**-knappen øverst til høyre på dette repoet og velg din egen konto som destinasjon.

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

### Gjør `ci-demo/` synlig i fil-treet

VS Code / Codespaces viser bare workspace-roten (fork-en) i Explorer-panelet, så `/workspaces/ci-demo/` er på disk men usynlig i sidepanelet. Legg den til som ekstra rot-mappe:

```shell
code -a /workspaces/ci-demo
```

`-a` er kortformen for `--add` — den *legger til* mappa i workspacet uten å bytte hovedmappe eller reloade vinduet. Nå ser du begge mappene sidestilt i Explorer, og kan åpne filer i `ci-demo/` med musa som normalt.

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

> **Hva er `./mvnw`?** Det er **Maven Wrapper** — et lite shell-script (og `.cmd`-variant for Windows) som følger med prosjektet. Første gang du kjører det, laster det ned den nøyaktige Maven-versjonen prosjektet er testet med (se `.mvn/wrapper/maven-wrapper.properties`) og bruker den for bygget. Det betyr at *alle* — du lokalt, medstudenten din, GitHub Actions-runnerne — bygger med samme Maven-versjon uten å måtte installere Maven manuelt. `./mvnw` er en drop-in erstatning for `mvn`, så alle kommandoer du kunne kjørt med `mvn` (`test`, `package`, `verify`, …) fungerer likt med `./mvnw`.

## Del 2 – Opprett et nytt GitHub-repo

### Autentisering mot GitHub (viktig!)

Før du kan pushe kode, må Git kunne bevise til GitHub at det er *deg* som pusher. I et helt ferskt Codespace har du **ikke** SSH-nøkler satt opp, og passord-innlogging over HTTPS ble deaktivert av GitHub for flere år siden. Derfor må du bruke ett av alternativene under.

**Anbefalt (Codespaces og lokalt): bruk `gh` CLI**

`gh` (GitHub CLI) er forhåndsinstallert i Codespaces og på de fleste utviklermaskiner.

> **NB (Codespaces):** Codespace-en har allerede en `GITHUB_TOKEN`-env-var satt som er scopet til fork-en. `gh` vil bruke den og hoppe over innlogging — men den tokenet får ikke opprette et nytt repo på kontoen din. Fjern den først:
>
> ```shell
> unset GITHUB_TOKEN
> ```

```shell
gh auth login
```

Velg:
* **GitHub.com**
* **HTTPS** som protokoll
* **Yes** når den spør om å bruke `gh` som Git credential helper
* **Login with a web browser** — kopier engangskoden og lim inn i nettleseren

Etter dette har Git en credential helper som automatisk sender en gyldig token ved hver `git push` — du slipper å taste noe mer.

### Sett git-identiteten din (bare første gang)

```shell
git config --global user.name  "Ola Nordmann"
git config --global user.email "ola@example.com"
```

### Opprett repoet og push

Fra rot-mappa av Spring-prosjektet:

```shell
git init
git add .
git commit -m "Initial commit from Spring Initializr"
git branch -M main
gh repo create ci-spring-boot-ditt-navn --public --source=. --push
```

Bytt `ditt-navn` med noe unikt (f.eks. `ci-spring-boot-ola`). Flaggene til `gh repo create`:

* `--public` — repoet blir offentlig. Bruk `--private` hvis du heller vil ha det privat.
* `--source=.` — bruk gjeldende mappe som kilde. `gh` finner den lokale `.git/`-mappa og setter opp riktig `origin`-remote.
* `--push` — push `main` opp til det nye repoet umiddelbart etter opprettelse.

Resultat: repoet finnes på GitHub-kontoen din, `origin` er satt, og `main` er pushet — alt i én kommando.

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
   import org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest;
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

   `--batch-mode` (kortform `-B`) skrur av interaktiv output og fargede tegn. Kjekt i CI og i terminaler der du vil ha kortfattet, maskin-lesbar logg. Kan droppes lokalt — `./mvnw test` alene fungerer også.

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
        uses: actions/setup-java@v5
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
