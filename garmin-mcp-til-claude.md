# Å faktisk følge en treningsplan: en Claude-drevet morgencoach på egen hjemme-lab

## Problemet

Ting skjer. Man forsover seg, glemmer ting, har dårlig fokus en dag. Tiden går fort hvis man ikke stopper opp og planlegger, og det planleggingsarbeidet kan i seg selv kjennes som en ork - spesielt klokka syv om morgenen.

En treningsplan på papir hjelper bare så mye, fordi den er statisk. Den vet ikke at du sov dårlig i natt, at kalenderen er full til fire, eller at du faktisk trente hardt i går selv om det ikke sto i planen. Skipper du mandagsøkta, hva gjør du tirsdag - følger du tirsdagsøkta som kanskje er en hviledag, eller improviserer du? Det svarer ikke planen på.

Det jeg egentlig trengte var et informert perspektiv hver morgen, noe som vet hva som faktisk skjedde i går og hva som skjer i dag, og som kan si "gjør X" i stedet for at jeg må stykke sammen svaret selv fra tre forskjellige apper. En vanlig KI uten innsyn i disse dataene er verdiløs her - den kan gi generiske svar, ikke informerte.

## Løsningen, kort fortalt

En scheduled prompt i Claude som kjører hver morgen, leser kalenderen min og treningsdataene fra Garmin, og gir meg en konkret vurdering for dagen: tren, hvil, eller sqeeze inn en lett økt før møtet klokka ni.

## Hele kjeden

_(kommer til å erstatte dette med et faktisk diagram)_

```
Proxmox (eksisterende hjemme-lab)
  → LXC med garmin-mcp (HTTP-modus)
    → Cloudflare Tunnel
      → kjøpt domene
        → OAuth 2.1/PKCE-lag (del av garmin-mcp)
          → lagt til som custom connector i Claude.ai
Google Calendar-connector (Anthropics egen) ─┐
                                              ├→ scheduled task i Claude.ai, hver morgen
garmin-mcp-connector ─────────────────────────┘
```

Selve MCP-serveren skrev jeg ikke fra bunnen av - jeg brukte et eksisterende åpen kildekode-prosjekt, [garmin-mcp](https://github.com/Tyler-Irving/garmin-mcp), som eksponerer Garmin-data som verktøy Claude kan kalle, og som allerede har OAuth-laget ferdig implementert. Det sparte meg for det kjedeligste arbeidet.

Grunnen til at Cloudflare Tunnel og et ekte domene var nødvendig i det hele tatt: Claude sine remote connectors krever et offentlig gyldig TLS-sertifikat. Det interne/selvsignerte sertifikatet jeg allerede kjører i hjemme-laben blir ikke godtatt. Tunnelen løser dette uten at jeg måtte åpne en port direkte mot internett - den oppretter en utgående, kryptert forbindelse fra LXC-en til Cloudflares edge, som igjen serverer trafikken videre over gyldig TLS. Trafikken er altså kryptert hele veien, ikke bare det siste steget.

## Logikken bak anbefalingen

Prompten vurderer i hovedsak fire ting, i denne rekkefølgen:

1. **Kalenderen (Google Calendar)** - er det beste treningsvinduet mitt faktisk ledig i dag, eller er jeg opptatt?
2. **Har jeg "slacka" i det siste, eller trente jeg i går?** Hvis jeg trente i går som en "Fri"-økt på klokka (sånn jeg vanligvis logger styrke), må Claude selv analysere hvilke øvelser/muskelgrupper Garmin registrerte og vurdere om det var en Push-, Pull- eller Legs-økt - det står ikke ferdig kategorisert noe sted.
3. **Søvn/recovery** - var søvnen god nok til at jeg faktisk får noe ut av å presse inn en økt i dag, eller bør jeg hvile?
4. **Tidspunkt på dagen** - enkelte treningstilbud er billigere (og dermed mer aktuelle som student) tidlig på morgenen, så det spiller inn i anbefalingen.

## Hvorfor selvhosting

Jeg har allerede en server som kjører døgnet rundt, så det var det mest naturlige stedet å legge dette - mer gunstig enn å betale for noe i skyen. Det er også et personvernpoeng: dataene mine blir liggende hos meg selv i stedet for hos en tredjepart, så lenge trafikken faktisk er kryptert (noe tunnelen sørger for, se over). Å eie egne treningsdata er verdt noe i seg selv.

## Mer enn bare morgenanbefalingen

Fordi Claude har direkte tilgang til Garmin-dataene mine via MCP, er ikke dette begrenset til den faste morgenrutinen. Jeg kan også bare spørre direkte - for eksempel hvor mye jeg har trent siste måned sammenlignet med forrige, eller be om en vurdering av om treningsmengden har vært for lav eller for høy. Anbefalingen om morgenen er bare det faste, tilbakevendende bruksområdet.

## Sikkerhet

> Dette avsnittet er reservert plass - jeg har ikke rukket å konfigurere det enda, men det hører hjemme her.

Jeg kjører allerede Wazuh i hjemme-laben, men den overvåker per nå ikke LXC-en denne løsningen kjører i. Det bør den gjøre: selv om garmin-mcp er read-only som standard og skrivetilgang må skrus på eksplisitt, er dette fortsatt en tjeneste eksponert mot internett gjennom én passordbeskyttet OAuth-server. Gitt at jeg studerer digital sikkerhet er det ganske dårlig stil å la en internett-eksponert tjeneste stå uten overvåkning av autentiseringsforsøk og uvanlig trafikk. Dette kommer jeg tilbake til med faktisk konfigurasjon i en senere versjon av dette dokumentet.

## Utfordringer underveis

- **TLS-kravet** var ikke noe jeg hadde tenkt på før jeg faktisk prøvde å koble til - måtte kjøpe et billig domene og sette opp en tunnel i stedet for å bare peke Claude mot egen server.
- **En jobb uten tilsyn kan ikke vente på en MFA-kode klokka syv om morgenen.** garmin-mcp løser dette ved å kunne reautentisere seg selv mot Garmin med lagrede credentials, i stedet for å kreve manuell innlogging hver gang sesjonen utløper - nødvendig for at hele opplegget skulle kunne kjøre helt automatisk.

## Videre planer

- Få Wazuh til å faktisk overvåke LXC-en, og dokumentere det her.
- Diagram over hele kjeden.
- Vurdere om write-tool-en i garmin-mcp (som kan opprette økter direkte i Garmin-biblioteket) skal skrus på, slik at anbefalingen ikke bare blir foreslått, men faktisk lagt inn - foreløpig er det jeg som følger opp anbefalingen manuelt.
- Følge med over tid på om anbefalingene faktisk endrer noe, ikke bare at oppsettet fungerer teknisk.
- Vurdere kjøp av GPU for privat LLM for økt persornvern.
