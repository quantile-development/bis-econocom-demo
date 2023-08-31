# Meltano project guide

Deze guide legt uit hoe je je eerste Meltano project kun opzetten. De volgende aspecten worden hierin behandeld:

- Set-up: hoe zet je het meltano project op?
- Taps: hoe haal je data uit Hubspot en MS SQL?
- Targets: hoe verplaats ik data naar Hubspot / MS SQL?
- Pipeline runnen: hoe koppel je een tap / target

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

Wanneer een tap is geinstalleerd en je de configuratie hebt toegevoegd kun je met de volgende command testen of je data kunt binnen halen:

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
      replication-key: updated_at # Kolom waarop je incrementeel wilt laten
```

## Targets: data verplaatsen naar Hubspot / MS SQL

Het toevoegen van een target aan het Meltano project kan op vergelijkbare wijze:

`meltano add loader <target-name>`

Op de [Meltano Hub](https://hub.meltano.com/extractors/tap-hubspot) kun je ook targets vinden die publiek beschikbaar zijn.

Het testen van een target kan soms wat lastiger zijn omdat je target niet altijd opties heeft voor een veilige test omgeving. Daarom kan het fijn werken om een klein output bestand van een tap te gebruiken bij het testen van een target. Hiervoor kun je het volgende command gebruiken:

`cat ./output/tap-output-file-with-a-few-rows.txt | meltano invoke <target-name>`

Op deze wijze kun je eerst op kleine schaal testen of de target naar wens werkt voordat je aanpassingen gaat maken in een productie systeem.

### 1. Hubspot target toevoegen

Met de onderstaande command voeg je de Meltano variant van de Hubspot target toe:

`meltano add loader target-hubspot`

Nadat dit command succesvol is uitgevoerd zie je de volgende toevoeging in het `meltano.yml` bestand:

```yaml
plugins:
  loaders:
    - name: target-hubspot
      variant: meltanolabs
      pip_url: git+https://github.com/MeltanoLabs/target-hubspot.git
```

De volgende stap is om de target te configureren zodat het daadwerkelijk data naar Hubspot kan wegschrijven. Op de [Git repository van de target](https://github.com/MeltanoLabs/target-hubspot) lees je welke configuraties vereist en/of mogelijk zijn.

Hieronder zie je stap voor step aan welke configuraties je moet denken:

```yml
- name: target-hubspot
  variant: meltanolabs
  pip_url: git+https://github.com/MeltanoLabs/target-hubspot.git
  config:
    access_token: ${HUBSPOT_TOKEN}
    import_operations: UPSERT
```

Net zoals bij de tap is de `access_token` vereist om connectie te kunnen maken met Hubspot. Met de `import_operations` parameter kun je aangeven hoe je wilt dat data wordt geimporteerd naar Hubspot. Je kunt kiezen tussen `UPDATE`, `CREATE` en `UPSERT`. In dit voorbeeld is gekozen voor `UPSERT` zodat bestaande records worden geupdate en nieuwe records worden toegevoegd.

Vervolgens voegen we een `column_mapping` toe zoals hieronder weergeven:

```yml
- name: target-hubspot
  variant: meltanolabs
  pip_url: git+https://github.com/MeltanoLabs/target-hubspot.git
  config:
    access_token: ${HUBSPOT_TOKEN}
    import_operations: UPSERT
    column_mapping:
      - columnName: PROPERTIES__CUSTOM_PROPERTY # Kolom naam in tap
        propertyName: custom_property # Kolom naam in Hubspot
        columnObjectTypeId: 0-1 # Kolom type
      - columnName: PROPERTIES__EMAIL
        propertyName: email
        columnObjectTypeId: 0-1
      - columnName: PROPERTIES__FIRSTNAME
        propertyName: firstname
        columnObjectTypeId: 0-1
      - columnName: PROPERTIES__LASTNAME
        propertyName: lastname
        columnObjectTypeId: 0-1
```

Deze `column_mapping` geeft aan hoe de kolommen uit de tap overeenkomen met velden in Hubspot. Voor elke kolom moet je het volgende aangeven:

1. `columnName`: de kolom naam zoals het voorkomt in de tap
2. `propertyName`: de interne naam die Hubspot gebruikt voor dit veld (in Hubspot kun je dit vinden via Settings > Data Management > Properties)
3. `columnObjectTypeId`: de typeid van het Hubspot object waartoe dit veld behoort (e.g. Deals, Contact etc. / zie dit [Hubspot artikel](https://developers.hubspot.com/docs/api/crm/understanding-the-crm#object-type-id))

In het bovenstaande voorbeeld zijn er dus 4 kolommen die uit de tap komen genaamd: `PROPERTIES__CUSTOM_PROPERTY`, `PROPERTIES__EMAIL`, `PROPERTIES__FIRSTNAME` en `PROPERTIES__LASTNAME`. Deze corresponderen met de Hubspot velden: `custom_property`, `email`, `firstname` en `lastname`. Dit geld voor het Hubspot object type `0-1`, wat het Contacts object in Hubspot is.

Het kan echter voorkomen in praktijk dat de tap die je wilt koppelen aan de Hubspot target ook kolommen mee geeft die niet bestaan of die je niet zelf wilt updaten. Bijvoorbeeld een `updated_at` kolom laat je liever aan Hubspot zelf om te updaten. Zulke kolommen kun je aan de hand van `stream_maps` achterwege laten. Hieronder zie je hoe je dit configureert:

```yaml
- name: target-hubspot
  variant: meltanolabs
  pip_url: git+https://github.com/MeltanoLabs/target-hubspot.git
  config:
    access_token: ${HUBSPOT_TOKEN}
    import_operations: UPSERT
    stream_maps:
      DEVELOPMENT-TAP_HUBSPOT-CONTACTS: # Naam van de stream
        UPDATED_AT: __NULL__
        ID: __NULL__
```

In de `stream_maps` config moet je eerst de stream naam specificeren waarvoor je aanpassingen wilt maken. De stream naam van een tap kun je vinden door de output van je tap te bekijken. Met dit command `meltano invoke tap-naar-keuze > ./output/tap-output.txt` schrijf je de output makkelijk wel naar een tekst bestand.

Een van de eerste regels van dit bestand beschrijft de schema van de tap met daarin ook een stream key waarvan de waarde aangeeft wat de stream naam is. In het geval hieronder dus `dbo-mock_data`.

`{"type": "SCHEMA", "stream": "dbo-mock_data", "schema": ...}`

Vervolgens kun je bepaalde mappings en/of andere aanpassingen uitvoeren op de beschikbare kolommen voordat deze naar de target worden gestuurd. In het voorbeeld hierboven zien we dat de `UPDATED_AT` en `ID` kolom beiden op `__NULL__` worden gezet. Dit houdt in dat je er expliciet voor kiest om deze kolommen niet mee te geven aan de target.

Voor meer en/of andere mapping mogelijkheden kun je het beste terecht bij [deze documentatie pagina](https://sdk.meltano.com/en/latest/stream_maps.html) van Meltano.

Al met al is de configuratie van de `target-hubspot` dus afhankelijk van met welke tap je deze wilt gaan gebruiken. In de `column_mapping` geef je aan hoe de kolommen tussen de tap en Hubspot overeenkomen en het kan voorkomen dat je bepaalde kolommen uit de tap expliciet wilt negeren waarvoor je de `stream_maps` kunt gebruiken.

### 2. MS SQL target toevoegen

Met de onderstaande command voeg je de Meltano variant van de MS SQL target toe:

`meltano add loader target-mssql`

Nadat dit command succesvol is uitgevoerd zie je de volgende toevoeging in het `meltano.yml` bestand:

```yaml
- name: target-mssql
  variant: storebrand
  pip_url: git+https://github.com/storebrand/target-mssql.git
```

De volgende stap is om de target te configureren zodat het daadwerkelijk data naar MS SQL kan wegschrijven. Op de [Git repository van de target](https://github.com/storebrand/target-mssql) lees je welke configuraties vereist en/of mogelijk zijn.

```yaml
- name: target-mssql
  variant: storebrand
  pip_url: git+https://github.com/storebrand/target-mssql.git
  config:
    host: host.docker.internal
    database: test
    port: "1433"
    username: sa
    password: password123UP
```

## Pipeline runnen: hoe koppel je een tap / target

Nadat de taps en targets zijn geinstalleerd met configuratie kun je ze gezamelijk gaan uitvoeren. Dit doe je in meltano aan de hand van het `meltano run <tap-naam> <target-naam>` command. De volgende command zou bijvoorbeeld data uit Hubspot ingesten en naar MS SQL wegschrijven:

`meltano run tap-hubspot target-mssql`

Wanneer een tap incremental loading ondersteunt houdt Meltano ook automatisch de state bij van jouw pipelines. De state van je runs kun je terug vinden aan de hand van de volgende command:

`meltano state list`

Dit command zal een lijst laten zien van id's van specifiek runs die je hebt uitgevoerd. Een id is altijd opgebouwd uit de volgende componenten: `<environment_name>:<tap_name>-to-<target_name>(:<state_id_suffix)`. Wanneer voor een specifieke run de laatste state value wil bekijken kun je het volgende command gebruiken:

`meltano state get dev:tap-hubspot-to-target-mssql`

# Dagster

Het is mogelijk om je Meltano pipelines te orchestraten binnen Dagster. Hiervoor kun je de `dagter-meltano` plugin gebruiken die wij hebben ontwikkelt. Dit proces bestaat uit een aantal stappen:

### 1. Pipelines definieren die je wilt uitvoeren in `meltano.yml`

Hieronder maken we bijvoorbeeld een job aan die een reeks van taken moet gaan uitvoeren. In dit geval bijvoorbeeld zowel het uitvoeren van de `tap-hubspot` naar `target-mssql` als andersom.

```yaml
jobs:
  - name: test-job
    tasks:
      - tap-hubspot target-mssql
      - tap-mssql target-hubspot
```

### 2. Schedule aangeven hoe vaak je de pipeline wilt uitvoeren in `meltano.yml`

Bijvoorbeeld een dagelijkse run

```yaml
schedules:
  - name: daily-test-job
    interval: "@daily"
    job: test-job
```

### 3. Dagster koppelen aan het meltano project met de `dagster-meltano` plugin

In de `orchestrate/dagster/respository.py` koppelen we het Dagster aan het Meltano project. Met slechts enkel wat boilerplate code staat zo onze pipeline in Dagster.

Je kunt Dagster lokaal starten door middel van het volgende command (vanaf root folder):

`dagster dev -f orchestrate/dagster/repository.py`

Let op dat je hierbij de poetry environment gebruikt. Ofwel als je de error `bash: dagster: command not found` krijgt moet je nog even `poetry shell` runnen.
