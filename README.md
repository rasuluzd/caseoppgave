# Case-oppgave: Backend

Jeg har valgt backend-oppgaven. Nummereringen følger oppgaven, og under hver overskrift står spørsmålet jeg svarer på.

Jeg går ut fra at kunden har lagt inn bilen sin i steget før, og at det da ble laget et tilbud med priser.

## 1. Backend-oppgaven

### Klasser og felt

> Hvilke klasser og tilhørende felt tenker du er nødvendig for å kunne presentere dataene på denne siden, gitt at hele komponenten skal skapes med data fra et API?

Komponenten trenger fire klasser, som jeg har skrevet som Java-records:

```java
public record Dekningsvalg(
        String tittel,              // "Velg dekning"
        List<Dekning> dekninger,    // i rekkefølgen de skal vises
        Anbefaling anbefaling,      // null hvis ingen er anbefalt
        String valgtKode,           // "KASKO_MED_LEIEBIL", eller null til kunden har valgt
        Lenke sammenligning) {}     // "Sammenlign dekninger"

public record Dekning(
        String kode,                // fast ID, for eksempel "KASKO_MED_LEIEBIL"
        String navn,                // "Kasko med leiebil"
        long maanedsprisOre,        // 117900 betyr 1 179 kr
        String beskrivelse,         // vises når dekningen er valgt
        List<String> inkluderer) {} // ["minikasko", "ansvar"]

public record Anbefaling(
        String kode,                // "TOPPKASKO"
        String tekst) {}            // "Anbefalt"

public record Lenke(String tekst, String url) {}
```

Alt som vises kommer fra API-et, også tittelen, teksten i taggen og lenken, og frontend gjør bare formatering, som å vise 117900 øre som «1 179 kr/md.». Prisen er et heltall i øre fordi `double` gir avrundingsfeil med penger, og anbefalt og valgt dekning ligger på toppnivå så det aldri kan bli to av dem.

### API

> Hvordan ville du implementert et API (eller lignende) for å hente dataene?

Jeg ville laget et REST API med to endepunkter:

| Metode | URL | Request body | Svar |
|---|---|---|---|
| `GET` | `/api/v1/tilbud/{tilbudId}/dekningsvalg` | ingen | `200` med `Dekningsvalg` |
| `PUT` | `/api/v1/tilbud/{tilbudId}/valgt-dekning` | `{ "kode": "KASKO" }` | `204` |

`tilbudId` er en UUID som ble laget i steget før. GET henter prisene fra tilbudet og navn og beskrivelser fra produktkatalogen og setter dem sammen til ett svar, så komponenten trenger bare ett kall. PUT lagrer dekningen kunden klikker på. Hører tilbudet til en annen sesjon, svarer API-et `404`.

Svaret fra GET med dataene fra skissen (Kasko, Minikasko og Ansvar er tatt ut for å spare plass):

```json
{
  "tittel": "Velg dekning",
  "dekninger": [
    {
      "kode": "TOPPKASKO",
      "navn": "Toppkasko",
      "maanedsprisOre": 123500,
      "beskrivelse": "Den mest omfattende forsikringen vår. Passer for nye og nyere biler.",
      "inkluderer": ["kasko med leiebil", "minikasko", "ansvar"]
    },
    {
      "kode": "KASKO_MED_LEIEBIL",
      "navn": "Kasko med leiebil",
      "maanedsprisOre": 117900,
      "beskrivelse": "En forsikring som passer de fleste biler. Dekker det meste av skader, også de du selv er ansvarlig for.",
      "inkluderer": ["minikasko", "ansvar"]
    }
  ],
  "anbefaling": { "kode": "TOPPKASKO", "tekst": "Anbefalt" },
  "valgtKode": "KASKO_MED_LEIEBIL",
  "sammenligning": { "tekst": "Sammenlign dekninger", "url": "/bilforsikring/sammenlign-dekninger" }
}
```

I Spring Boot blir GET-endepunktet omtrent slik:

```java
@GetMapping("/api/v1/tilbud/{tilbudId}/dekningsvalg")
public ResponseEntity<Dekningsvalg> hentDekningsvalg(@PathVariable UUID tilbudId, HttpSession sesjon) {
    // Kaster en 404-feil hvis tilbudet ikke finnes eller hører til en annen sesjon
    Dekningsvalg dekningsvalg = tilbudService.hentDekningsvalg(tilbudId, sesjon);
    return ResponseEntity.ok()
            .cacheControl(CacheControl.noStore()) // personlige priser skal ikke caches
            .body(dekningsvalg);
}
```

## 2. Sikkerhet og personvern

> Nevn noen tiltak du mener utviklerne bør gjøre for å ivareta sikkerhet og/eller personvern i denne løsningen?

Registreringsnummeret kan knyttes til den som eier bilen, og prisen bygger på opplysninger om kunden, så begge deler er personopplysninger. Dette er tiltakene jeg mener utviklerne bør gjøre, med det viktigste først:

1. Sjekk på hvert kall at tilbudet hører til den som spør, og svar `404` hvis ikke, så ingen kan finne ut hvilke tilbud som finnes. Dette er nummer én på OWASP API Security Top 10, og tilfeldige UUID-er gjør ID-ene umulige å gjette.
2. Ikke la prisen avsløre noe om eieren av bilen. Hvis prisen bygger på bonus eller alder som vi henter om eieren, kan hvem som helst skrive inn naboens registreringsnummer og lese det ut av prisen. Før kunden har logget inn, bør prisen derfor bare bygge på det kunden selv har oppgitt.
3. Hold registreringsnummer og andre personopplysninger unna URL-er og logger, fordi URL-er havner i serverlogger og nettleserhistorikk.
4. Be bare om det prisen trenger, og slett tilbud som ikke blir kjøpt etter en fast tid, for eksempel 30 dager.
5. Bruk HTTPS på all trafikk.
6. Ha rate limiting, så ingen kan lage tusenvis av tilbud for å kartlegge prismodellen.

## 3. Universell utforming

> Hva mener du utviklere og designere bør tenke på for å ivareta universell utforming i denne løsningen?

Det er lovpålagt at nettsider skal være universelt utformet, og i praksis betyr det å følge WCAG. Når jeg ser på skissen, er dette det jeg ville passet på.

For designere:

- Valgt dekning må synes på mer enn bare farge. Skissen gjør det bra med tykkere ramme og fylt knapp, så det bør beholdes.
- Det må være god nok kontrast, så rammer, tekst og «Anbefalt»-taggen er tydelige også for folk som ser dårlig.
- Mange vet ikke forskjellen på kasko og minikasko, så beskrivelsene bør være korte og enkle, og «Sammenlign dekninger» bør være lett å finne.

For utviklere:

- Det må gå an å velge dekning med bare tastaturet, og det må tydelig synes hvilken dekning man står på, ikke bare hvilken som er valgt. En skjermleser må lese opp navnet, prisen og om dekningen er anbefalt. Det enkleste er å bruke vanlige radioknapper i HTML i stedet for å lage egne, for da får man det meste gratis.
- Hele boksen bør være klikkbar, ikke bare den lille sirkelen. Det er lettere på mobil og for folk som ikke har så stø hånd.
- Teksten må kunne forstørres uten at navnet og prisen havner oppå hverandre.

## Bruk av KI

Jeg har brukt KI til å formatere og rettskrive teksten så den blir lettere å lese, blant annet tabellen, spørsmålene fra oppgaven og kodeeksemplene.
