# Meltano project guide

Deze guide legt uit hoe je je eerste Meltano project kun opzetten. De volgende aspecten worden hierin behandeld:

- Set-up: hoe zet je het meltano project op?
- Taps: hoe haal je data uit Hubspot en MS SQL?
- Targets: hoe verplaats ik data naar Hubspot / MS SQL?

In de guide wordt verwezen naar handige en nuttige bronnen waar je terecht kunt voor meer informatie.

## Set-up: opzet van het Meltano project

De eerste stap is uiteraard het installeren van Meltano zodat het beschikbaar is in jouw werkomgeving. In deze repo wordt gebruik gemaakt van een VS Code devcontainer waarin alle dependencies inclusief Meltano automatisch worden geinstalleerd.

Voor informatie over alternatieve installatie methoden kun je het beste terecht bij op de volgende webpagina: https://docs.meltano.com/getting-started/installation.

Vervolgens is de eerste stap om een Meltano project op te zetten. Dit doe je met de onderstaande command:

`meltano init my-meltano-project`

Dit genereert de standaard Meltano project folder structuur:

```
my-new-project/
|-- .meltano
|-- meltano.yml
|-- README.md
|-- requirements.txt
|-- output/.gitignore
|-- .gitignore
|-- extract/.gitkeep
|-- load/.gitkeep
|-- transform/.gitkeep
|-- analyze/.gitkeep
|-- notebook/.gitkeep
|-- orchestrate/.gitkeep
```

Het `meltano.yml` bestand is het centrale configuratiebestand voor het Meltano-project. Hierin komen alle configuraties voor de taps en targets die we hieronder gaan installeren. Ook kun je hier verschillende ontwikkelomgevingen configureren als je onderscheid wil maken tussen development en productie omgevingen.

## Taps: data ophalen uit Hubspot / MS SQL

Het toevoegen van een tap aan het Meltano project is eenvoudig met het volgende command:

`meltano add extractor <tap-name>`

Op de [Meltano Hub](https://hub.meltano.com/extractors/tap-hubspot) kun je vinden welke taps (i.e. extractors) publiek beschikbaar zijn.

Wanneer een tap is geinstalleerd en je de configuratie hebt toegevoegd kunt je met de volgende command testen of je data kunt binnen halen:

`meltano invoke <tap-name>`

Voor debugging purposes kan het soms handig zijn om de output weg te schrijven naar een bestand, dit doe je zo:

`meltano invoke <tap-name> > ./output/your-ingested-data.txt`

### 1. Hubspot tap toevoegen

Voor Hubspot zijn er verschillende taps beschikbaar op de Meltano. In dit project heb ik gekozen voor de Potloc variant omdat deze het binnen halen van custom velden ondersteunt. Incremenal loading is nog niet ondersteund in deze tap, maar hievoor werk ik aan een pull request om dit toe te gaan laten voegen. Hieronder laat ik zowel zien hoe je de Potloc variant kun toevoegen aan het Meltano project als hoe je de fork variant van mij kunt toevoegen die incremental loading ondersteunt.

#### Potloc variant toevoegen

Met de onderstaande command voeg je de Potloc variant van de Hubspot tap toe:

`meltano add extractor tap-hubspot --variant potloc`

Nadat dit command succesvol is uitgevoerd zie je de volgende toevoeging in het `meltano.yml` bestand:

```yaml
plugins:
  extractors:
    - name: tap-hubspot
      variant: potloc
      pip_url: git+https://github.com/potloc/tap-hubspot.git
```

De volgende stap is om de tap te configureren zodat het daadwerkelijk data uit Hubspot kan halen. Op de [Git repository van de tap](https://github.com/potloc/tap-hubspot) lees je welke configuraties vereist en/of mogelijk zijn.

In dit project is gekozen voor de volgende configuratie:

```yml
- name: tap-hubspot
  variant: potloc
  pip_url: git+https://github.com/potloc/tap-hubspot.git
  config:
    access_token: ${HUBSPOT_TOKEN}
    flattening_enabled: true # Deze parameter zorgt dat tap resultaten worden platgeslagen
    flattening_max_depth: 1 # Flattening gebeurt voor maximaal 1 diepte niveau
```

De `access_token` is de enige vereiste configuratie parameter en correspondeerd met de access token van een [private app](https://developers.hubspot.com/docs/api/private-apps) in Hubspot. In dit geval wordt de acces token ingeladen vanuit een environment variable `HUBSPOT_TOKEN` die gedefineerd wordt in het `.env` bestand.

De `tap-hubspot` haal per default heel veel data naar binnen zoals o.a. deals, companies, contacten en meer. Door de onderstaande command uit te voeren kun je zien welke informatie de tap allemaal kan ophalen.

`meltano select tap-hubspot --select --all`

Vaak hoef je niet per se alle data op te halen en wil je een specifieke selectie maken van de mogelijkheden. Dit is mogelijk aan de hand van de [`meltano select` command](https://docs.meltano.com/reference/command-line-interface/#select). Op basis van wildcards kun je specifieke attributen en/of entiteiten selecteren die de tap moet ophalen.

Zo kun je bijvoorbeeld met de onderstaande commands bijvoorbeeld alle properties van de contacts ophalen en expliciet de associations van contacts objecten negeren.

`meltano select tap-hubspot contacts "properties.*"`

`meltano select tap-hubspot --exclude contacts "associations"`

`meltano select tap-hubspot --exclude contacts.associations "*"`

Het uitvoeren van deze commands voegt deze selectie automatisch toe aan het `meltano.yml` bestand. Uiteraard is het ook gewoon een optie om het zelf daar gelijk uit te typen. Uiteindelijk komen we zo op de onderstaande configuratie uit voor de Hubspot tap.

```yaml
- name: tap-hubspot
  variant: potloc
  pip_url: git+https://github.com/potloc/tap-hubspot.git
  config:
    access_token: ${HUBSPOT_TOKEN}
    flattening_enabled: true
    flattening_max_depth: 1
  select:
    - contacts.properties.* # Haal alle properties van contact entiteit op
    - "!contacts.associations" # Negeer de associations van contact entiteit
    - "!contacts.associations.*" # Negeer ook alle attributen van deze associations
```

#### Fork variant (met incremental loading) toevoegen

Om mijn fork van de Potloc tap-hubspot te gebruiken die incremental loading ondersteund is een kleine aanpassing nodig omdat dit in een andere repository staat. Aangezien deze fork niet op de Meltano Hub beschikbaar is moet je het zelf handmatig invullen in het `meltano.yml` bestand zoals hier weergeven.

De configuratie is verder hetzelfde als voor de Potloc tap. Ook is het nu mogelijk om een `start_date` parameter aan te geven als je pas vanaf een specifiek moment data wilt ophalen.

```yaml
- name: tap-hubspot-incremental
  namespace: tap_hubspot
  pip_url: git+https://github.com/BernardWez/tap-hubspot.git
  executable: tap-hubspot
  config:
    access_token: ${HUBSPOT_TOKEN}
    start_date: "2023-08-25T00:00:00Z" # Vanaf dit moment wordt data opgehaald
    flattening_enabled: true
    flattening_max_depth: 1
  select:
    - contacts.properties.*
    - "!contacts.associations.*"
    - "!contacts.associations"
```

Let op: deze fork variant ondersteund momenteel alleen het ophalen van contacten uit Hubspot. De komende week werk ik aan een pull request om dit voor contacten, deals en companies in de Potloc tap toe te laten voegen.

### 2. MS SQL tap toevoegen

Het toevoegen van de MS SQL tap is erg vergelijkbaar als de Potloc tap en doe je met de volgende command:

`meltano add extractor tap-mssql`

Dit voegt het volgende toe aan het `meltano.yml` bestand:

```yaml
- name: tap-mssql
  variant: wintersrd
  pip_url: tap-mssql
```

Om de configuratie van de tap af te maken moeten we de `host`, `database`, `port`, `user` en `password` parameters instellen. Dit doen we als volgt:

```yaml
- name: tap-mssql
  variant: wintersrd
  pip_url: tap-mssql
  config:
    host: host.docker.internal
    database: test
    port: 1433
    user: sa
    password: password123UP
```

Zonder verdere instellingen zal de `tap-mssql` alle tabellen uit de database proberen te lezen die beschikbaar zijn. Door de `catalog` instelling van de tap aan te passen kun je dit ook zelf configureren. Om dit te doen is het handig om eerst het onderstaande meltano command uit te voeren:

`meltano invoke --dump=catalog tap-mssql > ./output/catalog.json`

Vervolgens voegen we nog een `select` key toe aan de configuratie van de tap om de specifieke tabel die we willen ophalen te specificeren.

```yaml
- name: tap-mssql
  variant: wintersrd
  pip_url: tap-mssql
  config:
    host: host.docker.internal
    database: test
    port: 1433
    user: sa
    password: password123UP
  select:
    - dbo-mock_data.* # Selecteer alle kolommen van schema `dbo` en tabel `mock_data`
```

Tot slot kunnen we deze tap incrementeel data laten ophalen door de volgende configuratie toe te voegen:

```yaml
- name: tap-mssql
  variant: wintersrd
  pip_url: tap-mssql
  config:
    host: host.docker.internal
    database: test
    port: 1433
    user: sa
    password: password123UP
  select:
    - dbo-mock_data.*
  metadata:
    dbo-mock_data: # Hier stel je de schema_naam-tabel_naam in
      replication-method: INCREMENTAL
      replication-key: updatedAt # Kolom waarop je incrementeel wilt laten
```

## Targets: data verplaatsen naar Hubspot / MS SQL

TODO
