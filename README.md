# Infrastruktur som kode med Terraform og AWS App runner

Når du er ferdig med denne oppgaven vil du ha et repository som inneholder en Spring Boot applikasjon.
Når du gjør en commit på main branch i dette, så vil GitHub actions gjøre

* Lage et Docker Image, og Push til ECR. Både med "latest" tag - og med en spesifikk  tag som matcher git commit.
* Bruke Terraform til å lage AWS infrastruktur, IAM roller og en AWS App Runner Service.

* I denne oppgaven skal vi gjøre en en docker container tilgjengelig over internett ved hjelp av en tjeneste AWS Apprunner.
* Apprunner lager nødvendig infrastruktur for containeren, slik at du som utvikler kan fokusere på koden.

Vi skal også se nærmere på mer avansert GitHub Actions: For eksempel;

* To jobber og avhengigheter mellom disse.
* En jobb vil lage infrastruktur med terraform, den andre bygge Docker container image
* Bruke terraform i Pipeline - GitHub actions skal kjøre Terraform for oss.
* En liten intro til AWS IAM og Roller

## Litt repetisjon

* En terraform *provider* er det magiske elementet som gir Terraform mulighet til å fungere med en lang rekke tjenester og produkter, foreksempel AWS, Azure, Google Cloud, osv.
* En terraform *state-fil* er bindeleddet mellom den *faktiske infrastrukturen*  og terraformkoden.
* En terraform *backend* er et sted som å lagre terraform state. Det finnes mange implementasjoner, for eksempel S3
* Hvis du *ikke* deklarerer en backend i koden, vil terraform lage en state-fil på din maskin, i samme katalog som du
  kjører terraform fra.

## Lag en fork

Du må start med å lage en fork av dette repoet til din egen GitHub-konto.

![Alt text](img/fork.png  "a title")

## Logg i Cloud 9 miljøet ditt

![Alt text](img/aws_login.png  "a title")

* Logg på med din AWS bruker med URL, brukernavn og passord gitt i klassrommet
* Gå til tjenesten Cloud9 (Du nå søke på Cloud9 uten mellomrom i søket)
* Velg "Open IDE"
* Hvis du ikke ser ditt miljø, kan det hende du har valgt feil region. Hvilken region du skal bruke vil bli oppgitt i klasserommet.

### Lag et Access Token for GitHub

* Når du skal autentisere deg mot din GitHub konto fra Cloud 9 trenger du et access token.  Gå til  https://github.com/settings/tokens og lag et nytt.
* NB. Ta vare på tokenet et sted, du trenger dette senere når du skal gjøre ```git push```

Access token må ha "repo" tillatelser, og "workflow" tillatelser.

![Alt text](img/new_token.png  "a title")

### Lage en klone av din Fork (av dette repoet) inn i ditt Cloud 9 miljø

Fra Terminal i Cloud 9. Klone repositoriet *ditt* med HTTPS URL.

```
git clone https://github.com/≤github bruker>/terraform-app-runner.git
```

Får du denne feilmeldingen ```bash: /terraform-app-runner: Permission denied``` - så glemte du å bytte ut <github bruker> med
ditt eget Github brukernavn :-)

OBS Når du gjør ```git push``` senere og du skal autentisere deg, skal du bruke GitHub  brukernavn, og access token som passord,

For å slippe å autentisere seg hele tiden kan man få git til å cache nøkler i et valgfritt antall sekunder på denne måten;

```shell
git config --global credential.helper "cache --timeout=86400"
```

Konfigurer også brukernavnet og e-posten din for GitHub CLI. Da slipepr du advarsler i terminalen når du gjør commit senere.

````shell
git config --global user.name <github brukernavn>
git config --global user.email <email for github bruker>
````

## Slå på GitHub actions for din fork

I din fork av dette repositoriet, velg "actions" for å slå på støtte for GitHub actions i din fork.

![Alt text](img/7.png "3")

### Lag Repository secrets

* Lag AWS IAM Access Keys for din bruker.  
* Se på .github/workflows/pipeline.yaml - Vi setter hemmeligheter ved å legge til følgende kodeblokk i github actions workflow fila vår slik at terraform kan autentisere seg med vår identitet, og våre rettigheter.


```yaml
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_REGION: eu-west-1
```

### Terraform fra Cloud9 med lokal state fil.

I cloud 9 - Gå til terraform-demo mappen i dette repoet med
```
cd terraform-app-runner/terraform-demo
```

Gå til terraform-demo katalogen. Legg merke til at

* Vi kan ha elementer som "resource", "provider" - og "variabler" i samme fil. Dette er ikke god praksis, men mulig slik som her.
* Terraform bryr seg ikke om filnavn. Alle filer med etternavn ```*.tf``` fra katalogen terraform kjører og ses på samtidig - under ett. 

Utfør  kommandoene

 ```
terraform init 
terraform plan
```  
* Terraform vil spørre deg om bucket name - det et fordi det er en variabel i terraform koden, som heter repo_name, som ikke har noen default verdi
* Avbryt plan (ctrl+c)

Endre ecr.tf og sett en default verdi for variabelen feks; 
```hcl
variable "repo_name" {
  type = string
  default = "<noe personlig>-repo"
}
```

Kjør så bare plan igjen. ````terraform init```` trenger du bare gjøre en gang. Under init laster Terraform ned provider og moduler. (mer om moduler senere)
```sh
terraform plan
```  

Du vil se noe liknende dette;

```shell
  # aws_ecr_repository.myrepo will be created
  + resource "aws_ecr_repository" "myrepo" {
      + arn                  = (known after apply)
      + id                   = (known after apply)
      + image_tag_mutability = "MUTABLE"
      + name                 = "glennbech-demo-3"
      + registry_id          = (known after apply)
      + repository_url       = (known after apply)
      + tags_all             = (known after apply)

      + image_scanning_configuration {
          + scan_on_push = false
        }
    }
```
* Du blir nå ikke bedt om å oppgi et bucket navn, fordi variabelen har en default verdi. 
* Du kan også overstyre variabler fra kommandolinje. Argumenter på kommandolinje har presedens over defaultverdier 
* Forsøk å overstyre variabelnavnet slik 

```sh
terraform plan -var="repo_name=glennbech-mainrepo"                                                       
```
* 
* Og se at Terraform planlegger å lage et ECR repo som heter  *glennbech-mainrepo*
```text
Terraform used the selected providers to generate the following execution plan. Resource
actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_ecr_repository.myrepo will be created
  + resource "aws_ecr_repository" "myrepo" {
      + arn                  = (known after apply)
      + id                   = (known after apply)
      + image_tag_mutability = "MUTABLE"
      + name                 = "glennbech-mainrepo"
      + registry_id          = (known after apply)
      + repository_url       = (known after apply)
      + tags_all             = (known after apply)

      + image_scanning_configuration {
          + scan_on_push = false
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.

```

* Kjør terraform apply *uten å gi variabelnavn på kommandlinjen*, og se at Terraform lager en bucket med samme navn som defaultverdien for variabelen "bucket_name" 
* Du må svare "yes" for å bekrefte, dette funker dårlig i feks GitHub actions, så prøv også derfor 

```sh
terraform apply --auto-approve
```

## state 

* Hvis du slår på visning av skjulte filer i Cloud9 vil du nå se en ````.terraform```` katalog. Denne inneholder en terraform "provider" for AWS (Det som gjør at Terraform kan lage-, endre og slette infrastruktur i AWS) - Disse filene ble lastet ned på ```terraform init```
* Når apply er ferdig, vil du se en terraform.tfstate fil i katalogen du kjørte terrafomr fra. Se på filen. Den inneholder informasjon om ECR repoet du opprettet.
* Åpne state filen, se litt på innholdet
* Slette denne filen, og kjøre terraform apply en gang til. Terraform prøver å opprette ECR repo på nytt, hvorfor?
* Slik informasjon ligger i "state" filen til terraform som du nettopp slettet!
* Gå til Amazon ECR-tjenesten i AWS, og slett det ECR repoet du lagde.

Endre provider.tf ved å legge på en _backend_ blokk, slik at den ser omtrent slik ut, du må modifisere med egent student navn,

```
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "4.33.0"
    }
  }
  backend "s3" {
    bucket = "pgr301-2024-terraform-state"
    key    = "<student-navn>/apprunner-lab.state"
    region = "eu-west-1"
  }
}
```
* Dette er mer robust ! Her forteller vi Terraform at state-informasjon skal lagres i S3, i en Bucket som ligger i Stockholm regionen, med et filnavn du selv bestemmer
  ved å endre "key"
* For å starte med blank ark må du fjerne evt terraform.state, hele .terraform katalogen, og alle filer som starter med ````.terraform````

Deretter utfører du kommandoene

 ```
terraform init 
terraform plan
terraform apply --auto-approve
```  

* Legg merke til at du nå ikke har noe state fil i Cloud9. 
* Gå til AWS tjenesten S3 og se i bucketeten ```pgr301-2024-terraform-state``` at du har fått en fil i buckenten som heter objektnavnet du valgte.
  
## Viktig ! Rydd opp 

* Du skal ikke bruke bruke filer i terraform-demo mappen lenger. Slepp ```terraform-demo``` mappen for å unngå forvirring senere i laben!
  
## AWS App runner & Terraform med GitHub actions

### Lag Repository secrets

* Lag AWS IAM Access Keys for din bruker. NB Du må gjøre dette på nytt, hvis du har gjort dette før.
* Vi setter hemmeligheter ved å legge til følgende kodeblokk i github actions workflow fila vår slik at terraform kan autentisere seg med vår identitet, og våre rettigheter.

```yaml
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_REGION: eu-west-1
```

Vi skal nå se på hvordan vi kan få GitHub actions til å kjøre Terraform for oss. Et utgangspunkt for en workflow ligger i dette repoet. 
Det er par nye nyttige elementer i pipelinen.

Her ser vi et steg i en pipeline med en ```if``` - som bare skjer dersom det er en ```pull request``` som bygges, vi ser også at
pipeline får lov til å _fortsette dersom dette steget feiler.

```yaml
  - name: Terraform Plan
    id: plan
    if: github.event_name == 'pull_request'
    run: terraform plan -no-color
    continue-on-error: true
```

Når noen gjør en Git push til *main* branch, kjører vi ```terraform apply``` med ett flag ```--auto-approve``` som gjør at terraform ikke
spør om lov før den kjører.
```yaml

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve
```

Terraform trenger docker container som lages i en egen GitHub Actions jobb. Vi kan da bruke ```needs``` for å lage en avhengighet mellom en eller flere jobber;

```yaml
  terraform:
    needs: build_docker_image
```


## Finn ditt ECR repository

* Det er laget et ECR repository til hver student som en del av labmiljøet
* Dette heter *studentnavn-private*
* Gå til tjenesten ECR og sjekk at dette finnes

## Gjør nødvendig endringer 

* I rot-katalogen; Endre provider.tf 

```hcl
backend "s3" {
    bucket = "pgr301-2021-terraform-state"
    key    = "<ditt navn eller noe annet unikt>/apprunner-actions.state"
    region = "eu-north-1"
}
```

* se på workflow-filen. 

Som dere ser er "glenn" hardkodet ganske mange steder, bruk ditt eget ECR repository. Endre dette til ditt eget studentnavn

```sh
  docker build . -t hello
  docker tag hello 244530008913.dkr.ecr.eu-west-1.amazonaws.com/glenn:$rev
  docker push 244530008913.dkr.ecr.eu-west-1.amazonaws.com/glenn:$rev
```

## Endre terraform apply linjen

Finn denne linjen, du må endre prefix til å være ditt studentnavn, du må også legge inn studentnavn i image variabelen
for å fortelle app runner hvilken container som skal deployes.

```
 run: terraform apply -var="prefix=<studentnavn>" -var="image=244530008913.dkr.ecr.eu-west-1.amazonaws.com/<studentnavn>-private:latest" -auto-approve
```

## Test

* Når du pusher koden til ditt github repo første gang vil det lages en docker container som lastes opp til ditt ECR repository. Pipeline vil også kjøre terraform, og opprette en App runner service

## Oppgaver

* Når du bygger et container image; push to container images, ett som bruker github commit ($rev) - men også en tag som heter ````:latest````
