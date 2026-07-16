# Docker Compose 

Isti sistem veterinarske klinike, isti image-i sa vežbi 6 — ali umesto četiri `docker run` komande i četiri terminal taba, sve opisujemo u **jednom `docker-compose.yaml`** fajlu i dižemo jednom komandom. Na kraju: postavljanje image-a na **Docker Hub**.

---
## Teorija — zašto Compose

Sa vežbi 6 pokretanje sistema izgleda ovako:

```
docker network create rzk-network
docker run -p 8761:8761 --name eureka-server  --network rzk-network --rm rzk/eureka-server:0.1
docker run -p 8081:8081 --name animal-service --network rzk-network --rm rzk/animal-service:0.1
docker run -p 8082:8082 --name vet-service    --network rzk-network --rm rzk/vet-service:0.1
docker run -p 8765:8765 --name api-gateway    --network rzk-network --rm rzk/api-gateway:0.1
```

Pet komandi, četiri terminal taba, i moraš da pamtiš redosled. **Docker Compose** sve to opisuje **deklarativno** u YAML fajlu:

```
docker compose up
```

Jedna komanda. Compose sam napravi mrežu, pokrene kontejnere pravim redosledom, i skupi im logove u jedan terminal.

☝️🤓 **Compose ne pravi image-e** (bar ne kod nas). On samo **pokreće** slike koje već postoje — one koje si build-ovao sa `./mvnw spring-boot:build-image` na vežbama 6. Zato u zadatku i piše *„nije potrebno ako su već napravljene slike“*.

☝️🤓 **`docker run` ↔ `docker-compose.yaml` — sve je 1:1:**

| `docker run` | Compose |
|---|---|
| `rzk/animal-service:0.1` | `image: rzk/animal-service:0.1` |
| `-p 8081:8081` | `ports: - "8081:8081"` |
| `--name animal-service` | **naziv servisa** u YAML-u |
| `--network rzk-network` | `networks: - rzk-network` |
| *(ručni redosled pokretanja)* | `depends_on:` |
| *(nema)* | `mem_limit:` |
| *(nema)* | `environment:` |

---

Setup

1. **Docker Desktop pokrenut**, **PMF VPN upaljen** (baza je i dalje spoljna).
2. Proveri da image-i postoje:
   ```
   docker images
   ```
   → moraju biti `rzk/eureka-server:0.1`, `rzk/animal-service:0.1`, `rzk/vet-service:0.1`, `rzk/api-gateway:0.1`.
   *Ako ih nema — vrati se na vežbe 6, korak „build image“.*
3. **Ugasi sve kontejnere sa vežbi 6** (Compose pravi svoje):
   ```
   docker stop $(docker ps -q)
   ```
4. Staru mrežu ne moraš da brišeš — Compose pravi **svoju**.

---

## Korak 1 — `docker-compose.yaml`

Napravi fajl **`docker-compose.yaml`** u **root folderu projekta** (`vet-clinic/`, pored foldera servisa — **ne** unutar nekog od njih):

```yaml
services:

  eureka-server:
    image: rzk/eureka-server:0.1
    ports:
      - "8761:8761"
    networks:
      - rzk-network
    mem_limit: 700m

  animal-service:
    image: rzk/animal-service:0.1
    ports:
      - "8081:8081"
    networks:
      - rzk-network
    depends_on:
      - eureka-server
    environment:
      - eureka.client.service-url.defaultZone=http://eureka-server:8761/eureka
    mem_limit: 700m

  vet-service:
    image: rzk/vet-service:0.1
    ports:
      - "8082:8082"
    networks:
      - rzk-network
    depends_on:
      - eureka-server
    environment:
      - eureka.client.service-url.defaultZone=http://eureka-server:8761/eureka
    mem_limit: 700m

  api-gateway:
    image: rzk/api-gateway:0.1
    ports:
      - "8765:8765"
    networks:
      - rzk-network
    depends_on:
      - eureka-server
      - animal-service
      - vet-service
    environment:
      - eureka.client.service-url.defaultZone=http://eureka-server:8761/eureka
    mem_limit: 700m

networks:
  rzk-network:
```

### Šta koji ključ radi

| Ključ | Značenje |
|---|---|
| `services:` | Lista svega što treba pokrenuti. **Naziv servisa** (`animal-service`) postaje **ime kontejnera i hostname na mreži** — isto što je ranije radio `--name` |
| `image:` | Koju sliku pokrećemo (`rzk/animal-service:0.1`) |
| `ports:` | `"port_na_Mac-u:port_u_kontejneru"`. **Navodnici su obavezni** — bez njih YAML ume da protumači `8081:8081` kao vreme (sexagesimal), pa dobiješ čudne greške |
| `networks:` | Na koju mrežu se kontejner priključuje |
| `networks:` *(na dnu, van `services`)* | **Definicija** mreže. Prazna vrednost = default `bridge` tip |
| `depends_on:` | Redosled pokretanja — Compose ovaj servis pokreće **posle** navedenih |
| `environment:` | Environment varijable koje se ubacuju u kontejner — **pregaze** vrednosti iz `application.properties` |
| `mem_limit:` | Maksimum RAM-a za kontejner (`700m` = 700 MB) |

### Napomene ☝️🤓

- **`depends_on` čeka da kontejner **krene**, ne da bude **spreman**.** Compose će pokrenuti `eureka-server` pre `animal-service`-a, ali **neće čekati** da se Eureka u potpunosti podigne (Spring Boot startuje 10–20s). U praksi to nije problem: Eureka klijenti sami retry-uju registraciju, pa se za ~30s sve složi. Ako u logu vidiš `Cannot execute request on any known server` na početku — **normalno je**, samo sačekaj.

- **`environment` je redundantno kod nas** — `defaultZone` već stoji u `application.properties` (vežbe 6). Ali je **bolja praksa** i profesorka ga baš tako pokazuje: image ostaje „čist“ i **prenosiv**, a adresa Eureke se zadaje spolja, pri pokretanju. Ako sutra Eureka kontejner nazoveš drugačije, menjaš **samo** compose fajl — ne moraš da rebild-uješ image.

- **Zašto tačke rade u `environment`?** `eureka.client.service-url.defaultZone` sa tačkama nije klasičan format env varijable — ali Spring Boot koristi **relaxed binding** i uredno ga prepozna. *(Klasičan oblik `EUREKA_CLIENT_SERVICEURL_DEFAULTZONE` takođe radi.)*

- **`mem_limit: 700m`** nije obavezan, ali je pametan: 4 JVM-a ume da pojedu popriličan RAM. Ako ti kontejner puca sa `OOMKilled` u `docker ps -a`, **povećaj limit** (npr. `1g`), ne smanjuj.

- **Ime fajla:** `docker-compose.yaml` ili `docker-compose.yml` — oba rade. IntelliJ oba prepoznaje i nudi zeleno dugme ▶️ pored `services:`.

- **Nema `version:` na vrhu.** Nekad je bilo obavezno (`version: "3.8"`), danas je **deprecated** — Docker izbacuje upozorenje ako ga staviš. Prezentacija ga takođe nema. ✅

---

## Korak 2 — Pokretanje

Iz foldera gde je `docker-compose.yaml`:

```
docker compose up
```

Logovi svih servisa idu **u isti terminal**, obojeni i prefiksovani imenom servisa:
```
eureka-server   | Started EurekaServerApplication in 8.4 seconds
animal-service  | Started AnimalServiceApplication in 11.2 seconds
vet-service     | Started VetServiceApplication in 10.9 seconds
api-gateway     | Started ApiGatewayApplication in 9.1 seconds
```

**U pozadini (detached), da ti terminal ostane slobodan:**
```
docker compose up -d
```
Pa logove gledaj sa:
```
docker compose logs -f
docker compose logs -f animal-service     # samo jedan servis
```

**Provera:**
```
docker compose ps
```
→ 4 servisa, status `running`. Ili klasično `docker ps`.

**Eureka dashboard:** `http://localhost:8761` → sačekaj ~30s → moraju biti registrovani **ANIMAL-SERVICE, VET-SERVICE, API-GATEWAY**.

### Iz IntelliJ-a

Otvori `docker-compose.yaml` → klikni **zeleno ▶️** pored `services:` (diže sve) ili pored pojedinačnog servisa (diže samo njega + zavisnosti). Logovi idu u **Services** tab.

☝️🤓 **`docker compose` (razmak) vs `docker-compose` (crtica).** Prezentacija koristi stari `docker-compose up` — to je zaseban Python alat (Compose V1), koji je **povučen**. Moderni Docker Desktop ima Compose ugrađen kao **plugin**, pa je ispravna komanda `docker compose up`. Obe verovatno rade kod tebe (Docker Desktop drži alias), ali koristi verziju sa **razmakom**.

---

## Korak 3 — Testiranje API-ja

Identično kao na vežbama 6 — kod se nije menjao. Talend, na gateway-u:

**1) 403 bez ključa:**
```
GET http://localhost:8765/home
```
→ **403 Forbidden**

**2) 200 sa ključem:**
```
GET http://localhost:8765/home
Header: x-api-key: validKey123
```
→ **200 OK** + sve životinje

**3) Vet servis:**
```
GET http://localhost:8765/api/vet-records/1
Header: x-api-key: validKey123
```
→ kartoni

**4) Feign između kontejnera (krunski test):**
```
GET http://localhost:8765/api/animals/animal-info-with-records/1
Header: x-api-key: validKey123
```
→ **200** + `{ "animal": {...}, "vetRecords": [...] }`

**5) Resilience4J i dalje radi:**
```
docker compose stop vet-service
```
Pa `GET /api/animals/animal-records/1` → čeka ~8s (Retry) → **408** + `[]`.
Vrati servis:
```
docker compose start vet-service
```

☝️🤓 Za razliku od `docker stop` sa `--rm` (koji kontejner i **obriše**), `docker compose stop` ga samo **zaustavi** — pa ga `docker compose start` vraća bez ponovnog pravljenja. Zgodno za demonstraciju Circuit Breaker-a.

---

## Korak 4 — Gašenje

**Zaustavi i obriši kontejnere + mrežu:**
```
docker compose down
```

**Samo zaustavi (kontejneri ostaju, mogu se vratiti sa `start`):**
```
docker compose stop
```

☝️🤓 `docker compose down` **ne briše image-e** — `rzk/*:0.1` ostaju na disku. Sledeći put je `docker compose up` gotov za par sekundi.

Ako si pokrenuo sa `up` (bez `-d`), dovoljan je i **`Ctrl+C`** u terminalu.

---
## Docker Hub

*Docker Hub je „GitHub za image-e“ — javni registry sa kog svako može da preuzme tvoju sliku. Do sada su image-i postojali samo lokalno.*

### 1) Napravi nalog
`https://hub.docker.com/` → zapamti **username** (npr. `andrej`).

### 2) Prijavi se iz terminala
```
docker login
```
→ traži username i password (ili te odvede na browser). Uspeh: `Login Succeeded`.

### 3) Nađi ID slike
```
docker images
```
```
REPOSITORY            TAG    IMAGE ID       SIZE
rzk/animal-service    0.1    afd227db42c3   ...
```

### 4) Dodaj novi tag
Docker Hub zahteva da ime image-a počinje tvojim **username-om**, pa lokalni `rzk/animal-service:0.1` treba „prekrstiti“:

```
docker tag afd227db42c3 andrej/animal-service:0.1
```
Format: `docker tag <id_image-a> <username>/<repository>:<tag>`

☝️🤓 **Tag ne pravi kopiju!** Slika ostaje ista (isti `IMAGE ID`), samo dobija **drugo ime**. U `docker images` ćeš videti dva reda sa **istim ID-jem** — to je jedna slika sa dva imena.

### 5) Push
```
docker push andrej/animal-service:0.1
```
Otvori `https://hub.docker.com/` → slika je tamo, javno dostupna.

### 6) Pull (sa bilo koje mašine)
```
docker pull andrej/animal-service:0.1
```

### 7) Odjava
```
docker logout
```

☝️🤓 **Ako želiš da ceo sistem bude „pull-abilan“**, ponovi tag+push za sva 4 servisa, pa u `docker-compose.yaml` zameni `image: rzk/...` sa `image: andrej/...`. Tada kolega **ne mora ništa da build-uje** — dovoljno mu je da uzme tvoj compose fajl i kucne `docker compose up`. To je i cela poenta priče.

⚠️ Push-uješ **javno**. Ne stavljaj u image ništa osetljivo — a lozinka baze **jeste** u `application.properties`, koji je unutar image-a. Za vežbe je ok, samo budi svestan.

---
## Compose cheat sheet

| Šta hoću | Komanda |
|---|---|
| Digni sve | `docker compose up` |
| Digni sve u pozadini | `docker compose up -d` |
| Status | `docker compose ps` |
| Logovi (svi / jedan) | `docker compose logs -f` / `docker compose logs -f animal-service` |
| Zaustavi jedan servis | `docker compose stop vet-service` |
| Vrati ga | `docker compose start vet-service` |
| Restartuj jedan | `docker compose restart animal-service` |
| Zaustavi sve | `docker compose stop` |
| **Zaustavi i obriši** (kontejneri + mreža) | `docker compose down` |
| Uđi u kontejner | `docker compose exec animal-service sh` |

**Docker Hub:**

| Šta hoću | Komanda |
|---|---|
| Prijava | `docker login` |
| Novi tag | `docker tag <image_id> <username>/<repo>:<tag>` |
| Push | `docker push <username>/<repo>:<tag>` |
| Pull | `docker pull <username>/<repo>:<tag>` |
| Odjava | `docker logout` |

---

## Šta se desilo pod haubom

Kad kucneš `docker compose up`:

1. **Compose parsira `docker-compose.yaml`** i pravi mrežu `rzk-network` — tačnije, **`vet-clinic_rzk-network`**. Compose prefiksuje imena **imenom foldera** u kom je fajl (to je „projekat“). Zato `docker network ls` pokazuje duže ime nego što si napisao — a i dalje sve radi, jer su svi kontejneri na **istoj** toj mreži.
2. **Sortira servise po `depends_on`:** `eureka-server` → `animal-service`, `vet-service` → `api-gateway`.
3. **Za svaki servis pokreće kontejner** iz navedenog image-a, sa `--name` = naziv servisa, mapiranim portovima, `mem_limit`-om i env varijablama.
4. **Docker DNS** na toj mreži razrešava `eureka-server` u IP kontejnera → animal/vet/gateway se registruju.
5. **Odatle je sve isto kao na vežbama 6:** klijent → `localhost:8765` → gateway kontejner → filteri (`x-api-key`) → Eureka → `lb://animal-service` → animal kontejner → *(Feign)* → vet kontejner → MySQL na PMF-u (kroz VPN).

☝️🤓 Compose **ne menja ništa** u ponašanju sistema — on je samo **udobniji način** da se pokrene ono što si već imao. Sve iz vežbi 6 (rutiranje, filter, Resilience4J, Feign, load balancing) radi **identično**.

---