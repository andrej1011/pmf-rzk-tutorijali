
Feign, Naming Server, Load Balancing

# Vežbe 4 Primer (proširenje sistema sa vežbi 3)
---

Sistem sa vežbi 3 (config-server + horse-management + analytics) menjamo na dva mesta:

1. **Dodajemo `eureka-server`** (port 8761) — imenik u koji se servisi sami prijavljuju. Umesto da znaš gde je ko, pitaš imenik.
2. **U analytics-u bacamo `RestClient` i stavljamo `Feign`** — poziv horse-managementu više ne ide na zakucan `localhost:8080`, nego **po imenu servisa** (`horse-management`), koje Feign razreši preko Eureke.

Bonus koji odatle sledi: kad pokreneš **više instanci** horse-managementa, Eureka zna za sve, a Feign pozive automatski deli među njima — **Load Balancing**, bez ijedne linije koda.

Sve ostalo (baza, config-server, endpoint-i) ostaje isto kao na vežbama 3.


## Eureka-server (Naming Server)
---

_eureka-server — naming server koji automatski detektuje servise na mreži._

Na vežbama 3 su servisi bili „zalepljeni" jedan za drugi: analytics je u kodu imao zakucano `http://localhost:8080`. Ako se horse-management pomeri na drugi port — pukne. Ako ga pokreneš u dve instance — analytics zna samo za jednu.

**Naming server (Eureka)** to rešava: on je **telefonski imenik servisa**. Svaki servis se pri pokretanju **prijavi** Eureki pod svojim imenom (`horse-management`), a onaj ko ga poziva pita Eureku „gde je taj servis?" i dobije adresu (ili više adresa, ako ima više instanci). Nema više zakucanih portova u kodu.

Ovaj tutorijal pokriva **samo eureka-server** (nov, četvrti projekat). Izmene u horse-managementu i analytics-u su u sledeća dva fajla.

---
### Korak 1 — Kreiranje projekta

Nov projekat na https://start.spring.io/ (Maven, Java **21**, Spring Boot **4.0.1**):

| polje        | vrednost                                                         |
| ------------ | ---------------------------------------------------------------- |
| Group        | com.rzk                                                          |
| Artifact     | eureka-server                                                    |
| Dependencies | **Eureka Server** (Spring Cloud Discovery), Spring Boot DevTools |

#### Napomene:

- Pazi da ne uzmeš **Eureka Discovery Client** — to je druga zavisnost (za servise koji se _registruju_, ne za sam imenik). Server dobija **Eureka Server**, klijenti dobijaju **Eureka Discovery Client**.

---

### Korak 2 — Glavna klasa (`@EnableEurekaServer`)

`EurekaServerApplication.java`:

```java
package com.rzk.eurekaserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

Šta rade anotacije:

- `@SpringBootApplication` — standardno
- `@EnableEurekaServer` — uključuje Eureka mašineriju: registry (spisak servisa), REST API za registraciju i web dashboard. Bez nje je ovo prazan Boot projekat.


---

### Korak 3 — `application.properties`

```properties
spring.application.name=eureka-server
server.port=8761
eureka.client.register-with-eureka=false
```

#### Šta ovo znači:

- `spring.application.name=eureka-server` — ime aplikacije
- `server.port=8761` — **8761** je standardni port (po konvenciji) za Eureku. Klijenti ga podrazumevano traže baš tu.
- `eureka.client.register-with-eureka=false` — **Eureka ne registruje samu sebe** kao servis u sopstvenom imeniku

☝️🤓 Zašto `register-with-eureka=false`: zavisnost Eureka Server u sebi vuče i klijent deo, pa bi se server po defaultu **sam prijavio sebi** kao običan servis. 
U produkciji, gde ima više Eureka servera koji se međusobno repliciraju, ovo bi bilo `true` — ali za vežbe imamo jedan server.

---

### Korak 4 — Pokretanje i Eureka dashboard

Pokreni `EurekaServerApplication`. 

Otvori u browser-u:

```
http://localhost:8761
```

Dobijaš **Eureka dashboard** — web stranicu sa statusom servera. Bitan deo je tabela **„Instances currently registered with Eureka"**.

Sad je ona **prazna** (piše „No instances available") — normalno, jer se još nijedan servis nije prijavio. 
#### Napomene:

- Imena servisa u dashboard-u su **velikim slovima** — to je Eureka konvencija. U kodu (`@FeignClient(name = "horse-management")`) pišeš malim slovima, Eureka ne razlikuje velika/mala.
- Ako u logu vidiš warning-e o `EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP...` — to je normalno u razvoju (self-preservation mode), ne remeti rad.

---
## horse-management-service

Registracija na Eureku + `/debug` endpoint

---

_horse-management-service ostaje skoro isti kao na vežbama 3 — samo se sada registruje na Eureku, da bi ga analytics mogao pronaći po imenu._

Ovde ima **vrlo malo posla**: jedna zavisnost i jedan endpoint. Model, repozitorijum, servis i `application.properties` ostaju netaknuti.

> **Pre pokretanja:** eureka-server (8761) i config-server (8888) treba da rade, plus VPN za bazu.

---

### Korak 1 — `pom.xml`

U `pom.xml` dodaj zavisnost:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

Posle izmene: **Maven → Sync Project** (i po potrebi **Generate Sources and Update Folders**).

#### Napomene:

- Ovo je **Eureka Discovery Client** (ne Server!). Server je imenik, klijent se u imenik upisuje.
- `spring-cloud-dependencies` (verzija `2025.1.0`) već stoji u `dependencyManagement` iz vežbi 3, pa zavisnost ne treba verzija.
- Ostale zavisnosti (data-jpa, webmvc, config-client, mysql, lombok) ostaju kakve jesu.

---

### Korak 2 — `/debug` endpoint (za dokaz Load Balancing-a)

U `HorseController` dodaj injektovan `Environment` i novi endpoint:

```java
package com.rzk.horse_management_service.controller;

import com.rzk.horse_management_service.model.Horse;
import com.rzk.horse_management_service.service.HorseService;
import lombok.RequiredArgsConstructor;
import org.springframework.core.env.Environment;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
@RequiredArgsConstructor
@RequestMapping("/horses")
public class HorseController {

    private final HorseService horseService;
    private final Environment environment;

    @GetMapping
    public List<Horse> getAllHorses() {
        return horseService.getAllHorses();
    }

    @GetMapping("/{idHorse}")
    public Horse getHorseById(@PathVariable Integer idHorse) {
        return horseService.getHorseById(idHorse);
    }

    @GetMapping("/debug")
    public String debug() {
        return environment.getProperty("local.server.port");
    }
}
```

Šta je novo:

- `private final Environment environment;` — Spring-ov `Environment` (pristup svim propertyjima aplikacije); `@RequiredArgsConstructor` ga injektuje uz `HorseService`
- `@GetMapping("/debug")` → `GET /horses/debug` — vraća **port na kojem ova instanca radi**


☝️🤓 Zašto nam treba `/debug` endpoint: kasnije pokrećemo **dve instance** horse-managementa (npr. na 8080 i 8081). Kad analytics preko Feign-a više puta pozove `/debug`, odgovor će naizmenično biti `8080`, `8081`, `8080`... — i to je **vidljiv dokaz** da Load Balancer rotira pozive među instancama.

---

### Korak 4 — Pokretanje i provera registracije

Redosled: **eureka-server (8761)** → **config-server (8888)** → **horse-management (8080)** (+ VPN).

**1. Pokreni `horse-management-service`** 

**2. Proveri Eureka dashboard**

```
http://localhost:8761
```

U tabeli „Instances currently registered with Eureka" sada treba da se pojavi:

|Application|AMIs|Availability Zones|Status|
|---|---|---|---|
|HORSE-MANAGEMENT|n/a (1)|(1)|**UP** (1) — `localhost:horse-management:8080`|

Ako ga nema — sačekaj par sekundi pa osveži.

**3. Proveri `/debug`**

```
GET http://localhost:8080/horses/debug
```

Očekivano: **200** + tekst:

```
8080
```

**4. Stari endpoint-i i dalje rade**

```
GET http://localhost:8080/horses
GET http://localhost:8080/horses/1
```

Očekivano: **200** + konji, isto kao na vežbama 3.

#### Napomene:

- Ako se servis **ne pojavi** u dashboard-u: proveri da je eureka-server pokrenut **pre** njega i da je zavisnost `eureka-client` stvarno povučena (Maven → Sync Project).
- Ako u logu horse-managementa vidiš `Connection refused` ka `localhost:8761` — Eureka nije podignuta. Servis će svejedno raditi (samo neregistrovan) i pokušavaće ponovo.

---

### Šta se desilo pod haubom

1. Eureka klijent na classpath-u → Spring Boot ga automatski konfiguriše.
2. Na startu horse-management pošalje registraciju na `http://localhost:8761/eureka`: _„ja sam `horse-management`, na `localhost:8080`"_.
3. Eureka ga upiše u registry i prikaže u dashboard-u.
4. Dalje mu servis šalje **heartbeat** (~svakih 30s) — „živ sam". Ako prestane, Eureka ga izbaci.
5. Kad analytics zapita Eureku „gde je `horse-management`?", dobija listu instanci — i tu Feign preuzima priču.

## analytics-processing-service 
---
**Feign + Load Balancing**

Izbacujemo RestClient , ubacujemo Feign

---

_analytics-processing-service — umesto RestClient-a treba da koristi Feign za komunikaciju sa horse-management servisom._

 Endpoint-i analytics servisa ostaju isti, menja se način na koji zove horse-management: umesto ručnog RestClient poziva na zakucan `localhost:8080`, koristimo **Feign** koji servis pronalazi **po imenu** preko Eureke.

> **Pre pokretanja:** eureka-server (8761), config-server (8888), horse-management (8080), + VPN.

---

### Korak 1 — Dependencies (`pom.xml`)

Dodaj dve zavisnosti:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

Posle izmene: **Maven → Sync Project**.

#### Napomene:

- **OpenFeign** (Spring Cloud Routing) — deklarativni HTTP klijent.
- **Eureka Discovery Client** — da bi Feign uopšte mogao da pita „gde je `horse-management`?" (i da bi se sam analytics registrovao).

---

### Korak 2 — `application.properties`

```properties
spring.application.name=analytics-processing-service
server.port=8000
```

#### Napomene:

- Port je promenjen sa `8081` (vežbe 3) na **`8000`** — jer nam `8081` treba za **drugu instancu** horse-managementa (Korak 6).

---

### Korak 3 — `@EnableFeignClients` na glavnoj klasi

```java
package com.rzk.analytics_processing_service;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.openfeign.EnableFeignClients;

@SpringBootApplication
@EnableFeignClients
public class AnalyticsProcessingServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(AnalyticsProcessingServiceApplication.class, args);
    }
}
```


- Bez ove anotacije, `HorseProxy` bi ostao „samo interfejs" i injektovanje bi puklo (nema bean-a).

---

### Korak 4 — `HorseProxy` (Feign interfejs) ⭐

Ovo je centralna stvar vežbi 4. Nov paket `feign`, u njemu **samo interfejs** — bez ijedne linije implementacije.

`feign/HorseProxy.java`:

```java
package com.rzk.analytics_processing_service.feign;

import com.rzk.analytics_processing_service.dto.HorseDto;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

import java.util.List;

@FeignClient(name = "horse-management") //, url = "localhost:8080")
public interface HorseProxy {

    @GetMapping("/horses")
    List<HorseDto> getAllHorses();

    @GetMapping("/horses/{idHorse}")
    HorseDto getHorseById(@PathVariable Integer idHorse);

    @GetMapping("/horses/debug")
    String debug();
}
```

Šta rade anotacije:

- `@FeignClient(name = "horse-management")` — **ime servisa u Eureki**, ne URL. Feign će pitati Eureku gde se taj servis nalazi.
- `@GetMapping("/horses")` — ista anotacija kao u kontroleru, ali ovde znači suprotno: **„pošalji `GET /horses`"**, a ne „primi ga"
- `@PathVariable Integer idHorse` — vrednost se **ubacuje** u `{idHorse}` u putanji
- Povratni tipovi (`List<HorseDto>`, `HorseDto`, `String`) — Feign automatski deserijalizuje odgovor u njih

#### Napomene:

- Putanje (`/horses`, `/horses/{idHorse}`, `/horses/debug`) moraju **tačno** da odgovaraju endpoint-ima horse-management kontrolera.
- Klasa se zove **Proxy** jer je lokalni „proxy" udaljenog servisa. Zoveš `horseProxy.getAllHorses()` kao običnu Java metodu, a ispod se dešava HTTP poziv preko mreže.

☝️🤓 **Zakomentarisani `url = "localhost:8080"` je poenta cele priče.** 
Feign ume oba načina:

| pristup             | kod                                              | posledica                                                                  |
| ------------------- | ------------------------------------------------ | -------------------------------------------------------------------------- |
| **URL** (hardcoded) | `@FeignClient(name="...", url="localhost:8080")` | ide **uvek** na 8080; ne pita Eureku; **nema Load Balancing-a**            |
| **ime iz Eureke**   | `@FeignClient(name = "horse-management")`        | pita Eureku gde je servis; ako ima više instanci → **LoadBalancer rotira** |

Kad je `url` naveden, Feign ga koristi i **ignoriše** Eureku. Zato ga komentarišemo — hoćemo discovery + balansiranje. (Ali dobro je znati da `url` postoji: koristan je kad servis nije u Eureki, npr. neki eksterni API.)

---

### Korak 5 — Servis: RestClient → Feign

`service/AnalyticsService.java`:

```java
package com.rzk.analytics_processing_service.service;

import com.rzk.analytics_processing_service.dto.HorseDto;
import com.rzk.analytics_processing_service.feign.HorseProxy;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
@RequiredArgsConstructor
public class AnalyticsService {

//    private final RestClient restClient = RestClient.create("http://localhost:8080");
    private final HorseProxy horseProxy;

    public List<HorseDto> fetchAllHorses() {
//        return restClient.get()
//                .uri("/horses")
//                .retrieve()
//                .body(ParameterizedTypeReference.forType(List.class));
        return horseProxy.getAllHorses();
    }

    public HorseDto fetchHorseById(Integer idHorse) {
//        return restClient.get()
//                .uri("/horses/{id}", idHorse)
//                .retrieve()
//                .body(HorseDto.class);
        return horseProxy.getHorseById(idHorse);
    }

    public String debug() {
        return horseProxy.debug();
    }
}
```

Stari RestClient kod je namerno ostavljen **zakomentarisan** — da se vidi kontrast:

| RestClient (vežbe 3)                                         | Feign (vežbe 4)                           |
| ------------------------------------------------------------ | ----------------------------------------- |
| `RestClient.create("http://localhost:8080")`                 | `@FeignClient(name = "horse-management")` |
| ručno: `.get().uri(...).retrieve().body(...)`                | `horseProxy.getAllHorses()`               |
| hardcoded URL i port                                         | ime servisa, adresu razrešava Eureka      |
| jedna instanca                                               | Load Balancing preko više instanci        |
| `ParameterizedTypeReference.forType(List.class)` (raw lista) | `List<HorseDto>` (tip poznat iz potpisa)  |

Šta je novo:

- `@RequiredArgsConstructor` — sad je **potreban** (`HorseProxy` se injektuje kroz konstruktor); ranije je RestClient bio pravljen ručno u polju
- `private final HorseProxy horseProxy;` — Spring ubacuje **Feign-om generisanu implementaciju**
- `debug()` — nova metoda, samo prosledi poziv proxy-ju

☝️🤓 Uz Feign je i **tip** bolji: `getAllHorses()` vraća pravu `List<HorseDto>`, dok je RestClient verzija vraćala sirovu `List` (listu mapa). To je posledica toga što Feign čita generički tip iz potpisa metode.

---

### Korak 6 — `AnalyticsController` (+ `/debug`)

```java
package com.rzk.analytics_processing_service.controller;

import com.rzk.analytics_processing_service.dto.HorseDto;
import com.rzk.analytics_processing_service.service.AnalyticsService;
import lombok.AllArgsConstructor;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
@RequestMapping("/analytics/horses")
@AllArgsConstructor
public class AnalyticsController {

    private final AnalyticsService analyticsService;

    @GetMapping
    public List<HorseDto> getAllHorses() {
        return analyticsService.fetchAllHorses();
    }

    @GetMapping("/count")
    public String countHorses() {
        List<HorseDto> horses = analyticsService.fetchAllHorses();
        return "There are " + horses.size() + " horses in the database.";
    }

    @GetMapping("/name/{idHorse}")
    public String getHorseNameById(@PathVariable Integer idHorse) {
        HorseDto horseDto = analyticsService.fetchHorseById(idHorse);
        return "Horse with id: " + idHorse + " is named " + horseDto.getFullName() + " (" + horseDto.getNickname() + ").";
    }

    @GetMapping("/debug")
    public String debug() {
        return analyticsService.debug();
    }
}
```

Šta je novo u odnosu na vežbe 3:

- samo `@GetMapping("/debug")` → `GET /analytics/horses/debug` — vraća **port instance horse-managementa koja je odgovorila**

Ostalo (`/analytics/horses`, `/count`, `/name/{idHorse}`) je **nepromenjeno**. `HorseDto` takođe ostaje isti.

---

### Korak 7 — Prvo pokretanje (jedna instanca)

Redosled: eureka (8761) → config (8888) → horse-management (8080) → **analytics (8000)**.

**Eureka dashboard** (`http://localhost:8761`) sad treba da pokaže **dva** servisa:

|Application|Status|
|---|---|
|HORSE-MANAGEMENT|UP (1) — `localhost:horse-management:8080`|
|ANALYTICS-PROCESSING-SERVICE|UP (1) — `localhost:analytics-processing-service:8000`|

**Testiraj endpoint-e** (pazi: port je sad **8000**, ne 8081):

```
GET http://localhost:8000/analytics/horses
GET http://localhost:8000/analytics/horses/count
GET http://localhost:8000/analytics/horses/name/1
GET http://localhost:8000/analytics/horses/debug
```

Očekivano — isti odgovori kao na vežbama 3 (`There are 5 horses in the database.` itd.), plus:

```
GET http://localhost:8000/analytics/horses/debug
→ 8080
```

Sve radi **iako nigde nema `localhost:8080`** u analytics kodu — adresu je razrešila Eureka. To je poenta.

#### Napomene:

- Ako dobiješ `Load balancer does not contain an instance for the service horse-management` — horse-management se nije registrovao na Eureku (proveri dashboard, sačekaj par sekundi, restartuj).
- Ako se startovanje analytics-a ruši sa „No qualifying bean of type HorseProxy" — fali `@EnableFeignClients`.

---

### Korak 8 — Load Balancing (dve instance) ⭐

Sad dokazujemo da balansiranje radi. Ideja: pokrenuti **drugu instancu** horse-managementa na drugom portu i videti kako Feign rotira pozive.

**1. Napravi drugu Run konfiguraciju u IntelliJ-u**

- **Run → Edit Configurations…**
- Selektuj `HorseManagementServiceApplication` → **Copy Configuration** (ikonica sa dva lista) ili desni klik → _Copy_
- Novoj kopiji daj ime, npr. `HorseManagementServiceApplication (8081)`
- U polje **VM options** (ako ga ne vidiš: **Modify options → Add VM options**) upiši:
    
    ```
    -Dserver.port=8081
    ```
    
- **Apply → OK**

**2. Pokreni obe instance**

Prva (8080) već radi. Pokreni i onu sa `-Dserver.port=8081`. Sad imaš dva horse-managementa.

**3. Proveri Eureka dashboard**

```
http://localhost:8761
```

|Application|Status|
|---|---|
|HORSE-MANAGEMENT|**UP (2)** — `localhost:horse-management:8080`, `localhost:horse-management:8081`|

Ključno: **UP (2)** — dve instance pod **istim imenom**.

**4. Testiraj Load Balancing**

Pozovi **više puta zaredom**:

```
GET http://localhost:8000/analytics/horses/debug
```

Očekivani odgovori:

```
8081
8080
8081
8080
...
```

Port se **naizmenično menja** — svaki poziv ide na drugu instancu. To je **round-robin Load Balancing** i to je dokaz da radi.

#### Napomene:

- Ako uvek dobijaš isti port: sačekaj ~30s (Feign kešira listu instanci iz Eureke i osvežava je periodično), pa probaj ponovo.
- Rotacija ne mora da krene baš od 8080 — bitno je da se **menja**.
- Isto važi i za ostale endpoint-e (`/analytics/horses`, `/count`) — i oni se balansiraju; samo se to ne **vidi**, jer obe instance vraćaju iste konje iz iste baze. Zato nam `/debug` i treba.

☝️🤓 **Nigde nismo pisali kod za Load Balancing.** Nema `if`-a, nema brojača, nema liste portova. Sve što smo uradili: (1) servisi se registruju na Eureku, (2) Feign traži servis **po imenu**. Ostalo radi Spring Cloud LoadBalancer, koji je došao kao tranzitivna zavisnost. Ovo je i cela poenta naming servera — instance se dodaju/gase, a klijentski kod se ne dira.

☝️🤓 U produkciji bi dve instance bile na **dve mašine** (ne dva porta jedne mašine) i iza njih bi obično stajao još i API Gateway. Princip je isti.

---

### Šta se desilo pod haubom (poziv `/analytics/horses/debug`)

1. Klijent gađa `GET http://localhost:8000/analytics/horses/debug` (analytics).
2. `AnalyticsController.debug()` → `AnalyticsService.debug()` → `horseProxy.debug()`.
3. Feign proxy vidi `@FeignClient(name = "horse-management")` → pita **Spring Cloud LoadBalancer**: „daj mi instancu servisa `horse-management`".
4. LoadBalancer uzme listu instanci iz **Eureke** (`8080`, `8081`) i **round-robin**-om izabere jednu.
5. Feign pošalje `GET http://localhost:<izabrani-port>/horses/debug`.
6. Ta instanca horse-managementa vrati svoj `local.server.port` kao tekst.
7. Feign taj `String` vrati kao povratnu vrednost metode → analytics ga prosledi klijentu.

Sledeći poziv → korak 4 bira **drugu** instancu → drugi port u odgovoru.

---

### Rekapitulacija — vežbe 4 Primer

|komponenta|promena u odnosu na vežbe 3|
|---|---|
|**eureka-server**|**NOV** projekat, port `8761`, `@EnableEurekaServer`|
|**config-server**|bez izmena|
|**horse-management**|+ Eureka klijent, + `GET /horses/debug`|
|**analytics**|+ OpenFeign + Eureka klijent, `@EnableFeignClients`, **`HorseProxy`**, RestClient → Feign, port `8000`, + `/debug`|
|**Load Balancing**|besplatno (2 instance horse-managementa → round-robin)|

#### Struktura — analytics-processing-service

```
analytics-processing-service/
├── pom.xml                          (+ openfeign, + eureka-client)
└── src/main/
    ├── java/com/rzk/analytics_processing_service/
    │   ├── AnalyticsProcessingServiceApplication.java   (@EnableFeignClients)
    │   ├── feign/
    │   │   └── HorseProxy.java                          ← NOVO (@FeignClient)
    │   ├── dto/
    │   │   └── HorseDto.java                            (bez izmena)
    │   ├── service/
    │   │   └── AnalyticsService.java                    (RestClient → Feign)
    │   └── controller/
    │       └── AnalyticsController.java                 (+ /debug)
    └── resources/
        └── application.properties                       (port 8000)
```

#### Cela slika — sistem posle vežbi 4

```
              eureka-server  :8761   (imenik: ko je gde)
                ▲     ▲     ▲
     registruje │     │     │ pita "gde je horse-management?"
                │     │     │
config-server   │     │     │
     :8888      │     │     │
        │       │     │     │
        ▼       │     │     │
horse-management :8080 ──┤  │      ← instanca 1
horse-management :8081 ──┘  │      ← instanca 2 (-Dserver.port=8081)
        │                   │
      MySQL rzk             │
                            │
        analytics-processing-service  :8000
                 (Feign + LoadBalancer)
```

**Redosled pokretanja:** VPN → eureka (8761) → config (8888) → horse-management (8080) → horse-management (8081) → analytics (8000).



# Vežbe 4  Zadatak

Feign, Naming Server, Load Balancing (veterinarska klinika)

---

_Kreirati mikroservisnu arhitekturu koja sadrži dva mikroservisa i naming server._
## Arhitektura

```
              eureka-server  :8761
                 ▲         ▲
      registruje │         │ registruje
                 │         │
   animal-service  :8081   vet-service  :8082
   (Animal, Species)  ◄──────►  (VetRecord)
        │        Feign u oba smera        │
        │                                 │
        └────────► MySQL rzk ◄────────────┘
```

Dva servisa, **ista baza**, ali **različite tabele**:

|servis|port|tabele|Feign proxy koji ima|
|---|---|---|---|
|**eureka-server**|`8761`|—|—|
|**animal-service**|`8081`|`Animal`, `Species`|`VetProxy` → zove vet|
|**vet-service**|`8082`|`VetRecord`|`AnimalProxy` → zove animal|

☝️🤓 Ključna disciplina mikroservisa: **svaki servis dira samo svoje tabele.** vet-service _fizički_ može da dohvati tabelu `Animal` (ista baza, isti kredencijali), ali **ne sme** — to bi bilo kršenje granice servisa. Umesto toga, kad mu zatreba nešto o životinji, zove animal-service preko Feign-a. U pravom sistemu bi svaki servis imao **svoju bazu**, pa ni fizički ne bi mogao.

---

### Model baze

```
   ┌──────────────────┐            ┌────────────────────────┐
   │ Animal           │            │ VetRecord              │
   ├──────────────────┤            ├────────────────────────┤
   │ id       INT  PK │◄───────────│ id          INT  PK    │
   │ name     VC(100) │  (animal)  │ date        DATE       │
   │ species  INT  FK │            │ description VC(200)    │
   └────────┬─────────┘            │ animal      INT        │
            │                      └────────────────────────┘
            ▼
   ┌──────────────────┐
   │ Species          │
   ├──────────────────┤
   │ id    INT  PK    │
   │ name  VC(200)    │
   └──────────────────┘
```

|tabela|kolone|
|---|---|
|`Animal`|`id` INT, `name` VARCHAR(100), `species` INT (FK → `Species.id`)|
|`Species`|`id` INT, `name` VARCHAR(200)|
|`VetRecord`|`id` INT, `date` DATE, `description` VARCHAR(200), `animal` INT|

⚠️ **Najvažnija napomena celog Zadatka** (iz postavke):

> Tabela `VetRecord` treba da sadrži polje `Integer idAnimal` umesto polja `Animal animal`, zato što vet-service nema direktan pristup tabeli `Animal`, već joj pristupa preko animal-service-a putem Feign-a.

U JPA to izgleda ovako:

```java
@Column(name = "animal", nullable = false)
private Integer idAnimal;
```

Dakle **kolona se zove `animal`**, a **polje `idAnimal`** — i to **nije** `@ManyToOne`, nego običan `Integer`. Na dijagramu baze veza postoji (strani ključ), ali u kodu vet servisa je nema — jer vet ne poznaje entitet `Animal`.

☝️🤓 Uporedi sa `Animal` entitetom, gde `species` **jeste** `@ManyToOne Species` — jer su obe tabele „vlasništvo" animal-service-a, pa sme da ih poveže. Granica servisa određuje da li vezu modeluješ kao objekat ili kao goli id.

---
### Endpoints (šta pravimo)

|#|servis|endpoint|šta radi|
|---|---|---|---|
|1|vet|`GET /api/vet-records/{idAnimal}`|vraća listu `VetRecord`-a za datu životinju|
|2|animal|`POST /api/animals?id-species=X`|prima `Animal` u telu, nađe `Species`, doda je životinji, sačuva|
|3|animal|`GET /api/animals/animal-records/{idAnimal}`|**Feign** → zove #1 iz vet servisa|
|4|vet|`POST /api/vet-records?animal-name=X&id-species=Y`|**Feign** → zove #2 iz animal servisa, dobije sačuvanu životinju, njen id upiše u `VetRecord`, sačuva|
|5|animal|`GET /api/animals/animal-info-with-records/{idAnimal}`|nađe životinju + **Feign** zove #1 → sklopi `AnimalRecord` DTO|

Primeti kako se prepliću: **#3 i #5** su animal → vet; **#4** je vet → animal. Otud dva proxy interfejsa.

---

## Kreiranje projekata

Tri projekta na https://start.spring.io/ (Maven, Java **21**, Spring Boot **4.0.1**, Group `com.rzk`):

**1. eureka-server**

| polje        | vrednost                            |
| ------------ | ----------------------------------- |
| Group        | com.rzk                             |
| Artifact     | eureka-server                       |
| Dependencies | Eureka Server, Spring Boot DevTools |

_Identičan onom iz Primera_ (`@EnableEurekaServer`, port 8761, `register-with-eureka=false`). 

**2. animal-service**

| polje        | vrednost                                                                                                    |
| ------------ | ----------------------------------------------------------------------------------------------------------- |
| Group        | com.rzk                                                                                                     |
| Artifact     | animal-service                                                                                              |
| Dependencies | Spring Web, Spring Data JPA, MySQL Driver, OpenFeign, Eureka Discovery Client, Lombok, Spring Boot DevTools |

**3. vet-service**

| polje        | vrednost                                                                                                    |
| ------------ | ----------------------------------------------------------------------------------------------------------- |
| Group        | com.rzk                                                                                                     |
| Artifact     | vet-service                                                                                                 |
| Dependencies | Spring Web, Spring Data JPA, MySQL Driver, OpenFeign, Eureka Discovery Client, Lombok, Spring Boot DevTools |

#### Napomene:

- animal i vet imaju **identične zavisnosti** — oba pričaju sa bazom i oba koriste Feign.
- **Nema Config Client-a** — Zadatak ne koristi config-server.
- U oba `pom.xml` mora da postoji `<spring-cloud.version>2025.1.0</spring-cloud.version>` u `<properties>` **i** `<dependencyManagement>` blok sa `spring-cloud-dependencies`. Bez toga Maven ne zna verziju cloud zavisnosti (`openfeign` bi se prijavio kao „not found").

---

### `application.properties`

**eureka-server:**

```properties
spring.application.name=eureka-server
server.port=8761
eureka.client.register-with-eureka=false
```


**animal-service:**

```properties
spring.application.name=animal-service
server.port=8081

spring.datasource.url=jdbc:mysql://nastava.is.pmf.uns.ac.rs:3306/rzk
spring.datasource.username=rzk
spring.datasource.password=rzkStudent2019!
spring.jpa.hibernate.naming.physical-strategy=org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl

spring.datasource.hikari.maximum-pool-size=2
```

**vet-service:**

```properties
spring.application.name=vet-service
server.port=8082

spring.datasource.url=jdbc:mysql://nastava.is.pmf.uns.ac.rs:3306/rzk
spring.datasource.username=rzk
spring.datasource.password=rzkStudent2019!
spring.jpa.hibernate.naming.physical-strategy=org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl

spring.datasource.hikari.maximum-pool-size=2
```

Razlika je samo u **prve dve linije** (ime + port). Datasource je identičan.

#### Napomene:

- `spring.application.name` je ime pod kojim se servis registruje na Eureku — i **tačno to ime** ide u `@FeignClient("vet-service")` / `@FeignClient("animal-service")`. Ne sme da se razlikuje.
- **Nema nijedne `eureka.*` linije** — Eureka klijent podrazumevano gađa `http://localhost:8761/eureka`, gde nam server i jeste.
- `physical-strategy=...StandardImpl` — Hibernate ne dira nazive kolona (`idAnimal` polje → kolona `animal` je ionako eksplicitno mapirana, ali `name`/`date`/`description` moraju da ostanu kakve jesu).
- Za bazu treba **PMF VPN** (ako nisi na fakultetskoj mreži).

---
### Redosled pokretanja

1. **PMF VPN**
2. **eureka-server** (8761)
3. **vet-service** (8082)
4. **animal-service** (8081)

---

### Struktura (šta pravimo u sledeća dva koraka)

```
vet-service/                          animal-service/
├── model/VetRecord.java              ├── model/Animal.java
├── repository/VetRecordRepository    ├── model/Species.java
├── service/VetService.java           ├── repository/AnimalRepository.java
├── controller/VetController.java     ├── repository/SpeciesRepository.java
├── dto/AnimalDto.java                ├── service/AnimalService.java
└── feign/AnimalProxy.java  ──────┐   ├── controller/AnimalController.java
                                  │   ├── dto/VetRecordDto.java
                                  │   ├── dto/AnimalRecord.java
                                  └───┤ feign/VetProxy.java
                                      └──────────────────────
```

### Šta dalje

1. **vet-service** — `VetRecord` (sa `Integer idAnimal`), repo sa `findAllByIdAnimal`, servis, kontroler, `AnimalDto`, `AnimalProxy`
2. **animal-service** — `Animal`/`Species`, repo-i, servis, kontroler, `VetRecordDto`, `AnimalRecord`, `VetProxy` + **testiranje celog sistema**

## vet-service

### Tabela `VetRecord` + Feign ka animal servisu

---

_vet-service radi na portu 8082 i koristi tabelu `VetRecord` iz baze podataka._

vet-service ima dva endpoint-a:

- `GET /api/vet-records/{idAnimal}` — vraća listu `VetRecord`-a za datu životinju (čist rad sa bazom)
- `POST /api/vet-records` — prima `VetRecord` u telu + parametre `animal-name` i `id-species`; **preko Feign-a** zove animal servis da sačuva novu životinju, uzme njen id, upiše ga u `VetRecord` i sačuva

> **Pre pokretanja:** eureka-server (8761) + VPN.

---
### Korak 1 — Model `VetRecord`

`model/VetRecord.java`:

```java
package com.rzk.vet_service.model;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;

import java.time.LocalDate;

@Getter
@Setter
@Entity
public class VetRecord {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id", nullable = false)
    private Integer id;

    @Column(name = "date", nullable = false)
    private LocalDate date;

    @Column(name = "description", length = 200)
    private String description;

    @Column(name = "animal", nullable = false)
    private Integer idAnimal;

}
```

Šta rade anotacije:

- `@Entity`, `@Id`, `@GeneratedValue(IDENTITY)` — standardno, kao i dosad
- `@Column(name = "animal", nullable = false) private Integer idAnimal;` — **najvažnija linija zadatka**

#### Napomene:

- **Kolona se zove `animal`, polje `idAnimal`** — imena se namerno razlikuju, pa `@Column(name = "animal")` **mora** da bude tu (bez toga bi Hibernate tražio kolonu `idAnimal` koja ne postoji).
- `description` je jedino polje bez `nullable=false` → sme da bude `null`.

☝️🤓 **Zašto `Integer idAnimal` a ne `@ManyToOne Animal animal`?** Zato što vet-service **nema entitet `Animal`** — ta tabela pripada animal-service-u. Da smo stavili `@ManyToOne`, morali bismo u vet servis da prekopiramo `Animal` klasu i dozvolimo mu da čita tuđu tabelu — a to ruši granicu između servisa. Umesto toga vet čuva **samo strani ključ kao broj**, a kad mu zatrebaju podaci o životinji, **pita animal servis preko Feign-a**.

Ovo je opšti obrazac u mikroservisima: _preko granice servisa se prenose id-jevi, ne objekti._

---

### Korak 2 — `VetRecordRepository`

`repository/VetRecordRepository.java`:

```java
package com.rzk.vet_service.repository;

import com.rzk.vet_service.model.VetRecord;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;

public interface VetRecordRepository extends JpaRepository<VetRecord, Integer> {
    List<VetRecord> findAllByIdAnimal(Integer idAnimal);
}
```

#### Napomene:

- `findAllByIdAnimal(Integer idAnimal)` — **derived query**: Spring Data iz imena metode sam generiše upit.
- Ime metode se gradi od **imena polja u entitetu** (`idAnimal`), **ne** od imena kolone (`animal`). Da smo napisali `findAllByAnimal(...)`, Spring bi tražio polje `animal` — ne bi ga našao i pukao bi na startu.
- `save()` (za `POST`) dolazi besplatno iz `JpaRepository`.

---

### Korak 3 — `AnimalDto`

vet servis ne poznaje entitet `Animal`, pa mu treba **sopstvena, mala klasa** kojom priča sa animal servisom.

`dto/AnimalDto.java`:

```java
package com.rzk.vet_service.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;

@Getter
@Builder
@AllArgsConstructor
public class AnimalDto {
    private Integer id;
    private String name;
}
```

Šta rade anotacije:

- `@Getter` — geteri (treba nam `getId()`)
- `@Builder` — omogućava `AnimalDto.builder().name("Rex").build()`
- `@AllArgsConstructor` — konstruktor sa svim poljima (potreban Builder-u)

#### Napomene:

- DTO ima **samo `id` i `name`** — vet servisu ništa više o životinji ne treba.
- Koristi se u **oba smera**: kao **telo zahteva** ka animal servisu (`{name: "Rex"}`, bez id-ja) i kao **odgovor** koji stigne nazad (`{id: 7, name: "Rex"}`, sa id-jem).

☝️🤓 Primeti da DTO **nema `species`**. Vrsta se animal servisu šalje kao **request parametar** (`id-species`), ne kroz telo — a animal servis je sam nađe u bazi i zakači za životinju. vet servis nikad ne vidi objekat `Species`.

---

### Korak 4 — `AnimalProxy` (Feign ka animal servisu)

`feign/AnimalProxy.java`:

```java
package com.rzk.vet_service.feign;

import com.rzk.vet_service.dto.AnimalDto;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestParam;

@FeignClient("animal-service")
public interface AnimalProxy {

    @PostMapping("/api/animals")
    AnimalDto addAnimal(@RequestParam("id-species") Integer idSpecies, @RequestBody AnimalDto animal);

}
```

Šta rade anotacije:

- `@FeignClient("animal-service")` — ime servisa u Eureki (skraćeni zapis za `@FeignClient(name = "animal-service")`)
- `@PostMapping("/api/animals")` — pošalji **POST** na tu putanju animal servisa
- `@RequestParam("id-species") Integer idSpecies` — dodaj query parametar → `?id-species=3`
- `@RequestBody AnimalDto animal` — pošalji DTO kao JSON telo zahteva
- povratni tip `AnimalDto` — odgovor (sačuvana životinja **sa id-jem**) se deserijalizuje nazad u DTO

#### Napomene:

- Anotacije su iste kao u kontroleru, ali **značenje je obrnuto**: u kontroleru „primam ovo", u Feign interfejsu „šaljem ovo".
- Putanja i parametar (`/api/animals`, `id-species`) moraju **tačno** da odgovaraju endpoint-u animal kontrolera.
- Nema implementacije — generiše je `@EnableFeignClients` sa glavne klase.

---

### Korak 5 — Glavna klasa

```java
package com.rzk.vet_service;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.openfeign.EnableFeignClients;

@SpringBootApplication
@EnableFeignClients
public class VetServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(VetServiceApplication.class, args);
    }

}
```

- `@EnableFeignClients` — bez nje `AnimalProxy` ne postaje bean i injektovanje u `VetService` puca.

---

### Korak 6 — `VetService`

`service/VetService.java`:

```java
package com.rzk.vet_service.service;

import com.rzk.vet_service.dto.AnimalDto;
import com.rzk.vet_service.feign.AnimalProxy;
import com.rzk.vet_service.model.VetRecord;
import com.rzk.vet_service.repository.VetRecordRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
@RequiredArgsConstructor
public class VetService {

    private final VetRecordRepository vetRecordRepository;
    private final AnimalProxy animalProxy;

    public List<VetRecord> getVetRecordsForAnimal(Integer idAnimal) {
        return vetRecordRepository.findAllByIdAnimal(idAnimal);
    }

    public VetRecord addVetRecordNewAnimal(String animalName, Integer idSpecies, VetRecord vetRecord) {
        AnimalDto animalDto = animalProxy.addAnimal(idSpecies, AnimalDto.builder().name(animalName).build());
        vetRecord.setIdAnimal(animalDto.getId());
        return vetRecordRepository.save(vetRecord);
    }
}
```

Šta se dešava:

- `getVetRecordsForAnimal` — čist rad sa bazom, prosledi repo metodu
- `addVetRecordNewAnimal` — tri koraka:
    1. `animalProxy.addAnimal(idSpecies, AnimalDto.builder().name(animalName).build())` → **Feign poziv** ka animal servisu: „napravi mi životinju imena X, vrste Y". Šalje DTO **bez id-ja** (id će dodeliti baza).
    2. `vetRecord.setIdAnimal(animalDto.getId())` → iz odgovora izvuče **dodeljeni id** i upiše ga u karton
    3. `vetRecordRepository.save(vetRecord)` → sačuva karton u bazu


☝️🤓 `AnimalDto.builder().name(animalName).build()` — `@Builder` iz Lombok-a. Polje `id` ostaje `null`, što je i ispravno: id dodeljuje baza pri upisu.

---

### Korak 7 — `VetController`

`controller/VetController.java`:

```java
package com.rzk.vet_service.controller;

import com.rzk.vet_service.model.VetRecord;
import com.rzk.vet_service.service.VetService;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequiredArgsConstructor
@RequestMapping("/api/vet-records")
public class VetController {

    private final VetService vetService;

    @GetMapping("/{idAnimal}")
    public List<VetRecord> getVetRecordsForAnimal(@PathVariable Integer idAnimal) {
        return vetService.getVetRecordsForAnimal(idAnimal);
    }

    @PostMapping
    public VetRecord addVetRecordNewAnimal(@RequestParam("animal-name") String animalName, @RequestParam("id-species") Integer idSpecies, @RequestBody VetRecord vetRecord) {
        return vetService.addVetRecordNewAnimal(animalName, idSpecies, vetRecord);
    }
}
```

Šta rade anotacije:

- `@RequestMapping("/api/vet-records")` — bazna putanja
- `@GetMapping("/{idAnimal}")` → `GET /api/vet-records/{idAnimal}`
- `@PostMapping` (bez putanje) → `POST /api/vet-records`
- `@RequestParam("animal-name")` / `@RequestParam("id-species")` — query parametri; imena su **sa crticama**, pa se eksplicitno navode (Java polja se zovu `animalName`, `idSpecies`)
- `@RequestBody VetRecord vetRecord` — telo zahteva se mapira u entitet

#### Napomene:

- `POST` vraća direktno `VetRecord` → status **200** (nema `ResponseEntity`, nema 201).
- Kontroler nema validaciju ni obradu grešaka — ako animal servis ne nađe vrstu, doći će do greške niže u lancu.

---

### Korak 8 — Testiranje

> vet-service se za `POST` **oslanja na animal-service**, pa taj endpoint možeš testirati tek kad i animal servis proradi (sledeći tutorijal). `GET` možeš odmah.

Redosled: eureka (8761) → **vet-service (8082)**.

**1. Provera registracije** — `http://localhost:8761`:

|Application|Status|
|---|---|
|VET-SERVICE|UP (1) — `localhost:vet-service:8082`|

**2. `GET /api/vet-records/{idAnimal}`**

```
GET http://localhost:8082/api/vet-records/1
```

Očekivano: **200** + lista kartona za životinju sa id 1:

```json
[
    {
        "id": 1,
        "date": "2025-03-14",
        "description": "Redovna vakcinacija",
        "idAnimal": 1
    }
]
```

Primeti: u JSON-u polje je **`idAnimal`** (ime Java polja), iako se kolona u bazi zove `animal`. Jackson serijalizuje po **poljima klase**, ne po kolonama.

Ako životinja nema kartona → prazan niz `[]` (status i dalje 200).

**3. `POST /api/vet-records`** — _(radi tek kad animal servis bude gore)_

```
POST http://localhost:8082/api/vet-records?animal-name=Rex&id-species=1
```

Telo:

```json
{
    "date": "2026-07-12",
    "description": "Prvi pregled"
}
```

Očekivano: **200** + sačuvani karton, **sa popunjenim `idAnimal`**:

```json
{
    "id": 12,
    "date": "2026-07-12",
    "description": "Prvi pregled",
    "idAnimal": 8
}
```

gde je `8` id životinje koju je **animal servis** upravo napravio.

#### Napomene:

- U telu **ne šalješ `idAnimal`** — njega popunjava servis iz Feign odgovora. Ako ga i pošalješ, biće pregažen.
- `date` je obavezan (`nullable = false`) — bez njega puca upis.

---

## animal-service

### Tabele `Animal` i `Species` + Feign ka vet servisu
---

_animal-service radi na portu 8081 i koristi tabele `Animal` i `Species` iz baze podataka._

animal-service ima tri endpoint-a:

- `POST /api/animals?id-species=X` — prima `Animal` u telu, nađe vrstu u bazi, doda je životinji, sačuva
- `GET /api/animals/animal-records/{idAnimal}` — **preko Feign-a** zove vet servis i vrati listu kartona
- `GET /api/animals/animal-info-with-records/{idAnimal}` — nađe životinju u bazi **+ Feign** za kartone → sklopi `AnimalRecord`

> **Pre pokretanja:** eureka-server (8761), vet-service (8082), + VPN.

---

### Korak 1 — Modeli `Animal` i `Species`

`model/Animal.java`:

```java
package com.rzk.animal_service.model;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
@Entity
public class Animal {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id", nullable = false)
    private Integer id;

    @Column(name = "name", nullable = false, length = 100)
    private String name;

    @ManyToOne(optional = false)
    @JoinColumn(name = "species", nullable = false)
    private Species species;

}
```

`model/Species.java`:

```java
package com.rzk.animal_service.model;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
@Entity
public class Species {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id", nullable = false)
    private Integer id;

    @Column(name = "name", nullable = false, length = 200)
    private String name;

}
```

Šta rade anotacije:

- `@ManyToOne(optional = false)` — više životinja pripada jednoj vrsti; `optional=false` → životinja **mora** imati vrstu
- `@JoinColumn(name = "species", nullable = false)` — kolona `species` u tabeli `Animal` je strani ključ ka `Species.id`

☝️🤓 **Uporedi sa `VetRecord` iz prethodnog tutorijala.** Tamo je veza ka životinji bila **`Integer idAnimal`**, ovde je veza ka vrsti pravi **`@ManyToOne Species`** objekat. Razlika nije hir — `Animal` i `Species` su **obe tabele animal-servisa** (unutar iste granice), pa ih sme povezati JPA relacijom. `VetRecord` i `Animal` su u **različitim servisima**, pa tamo veza mora ostati goli id. Ovo je pravilo: _unutar servisa → objekti; preko granice → id-jevi._

---

### Korak 3 — Repozitorijumi

`repository/AnimalRepository.java`:

```java
package com.rzk.animal_service.repository;

import com.rzk.animal_service.model.Animal;
import org.springframework.data.jpa.repository.JpaRepository;

public interface AnimalRepository extends JpaRepository<Animal, Integer> {
}
```

`repository/SpeciesRepository.java`:

```java
package com.rzk.animal_service.repository;

import com.rzk.animal_service.model.Species;
import org.springframework.data.jpa.repository.JpaRepository;

public interface SpeciesRepository extends JpaRepository<Species, Integer> {
}
```

Oba su prazna — trebaju nam samo `findById()` i `save()`, koji dolaze iz `JpaRepository`.

---

### Korak 4 — DTO klase

`dto/VetRecordDto.java` — kako animal servis vidi karton koji stigne od vet servisa:

```java
package com.rzk.animal_service.dto;

import lombok.AllArgsConstructor;
import lombok.Getter;

import java.time.LocalDate;

@Getter
@AllArgsConstructor
public class VetRecordDto {
    private Integer id;
    private LocalDate date;
    private String description;
}
```

`dto/AnimalRecord.java` — spojeni odgovor (životinja + njeni kartoni):

```java
package com.rzk.animal_service.dto;

import com.rzk.animal_service.model.Animal;
import lombok.Builder;
import lombok.Getter;

import java.util.List;

@Builder
@Getter
public class AnimalRecord {
    private Animal animal;
    private List<VetRecordDto> vetRecords;
}
```

#### Napomene:

- `VetRecordDto` **nema `idAnimal`** — animal servis ga već zna (poslao ga je u zahtevu), pa mu ne treba nazad. Kad Feign deserijalizuje odgovor vet servisa (`{id, date, description, idAnimal}`), višak polja se jednostavno **ignoriše**.
- `AnimalRecord` je iz postavke: _„DTO klasa koja ima polje tipa `Animal` i listu `VetRecordDto`-a"_. Primeti da drži **pravi entitet `Animal`** (jer je iz sopstvene baze) i **DTO-e** za kartone (jer dolaze spolja).
- `@Builder` — koristi se u servisu (`AnimalRecord.builder().animal(...).vetRecords(...).build()`).

☝️🤓 Zašto su ovo dve **različite** `VetRecordDto` i `VetRecord` klase, u dva projekta? Jer servisi **ne dele kod**. vet servis ima svoj entitet `VetRecord` (mapiran na bazu), a animal servis ima svoj `VetRecordDto` (samo za čitanje JSON-a). Nemaju zajednički modul — komuniciraju **JSON-om**, ne Java tipovima. Zato oba mogu da evoluiraju nezavisno: dok se imena polja u JSON-u poklapaju, sve radi.

---

### Korak 5 — `VetProxy` (Feign ka vet servisu) ⭐

`feign/VetProxy.java`:

```java
package com.rzk.animal_service.feign;

import com.rzk.animal_service.dto.VetRecordDto;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

import java.util.List;

@FeignClient("vet-service")
public interface VetProxy {

    @GetMapping("/api/vet-records/{idAnimal}")
    List<VetRecordDto> getVetRecordsForAnimal(@PathVariable Integer idAnimal);

}
```

Šta rade anotacije:

- `@FeignClient("vet-service")` — ime vet servisa u Eureki
- `@GetMapping("/api/vet-records/{idAnimal}")` — pošalji `GET` na tu putanju
- `@PathVariable Integer idAnimal` — vrednost se ubacuje u `{idAnimal}`
- povratni tip `List<VetRecordDto>` — Feign deserijalizuje JSON niz u listu DTO-a

☝️🤓 **Sad imamo Feign u oba smera:**

|smer|proxy|gde živi|poziva|
|---|---|---|---|
|animal → vet|`VetProxy`|animal-service|`GET /api/vet-records/{idAnimal}`|
|vet → animal|`AnimalProxy`|vet-service|`POST /api/animals?id-species=...`|

Servisi su **međusobno zavisni**, ali nijedan ne zna gde je onaj drugi — obojica pitaju Eureku. Da smo koristili zakucane URL-ove (`url = "localhost:8082"`), morali bismo da ih menjamo pri svakoj promeni porta.

---

### Korak 6 — Glavna klasa

```java
package com.rzk.animal_service;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.openfeign.EnableFeignClients;

@SpringBootApplication
@EnableFeignClients
public class AnimalServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(AnimalServiceApplication.class, args);
    }

}
```

`@EnableFeignClients` — isto kao u vet servisu; bez nje `VetProxy` ne postaje bean.

---

### Korak 7 — `AnimalService`

`service/AnimalService.java`:

```java
package com.rzk.animal_service.service;

import com.rzk.animal_service.dto.AnimalRecord;
import com.rzk.animal_service.dto.VetRecordDto;
import com.rzk.animal_service.feign.VetProxy;
import com.rzk.animal_service.model.Animal;
import com.rzk.animal_service.model.Species;
import com.rzk.animal_service.repository.AnimalRepository;
import com.rzk.animal_service.repository.SpeciesRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Optional;

@Service
@RequiredArgsConstructor
public class AnimalService {

    private final AnimalRepository animalRepository;
    private final SpeciesRepository speciesRepository;
    private final VetProxy vetProxy;

    public Animal addAnimal(Integer idSpecies, Animal animal) {
        Optional<Species> speciesOptional = speciesRepository.findById(idSpecies);
        if (speciesOptional.isEmpty()) {
            return null;
        }
        animal.setSpecies(speciesOptional.get());
        return animalRepository.save(animal);
    }

    public List<VetRecordDto> getAnimalRecords(Integer idAnimal) {
        return vetProxy.getVetRecordsForAnimal(idAnimal);
    }

    public AnimalRecord getAnimalInfoWithRecords(Integer idAnimal) {
        Optional<Animal> animalOptional = animalRepository.findById(idAnimal);
        if (animalOptional.isEmpty()) {
            return null;
        }

        List<VetRecordDto> vetRecordDtos = vetProxy.getVetRecordsForAnimal(idAnimal);
        return AnimalRecord.builder()
                .animal(animalOptional.get())
                .vetRecords(vetRecordDtos)
                .build();
    }
}
```

Tri metode, tri različita obrasca:

**`addAnimal`** — čist rad sa bazom:

1. nađe `Species` po id-ju → ako je nema, vrati `null` (kontroler to pretvara u **404**)
2. `animal.setSpecies(...)` — zakači vrstu na životinju iz tela zahteva
3. `save()` — upiše u bazu

**`getAnimalRecords`** — čist Feign, baza se uopšte ne dira: samo prosledi poziv vet servisu.

**`getAnimalInfoWithRecords`** — **kombinacija**: prvo baza (nađi životinju), pa Feign (dohvati kartone), pa sklopi `AnimalRecord`.

#### Napomene:

- `return null` kad entiteta nema — signal kontroleru. Nema izuzetaka kao u vežbama 2 (ovde je namerno jednostavnije).
- `speciesRepository.findById(...).isEmpty()` — provera **pre** `.get()`, za razliku od horse-managementa gde je `.get()` bio bez provere.

☝️🤓 U `getAnimalInfoWithRecords` provera životinje ide **pre** Feign poziva. To je namerno: ako životinja ne postoji, nema smisla gnjaviti vet servis preko mreže. Mrežni poziv je **red veličine skuplji** od upita u lokalnu bazu, pa se ovakve provere uvek rade prvo lokalno.

---

### Korak 8 — `AnimalController`

`controller/AnimalController.java`:

```java
package com.rzk.animal_service.controller;

import com.rzk.animal_service.dto.AnimalRecord;
import com.rzk.animal_service.dto.VetRecordDto;
import com.rzk.animal_service.model.Animal;
import com.rzk.animal_service.service.AnimalService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequiredArgsConstructor
@RequestMapping("/api/animals")
public class AnimalController {

    private final AnimalService animalService;

    @PostMapping
    public ResponseEntity<Animal> addAnimal(@RequestParam("id-species") Integer idSpecies, @RequestBody Animal animal) {
        Animal savedAnimal = animalService.addAnimal(idSpecies, animal);
        if (savedAnimal == null) {
            return ResponseEntity.status(HttpStatus.NOT_FOUND).build();
        }
        return new ResponseEntity<>(savedAnimal, HttpStatus.CREATED);
    }

    @GetMapping("/animal-records/{idAnimal}")
    public List<VetRecordDto> getAnimalRecords(@PathVariable Integer idAnimal) {
        return animalService.getAnimalRecords(idAnimal);
    }

    @GetMapping("/animal-info-with-records/{idAnimal}")
    public ResponseEntity<AnimalRecord> getAnimalInfoWithRecords(@PathVariable Integer idAnimal) {
        AnimalRecord animalRecord = animalService.getAnimalInfoWithRecords(idAnimal);
        if (animalRecord == null) {
            return ResponseEntity.status(HttpStatus.NOT_FOUND).build();
        }
        return ResponseEntity.ok(animalRecord);
    }
}
```

Šta rade anotacije:

- `@PostMapping` → `POST /api/animals`
- `@RequestParam("id-species") Integer idSpecies` — query parametar sa crticom
- `@RequestBody Animal animal` — telo → entitet
- `ResponseEntity<Animal>` — omogućava **kontrolu statusa**

Statusi:

|situacija|status|kako|
|---|---|---|
|životinja sačuvana|**201 CREATED**|`new ResponseEntity<>(savedAnimal, HttpStatus.CREATED)`|
|vrsta ne postoji|**404 NOT FOUND**|`ResponseEntity.status(HttpStatus.NOT_FOUND).build()`|
|`AnimalRecord` nađen|**200 OK**|`ResponseEntity.ok(animalRecord)`|
|životinja ne postoji|**404**|`.build()` (prazno telo)|

#### Napomene:

- `getAnimalRecords` **nema** `ResponseEntity` — vraća listu direktno (200). Ako životinja nema kartona, biće `[]`.
- Uporedi sa `VetController`-om, koji nema `ResponseEntity` nigde. Nedosledno, ali to je njeno rešenje — prenosimo verno.

---

### Korak 9 — Testiranje celog sistema

Redosled: **VPN** → **eureka (8761)** → **vet-service (8082)** → **animal-service (8081)**.

**1. Eureka dashboard** (`http://localhost:8761`) — oba servisa gore:

|Application|Status|
|---|---|
|ANIMAL-SERVICE|UP (1) — `localhost:animal-service:8081`|
|VET-SERVICE|UP (1) — `localhost:vet-service:8082`|

---

**2. `POST /api/animals` — dodaj životinju** _(bez Feign-a, čista baza)_

```
POST http://localhost:8081/api/animals?id-species=1
```

Telo:

```json
{
    "name": "Rex"
}
```

Očekivano: **201 CREATED**

```json
{
    "id": 8,
    "name": "Rex",
    "species": {
        "id": 1,
        "name": "Pas"
    }
}
```

Ako pošalješ nepostojeću vrstu (`?id-species=999`) → **404**, prazno telo.

---

**3. `GET /api/animals/animal-records/{idAnimal}` — kartoni preko Feign-a** _(animal → vet)_

```
GET http://localhost:8081/api/animals/animal-records/1
```

Očekivano: **200** + lista kartona:

```json
[
    {
        "id": 1,
        "date": "2025-03-14",
        "description": "Redovna vakcinacija"
    }
]
```

Primeti: **nema `idAnimal`** u odgovoru (za razliku od direktnog poziva vet servisu) — jer `VetRecordDto` to polje nema.

---

**4. `POST /api/vet-records` — vet zove animal** _(vet → animal, sad konačno radi!)_

```
POST http://localhost:8082/api/vet-records?animal-name=Bela&id-species=2
```

Telo:

```json
{
    "date": "2026-07-12",
    "description": "Prvi pregled"
}
```

Očekivano: **200** + karton sa **popunjenim `idAnimal`**:

```json
{
    "id": 12,
    "date": "2026-07-12",
    "description": "Prvi pregled",
    "idAnimal": 9
}
```

Životinja „Bela" je pritom **kreirana u animal servisu** (id `9`) — proveri sa `GET /api/animals/animal-info-with-records/9`.

---

**5. `GET /api/animals/animal-info-with-records/{idAnimal}` — baza + Feign** ⭐

```
GET http://localhost:8081/api/animals/animal-info-with-records/9
```

Očekivano: **200** + spojeni objekat:

```json
{
    "animal": {
        "id": 9,
        "name": "Bela",
        "species": {
            "id": 2,
            "name": "Mačka"
        }
    },
    "vetRecords": [
        {
            "id": 12,
            "date": "2026-07-12",
            "description": "Prvi pregled"
        }
    ]
}
```

Ovo je **kruna zadatka**: jedan poziv koji spaja podatke iz **dve baze/dva servisa** — `animal` iz lokalne tabele, `vetRecords` preko mreže od vet servisa.

Nepostojeća životinja (`/999`) → **404**.

#### Napomene ako nešto pukne:

- `Load balancer does not contain an instance for the service vet-service` → vet servis nije registrovan; proveri dashboard i sačekaj par sekundi.
- `No qualifying bean of type VetProxy` → fali `@EnableFeignClients` na glavnoj klasi.
- **500** na `POST /api/vet-records` → najverovatnije `date` nije poslat (obavezan je) ili vrsta ne postoji (animal servis vrati 404, Feign to pretvori u izuzetak).
- Greška oko baze → VPN.

---

### Šta se desilo pod haubom (`/animal-info-with-records/9`)

1. Klijent → `GET http://localhost:8081/api/animals/animal-info-with-records/9`.
2. `AnimalController` → `AnimalService.getAnimalInfoWithRecords(9)`.
3. `animalRepository.findById(9)` → `SELECT * FROM Animal WHERE id=9` (+ dohvat `Species`). Ako je prazno → `null` → 404.
4. `vetProxy.getVetRecordsForAnimal(9)` → Feign pita **Eureku**: „gde je `vet-service`?" → dobije `localhost:8082`.
5. Feign šalje `GET http://localhost:8082/api/vet-records/9`.
6. vet-service → `findAllByIdAnimal(9)` → `SELECT * FROM VetRecord WHERE animal=9` → vrati JSON niz.
7. Feign deserijalizuje u `List<VetRecordDto>` (polje `idAnimal` iz JSON-a se ignoriše — DTO ga nema).
8. `AnimalRecord.builder().animal(...).vetRecords(...).build()` → serijalizuje se u JSON i vraća klijentu.

---

### Rekapitulacija — Zadatak (vežbe 4)

|komponenta|port|ključne stvari|
|---|---|---|
|**eureka-server**|8761|`@EnableEurekaServer`, `register-with-eureka=false`|
|**vet-service**|8082|`VetRecord` (**`Integer idAnimal`**), `AnimalProxy` → animal|
|**animal-service**|8081|`Animal` + `@ManyToOne Species`, `VetProxy` → vet|

**Svih 5 endpoint-a:**

|#|poziv|tip|
|---|---|---|
|1|`GET :8082/api/vet-records/{idAnimal}`|samo baza|
|2|`POST :8081/api/animals?id-species=X`|samo baza|
|3|`GET :8081/api/animals/animal-records/{idAnimal}`|samo Feign (animal→vet)|
|4|`POST :8082/api/vet-records?animal-name=X&id-species=Y`|Feign (vet→animal) + baza|
|5|`GET :8081/api/animals/animal-info-with-records/{idAnimal}`|baza + Feign (animal→vet)|

**Naučeno:**

- **Eureka** — servisi se registruju po imenu, niko ne zna tuđi port
- **Feign** — poziv udaljenog servisa izgleda kao poziv obične Java metode
- **DTO preko granice** — svaki servis ima svoje klase; komunikacija je JSON-om, ne deljenim kodom
- **Id, ne objekat** — `VetRecord.idAnimal` umesto `@ManyToOne Animal`, jer je `Animal` tuđa tabela
- **Load Balancing** — dolazi besplatno (probaj: pokreni drugu instancu vet servisa sa `-Dserver.port=8083` i vidi `UP (2)` u dashboard-u)

### Struktura

```
animal-service/
├── pom.xml
└── src/main/
    ├── java/com/rzk/animal_service/
    │   ├── AnimalServiceApplication.java   (@EnableFeignClients)
    │   ├── model/
    │   │   ├── Animal.java                 (@ManyToOne Species)
    │   │   └── Species.java
    │   ├── repository/
    │   │   ├── AnimalRepository.java
    │   │   └── SpeciesRepository.java
    │   ├── dto/
    │   │   ├── VetRecordDto.java
    │   │   └── AnimalRecord.java           (Animal + List<VetRecordDto>)
    │   ├── feign/
    │   │   └── VetProxy.java               (@FeignClient("vet-service"))
    │   ├── service/AnimalService.java
    │   └── controller/AnimalController.java
    └── resources/
        └── application.properties          (port 8081)
```
