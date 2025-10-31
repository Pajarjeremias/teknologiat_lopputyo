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




# syntax=docker/dockerfile:1
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
    # Deletes caches, for again making image lighter
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

# Using uvicorn to start application, listening everywhere and use Render given port or if not given then port 8000
ENV PORT=8000
CMD ["uvicorn", "app.main:app", "--reload", "--host", "0.0.0.0", "--port", "8000"]

### vaihe 3 Render


## CI/CD Pipeline

### Github actions

### Eas workflows

## Yksikkötestaus
