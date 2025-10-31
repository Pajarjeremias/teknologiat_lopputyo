# Lopputyö
Tehtävän tavoitteena on käsitellä erilaisia nyky ajan teknologioita. Kaikki dokumentoimieni teknologiat ovat toteutettu Ohjelmistoprojekti 2 kurssin, ryhmäni tekemään työhön Varustevahtiin, joka on mobiilissa toimiva kuvan tunnistus ohjelma.

## Docker
Minun tehtäväkseni muodostui DevOps statukseni vuoksi Backendin laittaminen palvelimelle. Meidän Backendissä on mukana AI_Model joka tunnistaa sille opetettujen mallien perusteella kuvan, jonka käyttäjä joko valitsee galleriasta tai ottaa kännykän kameralla.

Aluksi piti valita miten lähtisin toteuttamaan annettua tehtävää, joten kävin läpi eri alustat joille palvelin puolen saisi pystyyn ja päädyin Renderiin, jonkin asteisen tietämyksen ja sen jatko kehittämisen vuoksi. Itse backend on toteutettu mySQL:llä pythonia ja FastApia käyttäen. Tietokanta ja sen sisältä data on kevyttä eikä sisällä montaakaan riviä koodia mutta itse kuvantunniste ohjelma on raskas, jonka vuoksi kävin eri vaihtoehtoja läpi, miten saisin mahdollisimman kevyen base imagen, jotta Docker imagesta ei tulisi liian raskas, lisätessäni Varustevahdin Backendin sinne. Tutkin eri vaihtoehtoja, kuten Pytorchia varten CUDA ajurin image tai perus ubuntu johon dockerfilessä asentaisin vain kaiken mutta valitsin kuitenkin kokeiluun python:3.12-slim, sillä luin artikkelin missä kehuttiin python 3.12 ja slim vaihtoehdon otin että se olisi riisutumpi.

### Vaihe 1 Requirements.py
Tarkistin että requirements.py:ssä on yhdessä kaikki tarvittavat tiedot  ladattavista kirjastoista, sillä AI_Modelissa ja Backendissä on omat readme.md. päädyin siihen että juuri hakemistossa olevaan requirementsiin lisäsin AI_Modelissa olevat requirementsit, jotta dockerfile löytää sen paremmin, tai oikeastaan en jaksanut alkaa sen enempään kikkailemaan sijaintien kanssa, koska se on ennenkin osoittautunut epäsuotuisaksi. 

### Vaihe 2 Dockerfile
Täytyi miettiä mitä kaikkea tarvitsen. Tärkeinpänä on python base image, jossa päädyin versioon 3.12 ja siitä slim, jotta olisi vähän riisutumpi versio, sillä tiesin että ai model tulee lisäämään imagen kokoa paljon. Alpine ei olisi käynyt sillä  Koko ajan oli polttava tunne että tulisi otettava enemmänkin pois ja löysin esimerkiksi ratkaisun, jossa linuksin päivittämisen ja asentamisen yhteydessä lisää lipun --no-install-recommends, tämä toimii niin että, jokaisessa paketissa on 3 erilaista riippuvuus pakettia: required, recommended ja suggested. Eli lipun tarkoitus on olla asentamatta recommended osiota, joka keventää Ubuntun blogin mukaan Docker imagea, jopa 60 % (lähde: https://ubuntu.com/blog/we-reduced-our-docker-images-by-60-with-no-install-recommends). Kirjoitin Googleen Python dockerfile löysin docker best practices (https://docs.docker.com/build/building/best-practices/#run), jota hyödynsin ja otin sieltä mallia tarvittavista imagen rakennus tarvikkeista. 
Ensimäisellä image buildilla se meni läpi mutta kun yritin ajaa sitä tuli RunTimeError: Form data requires "Python multi-part" to be installed. Eli täytyy lisätä requirements.txt tiedostoon python multipart ja kokeilla uudestaa buildia. Sain imagen toimimaan menemällä osoitteeseen: http://127.0.0.1:8080/docs
Varustevahdin imagessa saattaa olla vielä vähän sistitävää, sillä sen kooksi muodostui 12.97 GB.

<img width="685" height="427" alt="ohkeat_lopputyo" src="https://github.com/user-attachments/assets/ce59443c-9304-4faa-9a8f-71297a4cdfb6" />

Kuten kuvasta näkee, buildissa kestää pitkään. 1 virhe oli, joka liittyi vain välilyöntiin dockerfilen yhdessä komennoista
```dockerfile
# Base image to use, fewer included packages, because we need to keep image lighter for AI_Model and database
FROM python:3.12-slim

# All commands about to happen happens in this directory
WORKDIR /app

# Linux commads, for upgrade and install, flags first mean yes and second not to install "recommended dependencies packages" making it lighter. 
# Source: "https://ubuntu.com/blog/we-reduced-our-docker-images-by-60-with-no-install-recommends"
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    # Needed if pip-packages needs to be converted and git, because open_clip_torch gets model weigths from Github
    git \
    && rm -rf /var/lib/apt/lists/*
    # Deletes caches, for again making image lighter (source: https://docs.docker.com/build/building/best-practices/#run)
(ei toimi tälläisenään, kommentit pitää siirtää pois tieltä)

# Copying file to image, so pip can install
COPY requirements.txt .

# install and upgrade pip and then install requirements file
RUN python -m pip install --upgrade pip && pip install -r requirements.txt

# copy app to image
COPY app ./app

# copy AI_MODEL to image
COPY AI_Model ./AI_Model

# making folder for save_upload to save pictures, -p flag creates them if they do not already exist.
RUN mkdir -p /app/uploads

#jälkeenpäin jouduin lisäämään
    \
 && chgrp -R 0 /app \
 && chmod -R g=u /app

# Database path
ENV DATABASE_URL=sqlite:///./varustevahti.db

# Using uvicorn to start application, listening everywhere and use Render given port or if not given then port 8000
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--reload", "--host", "0.0.0.0", "--port", "8000"]
```
### vaihe 3 Palvelin
Olin julkaisemassa Varustevahdin backendiä Renderille, kunnes tajusin että backend kuvantunnistus ohjelmineen ei tule pyörimään ilmais version 512 mb RAMilla ja 0.1 CPUlla. Muistin ohjelmistoprojketi kurssilla Markku Ruonavaaran maininneen CSCn tarjoamien palveluiden, kuten Rahti tarjoavan opiskelijoille paljon teknisiä resursseja, joten päätin kokeilla tätä. Onnistuin luomaan projektin, mutta loin sen aluksi käyttäen Pythonin builder imagea, kunnes tajusin ettei se missään kohtaa lue Docker imageani. Tämän tajuttuani loin uuden projektin käytäen Dockerfileä, kaatui heti ja huomasin käyttäneeni pientä d kirjainta, korjasin asian ja build onnistui mutta podin käynnistyksessä ilmeni ongelma "AvailableFalseMinimumReplicasUnavailableDeployment does not have minimum availability". Podin logi kertoi ettei se pysty avaamaan ja antoi samalla taustatieto sivuston vurheelle. Sieltä ei löytynyt mitään minua auttavaa, joten etsin haku koneista virhe ilmoituksella mutta mitään ei löytynyt, jolloin menin CSCn FAQ Rahti osioon, josta kävin kaikki kysytyt kysymyset läpi ja tietenkin viimeisenä oli kysymys "Why this container report permission denied errors?" joka sisälsi tekstin:
"The most common reason for a container to fail to run in Rahti, is that the container image expects to be run as root. As Rahti does not run images as root, permission denied errors will stop the execution" ja toinen teksti "the other option is to modify the current image to make it work with a non root user, for example as described in the Creating images section."(source: https://docs.csc.fi/support/faq/why-this-container-does-not-work/). Creating image section ei auttanut minua sillä siellä käytettiin Alpinea joka käyttää työkaluna aptn sijasta kevyempää apk paketin hallinta työkalua, eikä se ole minulle tuttu. Eli siis Rahti ei anna ajaa imageja root käyttäjänä ja lopettaa toimeksiannon, jolloin tajusin että minun tulee Dockerfilessä vaihtaa /app hakemiston käyttäjä, sillä siellä on SQLite tiedostot. Käyttäjää en pystyisi vaihtaa pois Rootista sillä Rahdin käyttämä Kubernetesin OpenShift estää turvallisuus syistä Rootina olemisen vaihtelemalla käyttäjätunnisteita(UID) tietyn numero skaalan välillä(source: https://devops.pages.helsinki.fi/guides/tike-container-platform/instructions/uids-openshift.html). Joten minun tulisi muuttaa ryhmää, löysin RedHatBlogin, jossa kerrotaan että jokainen group on 0, joka on root group(source: https://www.redhat.com/en/blog/a-guide-to-openshift-and-uids ja kohdan UIDs and GUIs in Action (Container perspective) lopussa.). eli vaihdoin app hakemiston ryhmäksi 0 eli root group (chgrp -R 0 / app), jonka jälkeen annoin tälle ryhmälle samat oikeudet kuin omistajalle (chmod -R g=u /app). Pohdiskelu apua ja henkistä tukea aiheesta sain pikku serkultani Full Stack Developer Adam Wallinilta mutta tiedon haun ja lopullisen selvityksen tein itse. Tämän jälkeen Backend rupesin toimimaan osoitteessa: https://backend-git-varustevahti.2.rahtiapp.fi/

## CI/CD Pipeline

### Github actions

### Eas workflows

## Yksikkötestaus
