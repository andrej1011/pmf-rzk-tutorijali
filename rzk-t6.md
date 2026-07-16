# Docker (Primer)

Pokrećemo ceo sistem veterinarske klinike sa **vežbi 5** pomoću Docker-a. **Kod se ne menja** — menjaju se samo `pom.xml` (custom naziv image-a) i `application.properties` (Eureka adresa). Zatim: build image-a Maven-om, custom network, pokretanje kontejnera, testiranje kroz gateway.

---
## Teorija — šta je Docker i zašto nam treba

Do sada si sistem pokretao tako što si u IntelliJ-u kliknuo **Run** četiri puta (eureka, animal, vet, gateway). Radi — **na tvom laptopu**, sa tvojim JDK-om, tvojim Maven-om, tvojim portovima. Na kolegin računar to prenosiš uz „a jesi instalirao Javu 21? a Maven? a jesi setovao...“.

**Docker** to rešava tako što svaki servis zapakuje u **image** — samodovoljan paket koji sadrži **i aplikaciju i sve što joj treba** (JRE, biblioteke, konfiguraciju). Image je „recept“, statična stvar na disku.

**Kontejner** je **pokrenuta instanca image-a** — živ proces, izolovan od ostatka sistema.

| Pojam | Analogija | Docker komanda |
|---|---|---|
| **Image** | Klasa | `docker images` |
| **Kontejner** | Objekat (instanca klase) | `docker ps` |
| **Network** | Zajednička mreža kontejnera | `docker network create` |

☝️🤓 **Ključna stvar koju moraš da razumeš pre nego što išta ukucaš:**
Svaki kontejner ima **svoj sopstveni `localhost`**. Kad `animal-service` u kontejneru pokuša da se registruje na `http://localhost:8761/eureka`, on traži Eureku **unutar samog sebe** — a tamo je nema. Zato u Docker-u servisi jedan drugog zovu **po imenu kontejnera**, a ne po `localhost`-u. To je jedina konceptualna izmena u ovom tutorijalu; sve ostalo su komande.

---
## Setup

1. **Instaliraj i pokreni Docker Desktop**. Docker **mora da radi u pozadini** — `mvn spring-boot:build-image` bez njega puca sa `Connection to the Docker daemon ... failed`.
2. Proveri iz terminala:
   ```
   docker --version
   docker ps
   ```
   Ako `docker ps` vrati praznu tabelu (a ne grešku) — sve je spremno.
3. **Upali PMF VPN** ako nisi na faksu. Baza `nastava.is.pmf.uns.ac.rs` je i dalje spoljna — ne pravimo MySQL kontejner.
4. **Ugasi sve što ti trenutno radi iz IntelliJ-a** (eureka, animal, vet, gateway). Portovi 8761/8765/8081/8082 moraju biti slobodni, inače će `docker run` pucati sa `port is already allocated`.

---

## Korak 1 — Custom naziv image-a u `pom.xml`

*Bez ovoga bi image dobio ime tipa `animal-service:0.0.1-SNAPSHOT`. Zadatak traži svoje ime.*

U **svakom od 4 projekta**, unutar postojećeg `spring-boot-maven-plugin`, dodaj `<configuration><image><name>`:

**`eureka-server/pom.xml`**
```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <image>
                    <name>rzk/eureka-server:0.1</name>
                </image>
            </configuration>
        </plugin>
    </plugins>
</build>
```

**`api-gateway/pom.xml`**
```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <image>
            <name>rzk/api-gateway:0.1</name>
        </image>
    </configuration>
</plugin>
```

**`animal-service/pom.xml`** — ovde plugin **već ima** `<configuration>` (zbog Lombok-a), pa se `<image>` **dodaje unutar postojećeg** `<configuration>`, ne pravi se drugi:
```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <image>
            <name>rzk/animal-service:0.1</name>
        </image>
        <excludes>
            <exclude>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
            </exclude>
        </excludes>
    </configuration>
</plugin>
```

**`vet-service/pom.xml`** — isto kao animal, samo ime:
```xml
<image>
    <name>rzk/vet-service:0.1</name>
</image>
```

**Anatomija imena `rzk/animal-service:0.1`:**

| Deo | Šta je |
|---|---|
| `rzk` | „repozitorijum“ / namespace — proizvoljan, samo grupiše naše image-e |
| `animal-service` | ime image-a |
| `0.1` | **tag** (verzija). Ako ga izostaviš, Docker podrazumeva `:latest` |

#### Napomene:
- Posle izmene `pom.xml`: **Maven → Sync Project**.
- **Dva `<configuration>` bloka u istom plugin-u = greška.** Ako ti Maven prijavi čudan problem u `animal-service`/`vet-service`, prvo proveri to.
- Tagovi (`0.1`) su korisni: kad promeniš kod i rebild-uješ sa istim tagom, stari image ostaje kao *dangling* (`<none>`) — vidiš ga u `docker images`.

---

## Korak 2 — Eureka po imenu kontejnera (`application.properties`)

*Ovo je jedina „prava“ izmena u konfiguraciji — i najčešći uzrok toga što sistem u Docker-u „nekako ne radi“.*

Do sada su svi servisi ćutke koristili default `http://localhost:8761/eureka`. U Docker-u to **ne radi** (vidi ☝️🤓 iz teorije). Zato u **svakom Eureka klijentu** (animal, vet, gateway — **ne** u samoj Eureki) dodaj:

```properties
eureka.client.service-url.defaultZone=http://eureka-server:8761/eureka
```

Gde je **`eureka-server`** = **ime kontejnera** koji ćemo napraviti u Koraku 5 (`--name eureka-server`). Docker ima ugrađen DNS na custom network-u i to ime razrešava u IP adresu kontejnera.

**`animal-service/src/main/resources/application.properties`**
```properties
spring.application.name=animal-service
server.port=8081

eureka.client.service-url.defaultZone=http://eureka-server:8761/eureka

spring.datasource.url=jdbc:mysql://nastava.is.pmf.uns.ac.rs:3306/rzk
spring.datasource.username=rzk
spring.datasource.password=rzkStudent2019!
spring.jpa.hibernate.naming.physical-strategy=org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl

spring.datasource.hikari.maximum-pool-size=2

resilience4j.retry.instances.animalRecords.max-attempts=5
resilience4j.retry.instances.animalRecords.wait-duration=2s

resilience4j.ratelimiter.instances.allAnimals.limit-for-period=2
resilience4j.ratelimiter.instances.allAnimals.limit-refresh-period=60s

resilience4j.circuitbreaker.instances.animalInfoWithRecords.minimum-number-of-calls=20
resilience4j.circuitbreaker.instances.animalInfoWithRecords.wait-duration-in-open-state=5s
```

**`vet-service/src/main/resources/application.properties`**
```properties
spring.application.name=vet-service
server.port=8082

eureka.client.service-url.defaultZone=http://eureka-server:8761/eureka

spring.datasource.url=jdbc:mysql://nastava.is.pmf.uns.ac.rs:3306/rzk
spring.datasource.username=rzk
spring.datasource.password=rzkStudent2019!
spring.jpa.hibernate.naming.physical-strategy=org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl

spring.datasource.hikari.maximum-pool-size=2
```

**`api-gateway/src/main/resources/application.properties`**
```properties
spring.application.name=api-gateway
server.port=8765

eureka.client.service-url.defaultZone=http://eureka-server:8761/eureka

spring.cloud.gateway.server.webflux.discovery.locator.enabled=true
spring.cloud.gateway.server.webflux.discovery.locator.lower-case-service-id=true

#logovi u konzoli
logging.level.org.springframework.cloud.gateway=TRACE
```

**`eureka-server/src/main/resources/application.properties`** — **nepromenjen**.

#### Napomene ☝️🤓

- **Ime u `defaultZone` MORA da se poklopi sa `--name` iz `docker run`.** Ako kontejner nazoveš `eureka`, a u properties-u ti piše `eureka-server` → servisi se neće registrovati (u logu: `Cannot execute request on any known server`).

- **Port `8761` je port *unutar* kontejnera**, ne onaj koji si mapirao na Mac. Servisi se međusobno zovu preko internih portova — mapiranje (`-p`) je samo za tebe, spolja.

- **Zašto ovo ne kvari pokretanje iz IntelliJ-a?** Zato što `eureka-server` **nije** poznato ime van Docker-a → ako posle pokreneš sistem lokalno, registracija će pucati. Rešenja: ili privremeno vrati `localhost`, ili dodaj u `/etc/hosts` liniju `127.0.0.1 eureka-server`. **Za vežbe 6 radiš isključivo iz Docker-a**, pa te ovo ne mora brinuti — samo znaj zašto.

- **Feign i `lb://animal-service` se NE menjaju.** Oni idu preko Eureke, a Eureka će servise registrovati pod njihovim **hostname-om u kontejneru** — a Docker hostname je (podrazumevano) **isto ono što staviš u `--name`**. Zato je **kritično** da kontejneri nose imena `animal-service`, `vet-service`, `api-gateway`, `eureka-server`.

---

## Korak 3 — Build-ovanje Docker image-a

Iz **IntelliJ terminala**, pozicioniran u folder **svakog** projekta:

```
chmod +x mvnw 
./mvnw spring-boot:build-image -DskipTests
```

Pa isto za ostala tri:
```
cd ../animal-service
chmod +x mvnw 
./mvnw spring-boot:build-image -DskipTests

cd ../vet-service
chmod +x mvnw 
./mvnw spring-boot:build-image -DskipTests

cd ../api-gateway
chmod +x mvnw 
./mvnw spring-boot:build-image -DskipTests
```

Na kraju proveri:
```
docker images
```
Treba da vidiš:
```
REPOSITORY            TAG    IMAGE ID       SIZE
rzk/api-gateway       0.1    ...            ~300MB
rzk/vet-service       0.1    ...            ~350MB
rzk/animal-service    0.1    ...            ~350MB
rzk/eureka-server     0.1    ...            ~330MB
paketobuildpacks/...  ...    ...            ...
```

#### Napomene ☝️🤓

- **Nema `Dockerfile`-a!** Spring Boot koristi **Cloud Native Buildpacks** — sam detektuje da je Java 21, ubaci JRE, optimizuje slojeve. To je poenta `spring-boot:build-image` cilja.
- **Prvi build je spor** (5–10 min) — skida se builder image (`paketobuildpacks/builder`). Sledeći su znatno brži.
- **`-DskipTests`** preskače testove. Bez toga bi Maven pokretao i testove koji se konektuju na bazu — sporo i lako pukne bez VPN-a.
- **Docker Desktop mora da radi.** Ako nije pokrenut: `Connection to the Docker daemon ... failed`.
- **Apple Silicon (M1/M2/M3):** buildpacks će napraviti **ARM64** image — to je ispravno i radiće na tvom Mac-u. Ako Maven ipak zapne sa greškom o `linux/amd64` platformi, dodaj `-Dspring-boot.build-image.platform=linux/arm64` ili u Docker Desktop-u uključi **Rosetta / Use containerd**. Kolegi na Windows/Intel mašini bi trebalo da rebild-uje kod sebe.
- Možeš koristiti i `./mvnw` umesto `mvn` — Maven wrapper koji je Initializr generisao (ne zavisi od toga da li imaš Maven instaliran globalno).
- Ako menjaš kod → **moraš ponovo build-ovati image**. Kontejner ne „vidi“ tvoje izmene u IntelliJ-u.

---

## Korak 4 — Custom network

*Da bi kontejneri mogli da se dozovu po imenu, moraju biti na istoj Docker mreži.*

```
docker network create rzk-network
```

Provera:
```
docker network ls
```

☝️🤓 **Zašto baš „custom“ network, a ne default?**
Na **default** `bridge` mreži Docker-ov ugrađeni DNS **ne razrešava imena kontejnera** — kontejneri se mogu naći samo po IP-ju (koji se menja pri svakom pokretanju). Na **user-defined** mreži (kakvu pravimo ovom komandom) DNS radi, pa `http://eureka-server:8761/eureka` prosto **proradi**. To je jedini razlog zbog kog ovaj korak postoji.

---

## Korak 5 — Pokretanje kontejnera

**Redosled je bitan: Eureka prva.**

### 1) eureka-server
```
docker run -p 8761:8761 --name eureka-server --network rzk-network --rm rzk/eureka-server:0.1
```

Sačekaj da se podigne (log: `Started EurekaServerApplication`), pa otvori **http://localhost:8761** — Eureka dashboard, još prazan.

### 2) animal-service (nov tab u terminalu)
```
docker run -p 8081:8081 --name animal-service --network rzk-network --rm rzk/animal-service:0.1
```

### 3) vet-service (nov tab)
```
docker run -p 8082:8082 --name vet-service --network rzk-network --rm rzk/vet-service:0.1
```

### 4) api-gateway (nov tab)
```
docker run -p 8765:8765 --name api-gateway --network rzk-network --rm rzk/api-gateway:0.1
```

**Anatomija komande:**

| Deo | Značenje |
|---|---|
| `-p 8081:8081` | **port na Mac-u : port u kontejneru**. Bez ovoga servis postoji, ali mu **ne možeš pristupiti sa svog računara** |
| `--name animal-service` | Ime kontejnera — **ujedno i njegov hostname** na mreži. **Mora** da se poklopi sa `defaultZone` i sa `@FeignClient("...")` / `lb://...` |
| `--network rzk-network` | Priključi ga na našu mrežu (bez ovoga ne vidi Eureku) |
| `--rm` | Obriši kontejner **automatski** kad ga zaustaviš (Ctrl+C). Zgodno — inače se gomilaju |
| `rzk/animal-service:0.1` | Koji image pokrećemo |

**Provera:**
```
docker ps
```
→ 4 kontejnera u statusu `Up`. I na **http://localhost:8761** sada moraju biti registrovani **ANIMAL-SERVICE, VET-SERVICE, API-GATEWAY**.

#### Napomene ☝️🤓

- **Sačekaj ~30s** posle pokretanja pre nego što proveriš Eureku — registracija nije trenutna (heartbeat interval).
- **`--rm` vs. bez njega:** sa `--rm` kontejner nestaje po gašenju. Ako ga izostaviš, kontejner ostaje u `docker ps -a` i moraš `docker rm <ime>` pre nego što ponovo pokreneš isto ime — inače: `Conflict. The container name "/animal-service" is already in use`.
- **Kontejneri se pokreću u foreground-u** — svaki „zauzme“ terminal i ispisuje logove. Otvori **4 taba**. (Ili dodaj `-d` za detached, pa logove gledaj sa `docker logs -f animal-service`.)
- **Gašenje:** `Ctrl+C` u tabu, ili `docker stop <ime>` iz drugog tab-a.
- **Baza:** kontejner ka `nastava.is.pmf.uns.ac.rs` izlazi kroz mrežu tvog Mac-a, pa **VPN mora biti upaljen**. Ako u logu animal/vet servisa vidiš `Communications link failure` ili `Unknown host` — VPN je kriv, ne Docker.
- **Redosled:** ako pokreneš animal pre Eureke, Docker DNS neće naći `eureka-server` i servis će logovati greške registracije. Nije fatalno (retry-uje se), ali zašto komplikovati.

---

## Korak 6 — Testiranje API-ja

**Sve isto kao sa vežbi 5** — ništa se u kodu nije promenilo, pa se ni endpoint-i nisu promenili. Testiraj kroz **Talend API**, na **gateway-u**:

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
→ **200 OK** + lista svih životinja *(dokaz: gateway kontejner → Eureka kontejner → animal-service kontejner → spoljna baza)*

**3) Rutiranje na vet-service:**
```
GET http://localhost:8765/api/vet-records/1
Header: x-api-key: validKey123
```
→ lista kartona

**4) Feign između kontejnera:**
```
GET http://localhost:8765/api/animals/animal-info-with-records/1
Header: x-api-key: validKey123
```
→ **200** + `{ "animal": {...}, "vetRecords": [...] }`
Ovo je **najbolji test** — dokazuje da `animal-service` kontejner uspešno preko Eureke pronašao i pozvao `vet-service` kontejner.

**5) Resilience4J i dalje radi:**
```
docker stop vet-service
```
Pa:
```
GET http://localhost:8765/api/animals/animal-records/1
Header: x-api-key: validKey123
```
→ čeka ~8s (Retry: 5 pokušaja × 2s), pa **408** + `[]`. U logu animal-service kontejnera `API call received!` **5 puta**.

☝️🤓 Ako si koristio `--rm`, `docker stop vet-service` ga i **obriše** — da ga vratiš, samo ponovo pokreni `docker run` komandu za vet.

---

## Docker cheat sheet

| Šta hoću | Komanda |
|---|---|
| Napravi image (iz foldera projekta) | `mvn spring-boot:build-image -DskipTests` |
| Svi image-i | `docker images` |
| Napravi mrežu | `docker network create rzk-network` |
| Sve mreže | `docker network ls` |
| Pokreni kontejner | `docker run -p 8081:8081 --name animal-service --network rzk-network --rm rzk/animal-service:0.1` |
| **Pokrenuti** kontejneri | `docker ps` *(ili `docker container ls`)* |
| **Svi** kontejneri (i zaustavljeni) | `docker ps -a` *(ili `docker container ls -a`)* |
| Logovi kontejnera | `docker logs -f animal-service` |
| Zaustavi kontejner | `docker stop animal-service` |
| Obriši kontejner | `docker rm animal-service` *(ili `docker container rm ...`)* |
| Obriši image | `docker rmi rzk/animal-service:0.1` *(ili `docker image rm ...`)* |
| Uđi u kontejner (debug) | `docker exec -it animal-service sh` |

---

## Šta se desilo pod haubom

Put zahteva `GET localhost:8765/home` sa `x-api-key: validKey123`, sad kad sve radi u Docker-u:

1. **Talend → tvoj Mac, port 8765.** Docker vidi da je `8765` mapiran (`-p 8765:8765`) na kontejner `api-gateway` i prosleđuje mu paket **unutar `rzk-network`**.
2. **api-gateway kontejner:** globalni filteri (`LoggingFilter`, `AuthenticationFilter`) → ključ ispravan → `RewritePath` prepiše `/home` u `/api/animals`.
3. **Razrešavanje `lb://animal-service`:** gateway pita Eureku. Ali gde je Eureka? U `application.properties` piše `http://eureka-server:8761/eureka` → **Docker DNS** na `rzk-network` razreši ime `eureka-server` u IP kontejnera (npr. `172.18.0.2`).
4. **Eureka odgovara:** „ANIMAL-SERVICE je registrovan na hostname-u `animal-service`, port 8081“. *(Hostname je takav jer smo kontejner pokrenuli sa `--name animal-service`.)*
5. **Gateway → animal-service:** poziv na `http://animal-service:8081/api/animals` — opet preko Docker DNS-a, **direktno između kontejnera**, bez izlaska na tvoj Mac.
6. **animal-service kontejner:** `RateLimiter` propušta → `AnimalService` → `AnimalRepository.findAll()` → **Hibernate** otvara JDBC konekciju ka `nastava.is.pmf.uns.ac.rs:3306` — to je **jedini poziv koji izlazi izvan Docker mreže**, kroz tvoj Mac i **VPN**.
7. **Nazad:** JSON → gateway kontejner → port 8765 → Talend.

☝️🤓 Obrati pažnju: portovi **8081** i **8082** u koracima 4–5 su **interni portovi kontejnera**. Sistem bi radio i da ih nikad nisi mapirao sa `-p` — mapirali smo ih **samo** da bismo mogli da testiramo servise direktno, mimo gateway-a.

---

## Rekapitulacija

### Šta se promenilo u odnosu na vežbe 5

| | Vežbe 5 (IntelliJ) | Vežbe 6 (Docker) |
|---|---|---|
| Pokretanje | 4× klik na **Run** | 4× `docker run` |
| Eureka adresa | default `localhost:8761` | **`eureka-server:8761`** (ime kontejnera) |
| `pom.xml` | — | + `<image><name>rzk/...:0.1</name></image>` |
| Java kod | — | **nijedna izmena** ✅ |
| Baza | spoljna, MySQL na PMF-u | **isto** (nema MySQL kontejnera) |
| Testiranje | Talend, `localhost:8765` | **isto** |

### Image-i i kontejneri

| Servis | Image | Kontejner (`--name`) | Port |
|---|---|---|---|
| eureka-server | `rzk/eureka-server:0.1` | `eureka-server` | 8761 |
| animal-service | `rzk/animal-service:0.1` | `animal-service` | 8081 |
| vet-service | `rzk/vet-service:0.1` | `vet-service` | 8082 |
| api-gateway | `rzk/api-gateway:0.1` | `api-gateway` | **8765** |

**Mreža:** `rzk-network` (svi kontejneri).

### Tri imena koja MORAJU da se poklope

Ovo je najčešći izvor grešaka — sva tri se odnose na istu stvar:

1. **`--name eureka-server`** u `docker run`
2. **`eureka.client.service-url.defaultZone=http://eureka-server:8761/eureka`** u properties-u
3. **`spring.application.name`** servisa ↔ **`@FeignClient("vet-service")`** / **`lb://animal-service`** ↔ **`--name`** kontejnera

Ako se bilo koje razlikuje → „radi lokalno, ne radi u Docker-u“.

---

## Struktura projekta

```
vet-clinic/
├── eureka-server/
│   ├── pom.xml                      ← + <image><name>rzk/eureka-server:0.1</name>
│   └── src/main/resources/application.properties    (nepromenjen)
│
├── api-gateway/
│   ├── pom.xml                      ← + <image><name>rzk/api-gateway:0.1</name>
│   └── src/main/resources/application.properties    ← + eureka defaultZone
│
├── animal-service/
│   ├── pom.xml                      ← + <image> UNUTAR postojećeg <configuration>
│   └── src/main/resources/application.properties    ← + eureka defaultZone
│
└── vet-service/
    ├── pom.xml                      ← + <image> UNUTAR postojećeg <configuration>
    └── src/main/resources/application.properties    ← + eureka defaultZone
```

**Java kod: 0 izmena.** Svi kontroleri, servisi, Feign proxy-ji, filteri i konfiguracija ruta ostaju **identični** onima sa vežbi 5.

---

## Checklist pred odbranu ✅

- [ ] **Docker Desktop pokrenut**, **PMF VPN upaljen**
- [ ] Sva 4 `pom.xml` imaju `<image><name>rzk/...:0.1</name></image>`
- [ ] Sva 3 Eureka klijenta imaju `defaultZone=http://eureka-server:8761/eureka`
- [ ] `mvn spring-boot:build-image -DskipTests` × 4 → `docker images` pokazuje 4 `rzk/*` image-a
- [ ] `docker network create rzk-network`
- [ ] `docker run` × 4 (Eureka prva!) → `docker ps` pokazuje 4 kontejnera `Up`
- [ ] `http://localhost:8761` → registrovani ANIMAL-SERVICE, VET-SERVICE, API-GATEWAY
- [ ] `GET localhost:8765/home` **bez** ključa → **403**, **sa** ključem → **200** + životinje
- [ ] `GET localhost:8765/api/animals/animal-info-with-records/1` sa ključem → **200** *(Feign između kontejnera radi!)*

### Najčešće greške

| Simptom | Uzrok |
|---|---|
| `Cannot execute request on any known server` | `defaultZone` ≠ `--name` kontejnera, ili kontejneri nisu na istoj mreži |
| `port is already allocated` | Servis ti još radi iz IntelliJ-a — ugasi ga |
| `The container name "/x" is already in use` | Stari kontejner nije obrisan → `docker rm x` (ili koristi `--rm`) |
| `Communications link failure` (MySQL) | **VPN nije upaljen** |
| `Connection to the Docker daemon failed` | Docker Desktop nije pokrenut |
| Gateway vraća **503 Service Unavailable** | Servis nije (još) registrovan na Eureki — sačekaj 30s, pa proveri 8761 |
