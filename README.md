# CI for Spring Boot med GitHub Actions

I denne øvelsen skal du:

* Generere et nytt **Spring Boot**-prosjekt med **Spring Initializr**
* Opprette et **eget GitHub-repo** og pushe koden dit
* Sette opp **branch protection** på `main`
* **Invitere en medstudent** som collaborator
* Øve på **Pull Request-flyten** (feature branch → PR → review → merge)
* Sette opp en **GitHub Actions workflow** som kjører **unit-tester** på hver PR og på hver push til `main`

Målet er ikke å lære Spring Boot i dybden – vi bruker det kun som et konkret prosjekt å bygge og teste. Fokuset er på **arbeidsflyten** rundt kode: branching, code review, og kontinuerlig integrasjon (CI).

## Læringsmål

Etter å ha fullført øvelsen skal du kunne:

* Forklare hva **CI (Continuous Integration)** er og hvorfor det er nyttig
* Sette opp **branch protection rules** for å hindre direkte push til `main`
* Lage en **Pull Request**, be om review, og merge etter godkjenning
* Skrive en enkel **GitHub Actions workflow** som bygger og tester et Maven-prosjekt

## Lag en fork

Du må starte med å lage en fork av dette repoet til din egen GitHub-konto. Bruk **Fork**-knappen øverst til høyre på GitHub.

## Start et Codespace

I din fork, velg den grønne knappen "<> Code", og "Create codespace on main".

Dette repoet har en **devcontainer** (se `.devcontainer/devcontainer.json`) som gjør at Codespaces automatisk installerer alt du trenger:

* **JDK 21** (Temurin) – for å kompilere og kjøre Java-koden
* **Maven** – byggeverktøyet vi bruker for å bygge Spring Boot-prosjektet

Første gang Codespacet startes tar det noen minutter å bygge miljøet.

### Verifiser at verktøyene er tilgjengelige

Sjekk at alt er på plass ved å kjøre følgende i terminalen:

```shell
java -version
mvn -version
```

Du skal få et versjonsnummer tilbake for begge. Får du "command not found" er devcontaineren ikke ferdig bygget, eller bygget feilet – sjekk loggen fra "Codespaces: View Creation Log".

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

> **Tips:** Du kan også laste ned prosjektet direkte i Codespaces-terminalen. Eksempel:
>
> ```shell
> curl https://start.spring.io/starter.zip \
>   -d type=maven-project -d language=java \
>   -d groupId=no.kristiania.cidemo -d artifactId=ci-demo \
>   -d name=ci-demo -d packageName=no.kristiania.cidemo \
>   -d javaVersion=21 -d dependencies=web \
>   -o ci-demo.zip && unzip ci-demo.zip -d ci-demo
> ```
>
> Uten `bootVersion` bruker Initializr default (siste stabile). Hvis du vil pinne versjon, se `https://start.spring.io/metadata/client` for aktuelle valg.

Pakk ut innholdet i en tom mappe (lokalt eller i Codespace).

### Verifiser at prosjektet bygger

```shell
./mvnw test
```

Du skal få en grønn build med minst én test (`contextLoads`) som passerer.

## Del 2 – Opprett et nytt GitHub-repo

1. Gå til [github.com/new](https://github.com/new) og opprett et **tomt** repo (ikke huk av for README, .gitignore eller lisens – prosjektet fra Initializr har allerede dette).
2. Kall det `ci-spring-boot-<dine initialer>`.
3. Følg instruksjonene GitHub gir for å pushe et eksisterende prosjekt:

```shell
git init
git add .
git commit -m "Initial commit from Spring Initializr"
git branch -M main
git remote add origin git@github.com:<ditt-brukernavn>/ci-spring-boot-<initialer>.git
git push -u origin main
```

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
