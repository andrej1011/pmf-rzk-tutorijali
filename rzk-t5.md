# API Gateway, Circuit Breaker, Load Balancing
---

Nadograđujemo sistem veterinarske klinike sa vežbi 4. Dodajemo nov projekat `api-gateway` (jedna ulazna tačka za sve pozive, sa globalnim filterom za `x-api-key`), a u `animal-service` uvodimo Resilience4J — RateLimiter, Retry i Circuit Breaker sa fallback metodama. `eureka-server` i `vet-service` ostaju netaknuti.

---
## Teorija — API Gateway

Do sada je klijent morao da zna **gde je koji servis**: životinje na `8081`, veterinarski kartoni na `8082`. Čim dodaš treći servis, ili pokreneš drugu instancu, klijent to mora da prati.

**API Gateway** je jedna ulazna tačka ispred svih mikroservisa. Klijent zove **samo gateway** (kod nas `localhost:8765`), a gateway odlučuje kom servisu prosleđuje zahtev. Usput može da radi i:

- **autentifikaciju / autorizaciju** (kod nas: provera `x-api-key`)
- **logovanje** svih zahteva na jednom mestu
- **prepisivanje putanja** (`/home` → `/api/animals`)
- **load balancing** (biranje instance servisa)

**Spring Cloud Gateway** je default rešenje u Spring svetu. Radi na **WebFlux**-u, zato u gateway projektu **nema** `spring-boot-starter-webmvc`, a filteri vraćaju `Mono<Void>` umesto `void`.

☝️🤓 **Zašto gateway-u treba Eureka?**
Gateway zna da zahtev `/api/animals/**` ide na *animal-service*, ali ne zna **na kom je portu**. Zato je i on **Eureka klijent**: pita naming server „gde je `animal-service`?“, dobije listu instanci (npr. `localhost:8081`, `localhost:8091`) i prosledi zahtev jednoj od njih. Otuda `lb://animal-service` u rutama — `lb` = *load balanced*, isto ono što Feign radi interno.

---

## Teorija — Circuit Breaker, Retry, RateLimiter

Mikroservisi zovu jedan drugog preko mreže. Mreža puca, servisi padaju, baze se guše. **Resilience4J** je biblioteka koja te štiti od toga — tri obrasca koja koristimo:

### Retry
Ako poziv pukne, **pokušaj ponovo** N puta sa pauzom između pokušaja. Ako svi pokušaji propadnu → ide **fallback**. Korisno za *prolazne* greške (servis se restartuje, mreža trepnula).

### RateLimiter
Ograničava **broj poziva u vremenskom prozoru**. Višak poziva se odbija izuzetkom `RequestNotPermitted` → ide **fallback**. Štiti servis od preopterećenja (i od zlonamernog spamovanja).

### Circuit Breaker („osigurač“)
Ako neki servis stalno puca, nema smisla ga i dalje zvati — samo trošiš vreme i niti. Circuit breaker prati **procenat grešaka** i, kad pređe prag, prestaje da zove servis i **odmah** vraća fallback (*fail-fast*).

**Tri stanja:**

| Stanje | Šta se dešava | Kada se prelazi dalje |
|---|---|---|
| **CLOSED** | Normalan rad. Pozivi prolaze do metode, uspesi i greške se broje. | Kad se sakupi `minimum-number-of-calls` **i** procenat grešaka pređe `failure-rate-threshold` (default 50%) → **OPEN** |
| **OPEN** | Pozivi **ne stižu do metode uopšte**. Baca se `CallNotPermittedException` → odmah fallback. | Posle `wait-duration-in-open-state` → **HALF_OPEN** |
| **HALF_OPEN** | Propušta se ograničen broj *probnih* poziva. | Probni pozivi prolaze → **CLOSED**; i dalje pucaju → **OPEN** |

☝️🤓 Ključna razlika u odnosu na Retry: **Retry i dalje zove servis** (samo više puta), dok **OPEN circuit breaker uopšte ne zove servis**. Zato u kodu postoji `circuitBreakerCounter` — kad je osigurač OPEN, log iz tela metode se **više ne ispisuje**, i to je najlepši dokaz da radi.

### Fallback metoda
Rezervni odgovor kad glavna metoda pukne (ili je odbijena). Pravila:

- **isti povratni tip** kao glavna metoda
- **isti parametri** + **`Exception ex`** (ili konkretniji tip izuzetka) **na kraju**
- mora biti u **istoj klasi** (i dovoljno vidljiva — `public`)

---

## Zadatak 1 — api-gateway

Nov projekat. 

Rutira `/api/animals/**` na animal-service, `/api/vet-records/**` na vet-service, prepisuje `/home` na `/api/animals`, i globalnim filterom zahteva `x-api-key: validKey123` — inače 403.

### Korak 1 — Kreiranje projekta i `pom.xml`

Spring Initializr → **Maven**, **Java 21**, **Spring Boot 4.0.1**, group `com.rzk`, artifact `api-gateway`.

Dependencies:
- **Reactive Gateway** (Spring Cloud Routing)
- **Eureka Discovery Client** (Spring Cloud Discovery)
- Lombok, Spring Boot DevTools

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.0.1</version>
    <relativePath/>
</parent>
<groupId>com.rzk</groupId>
<artifactId>api-gateway</artifactId>
<version>0.0.1-SNAPSHOT</version>

<properties>
    <java.version>21</java.version>
    <spring-cloud.version>2025.1.0</spring-cloud.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway-server-webflux</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <scope>runtime</scope>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>io.projectreactor</groupId>
        <artifactId>reactor-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <scope>provided</scope>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

#### Napomene:
- Artifact se u Boot 4 / Spring Cloud 2025.1.0 zove **`spring-cloud-starter-gateway-server-webflux`** — ne više `spring-cloud-starter-gateway`.
- `<spring-cloud.version>` **i** `<dependencyManagement>` blok moraju postojati, inače cloud zavisnosti pucaju sa „not found“ (prazna verzija).
- **Nema** `spring-boot-starter-webmvc` ovde — gateway je reaktivan i sam podiže Netty. Ako slučajno dodaš webmvc, aplikacija se buni oko toga koji web server da pokrene.
- Posle izmene `pom.xml` u IntelliJ-u: **Maven → Sync Project** (+ *Generate Sources and Update Folders*).

### Korak 2 — `application.properties`

```properties
spring.application.name=api-gateway
server.port=8765

spring.cloud.gateway.server.webflux.discovery.locator.enabled=true
spring.cloud.gateway.server.webflux.discovery.locator.lower-case-service-id=true

#logovi u konzoli
logging.level.org.springframework.cloud.gateway=TRACE
```

**Šta koja linija radi:**

| Property | Značenje |
|---|---|
| `spring.application.name=api-gateway` | Ime pod kojim se gateway registruje na Eureku |
| `server.port=8765` | Port gateway-a — **jedina adresa koju klijent treba da zna** |
| `discovery.locator.enabled=true` | Gateway automatski pravi rutu za **svaki** servis registrovan na Eureki: `/{ime-servisa}/**` → taj servis |
| `discovery.locator.lower-case-service-id=true` | Ta automatska ruta koristi **mala slova** (`/animal-service/**` umesto `/ANIMAL-SERVICE/**`) |
| `logging.level...gateway=TRACE` | Detaljan log rutiranja u konzoli (vidi se koja ruta je „upalila“) |

☝️🤓 **Discovery locator vs. naše ručne rute.** Discovery locator ti *besplatno* daje `GET localhost:8765/animal-service/api/animals`. Ali zadatak traži lepše putanje (`/api/animals/**`) i prepisivanje `/home`, pa ipak pišemo eksplicitne rute u `ApiGatewayConfig`. **Obe stvari rade istovremeno** — automatska ruta i dalje postoji, korisna je za brzu proveru da je Eureka veza živa.

☝️🤓 Nema `eureka.client.service-url.defaultZone` — default je `http://localhost:8761/eureka`, a Eureka je baš tamo. Zato radi „samo od sebe“.


### Korak 3 — `LoggingFilter` (globalni filter #1)

Novi paket `filter`:

```java
package com.rzk.api_gateway.filter;

import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

@Component
@Slf4j
public class LoggingFilter implements GlobalFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        log.info("Path of the request recieved: {}", exchange.getRequest().getPath());
        return chain.filter(exchange);
    }
}
```

**Šta rade anotacije:**
- `@Component` — registruje klasu kao bean; **svaki `GlobalFilter` bean gateway automatski uključi u lanac**, ne treba ga nigde „prijavljivati“.
- `@Slf4j` (Lombok) — generiše `private static final Logger log`.

**Šta je šta u kodu:**
- `GlobalFilter` — filter koji se izvršava za **svaki** zahtev kroz gateway (za razliku od `GatewayFilter`-a koji se vezuje za pojedinačnu rutu).
- `ServerWebExchange exchange` — reaktivna „koverta“ koja nosi i **request** i **response**.
- `chain.filter(exchange)` — „pusti dalje“ kroz lanac. Ako ovo ne pozoveš, zahtev **staje ovde**.
- `Mono<Void>` — reaktivni tip: „operacija koja se završi bez rezultata“.

#### Napomene:
- Ovaj filter nije tražen zadatkom, ali je koristan: u konzoli se lepo vidi da je zahtev uopšte stigao do gateway-a.
- „recieved“ je slovna greška iz originalnog rešenja — prenosimo je verno da bi kod bio 1:1. 🙂

### Korak 5 — `AuthenticationFilter` (globalni filter #2) — **jezgro zadatka**

```java
package com.rzk.api_gateway.filter;

import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

@Component
public class AuthenticationFilter implements GlobalFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String apiKey = exchange.getRequest().getHeaders().getFirst("x-api-key");

        if (apiKey == null || !apiKey.equals("validKey123")) {
            exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
            return exchange.getResponse().setComplete();
        }

        return chain.filter(exchange);
    }
}
```

**Logika, red po red:**
1. `getHeaders().getFirst("x-api-key")` — pročitaj header (case-insensitive, može i `X-API-KEY`).
2. Ako header **ne postoji** (`null`) ili **vrednost nije** `validKey123` → postavi status **403 FORBIDDEN** i `setComplete()` = „zatvori odgovor **odmah**“. Lanac se prekida, zahtev **nikad ne stiže** do mikroservisa.
3. Inače → `chain.filter(exchange)`, zahtev ide dalje.

#### Napomene:
- Provera `apiKey == null` **mora** biti prva — bez nje bi `apiKey.equals(...)` bacio `NullPointerException` (i vratio 500 umesto 403).
- Telo odgovora je prazno — samo status 403. To je i traženo.
- Pošto je ovo **globalni** filter, važi za **sve** rute, uključujući i one automatski generisane preko discovery locator-a.

### Korak 6 — `ApiGatewayConfig` (rute)

Novi paket `config`:

```java
package com.rzk.apigateway.config;

import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ApiGatewayConfig {

    @Bean
    public RouteLocator gatewayRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
                .route("home", r -> r.path("/home")
                        .filters(f -> f.rewritePath("/home", "/api/animals")
//                                .addRequestHeader("x-api-key", "validKey123")
                        )
                        .uri("lb://animal-service"))
                .route("animal-service", r -> r.path("/api/animals/**")
                        .filters(f -> f.addRequestHeader("x-api-key", "validKey123"))
                        .uri("lb://animal-service"))
                .route("vet-service", r -> r.path("/api/vet-records/**")
//                        .filters(f -> f.addRequestHeader("x-api-key", "validKey123"))
                        .uri("lb://vet-service"))
                .build();
    }
}
```

**Šta rade anotacije:**
- `@Configuration` — klasa sadrži definicije bean-ova.
- `@Bean` — Spring uzima vraćeni `RouteLocator` kao tabelu ruta gateway-a.

**Anatomija jedne rute:**

| Deo | Značenje |
|---|---|
| `.route("home", ...)` | ID rute (proizvoljan string, vidi se u logovima) |
| `r.path("/home")` | **Predikat** — uslov koji zahtev mora da ispuni da bi ruta „upalila“ |
| `.filters(f -> f.rewritePath(...))` | **Filteri te rute** — menjaju zahtev/odgovor pre prosleđivanja |
| `.uri("lb://animal-service")` | Odredište; `lb://` = razreši ime preko Eureke + load balansiraj |

**Tri rute:**

| Ruta | Predikat | Filter | Ide na |
|---|---|---|---|
| `home` | `/home` | `rewritePath("/home", "/api/animals")` | `lb://animal-service` |
| `animal-service` | `/api/animals/**` | `addRequestHeader("x-api-key", "validKey123")` | `lb://animal-service` |
| `vet-service` | `/api/vet-records/**` | — | `lb://vet-service` |

**`rewritePath(regex, replacement)`** — prepiše putanju pre slanja. Klijent zove `GET localhost:8765/home`, a animal-service zapravo dobije `GET /api/animals`. Ovim je rešena prva stavka zadatka.

☝️🤓 **`**` u `/api/animals/**`** hvata **sve** ispod, uključujući `/api/animals/animal-records/5` i `/api/animals/animal-info-with-records/5`. Putanja se prosleđuje **nepromenjena** (nema rewrite-a), pa se poklapa sa `@RequestMapping("/api/animals")` u kontroleru. Zato ovde rewrite **nije** potreban.

#### Napomene — zakomentarisane linije ☝️🤓

Obrati pažnju: `addRequestHeader("x-api-key", "validKey123")` je **aktivan** samo na `animal-service` ruti, a **zakomentarisan** na `home` i `vet-service`. To nije slučajno — to je „prekidač“ za demonstraciju.

Kad je taj filter aktivan, **gateway sam ubacuje `x-api-key` u zahtev**. Zajedno sa globalnim `AuthenticationFilter`-om to znači da rezultat zavisi od **redosleda filtera**: rutni filteri (`AddRequestHeader`) i globalni filteri se sortiraju u **jedan zajednički lanac** po `Order` vrednosti. `AuthenticationFilter` ne implementira `Ordered`, pa mu Spring dodeljuje najniži prioritet (izvršava se **kasno**), dok rutni `AddRequestHeader` ide **rano**.

**Praktična posledica:** na ruti `/api/animals/**` header je već ubačen dok filter stigne na proveru → **prolazi i bez `x-api-key` od klijenta**. Na `/home` i `/api/vet-records/**` (gde je linija zakomentarisana) **403 se uredno vraća**.

**Obavezno proveri ponašanje kod sebe** (kroz Talend, vidi sledeći korak) — a ako ti treba „čist“ zadatak u kome sve tri rute traže ključ, samo **zakomentariši i tu jednu liniju** na `animal-service` ruti. Tada je `x-api-key` obavezan svuda, kao što zadatak i traži.

## Zadatak 2 — animal-service + Resilience4J

*Dodajemo Resilience4J i tri „štita“ na tri endpoint-a, svaki sa svojom fallback metodom. 

Kopiraj `animal-service` i `eureka-server` sa vežbi 4.

### Korak 8 — `pom.xml` (dodavanje Resilience4J)

U postojeći `animal-service/pom.xml`, uz `data-jpa`, `webmvc`, `mysql`, `lombok`, `openfeign` i `eureka-client`, dodaj:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
```

#### Napomene:
- U Initializr-u se ta zavisnost zove **Resilience4J** (kategorija *Spring Cloud Circuit Breaker*).
- Ovaj starter povlači i **AOP** i anotacione aspekte (`@Retry`, `@RateLimiter`, `@CircuitBreaker`), pa ništa dodatno ne treba dodavati.
- Posle izmene: **Maven → Sync Project**.

### Korak 9 — `application.properties`

```properties
spring.application.name=animal-service
server.port=8081

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

**Struktura ključa je uvek ista:**
```
resilience4j.<pattern>.instances.<IME_INSTANCE>.<parametar>
```
`<IME_INSTANCE>` je ono što upisuješ u `name = "..."` u anotaciji. Ovde su to `animalRecords`, `allAnimals`, `animalInfoWithRecords`.

**Šta znači koji parametar:**

| Parametar | Vrednost | Značenje |
|---|---|---|
| `retry...animalRecords.max-attempts` | 5 | Ukupno **5 pokušaja** (1 prvi + 4 ponovljena) |
| `retry...animalRecords.wait-duration` | 2s | Pauza između pokušaja |
| `ratelimiter...allAnimals.limit-for-period` | 2 | Najviše **2 poziva** po prozoru |
| `ratelimiter...allAnimals.limit-refresh-period` | 60s | Dužina prozora — dakle **2 zahteva u 60s** |
| `circuitbreaker...minimum-number-of-calls` | 20 | Tek posle **20 poziva** osigurač uopšte počinje da računa procenat grešaka |
| `circuitbreaker...wait-duration-in-open-state` | 5s | Koliko ostaje **OPEN** pre nego što pređe u **HALF_OPEN** |

☝️🤓 **Ono što NIJE napisano — defaultne vrednosti circuit breaker-a:**
- `failure-rate-threshold = 50` → OPEN kad **50%+** poziva pukne
- `sliding-window-type = COUNT_BASED`, `sliding-window-size = 100` → gleda se poslednjih 100 poziva
- `permitted-number-of-calls-in-half-open-state = 10`

Praktično: dok vet-service ne radi, **svih** 20 poziva puca → 100% > 50% → osigurač se **otvara na 20. pozivu**.

☝️🤓 **`spring.datasource.hikari.maximum-pool-size=2`** nema veze sa Resilience4J — to je ograničenje broja konekcija ka bazi. Bitno jer ćemo pokretati **više instanci** animal-service-a, a školska MySQL baza ima ograničen broj konekcija po korisniku. Ne diraj.

### Korak 10 — `AnimalController` (RateLimiter, Retry, Circuit Breaker + fallback)

```java
package com.rzk.animal_service.controller;

import com.rzk.animal_service.dto.AnimalRecord;
import com.rzk.animal_service.dto.VetRecordDto;
import com.rzk.animal_service.model.Animal;
import com.rzk.animal_service.service.AnimalService;
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.ratelimiter.annotation.RateLimiter;
import io.github.resilience4j.retry.annotation.Retry;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.Collections;
import java.util.List;

@Slf4j
@RestController
@RequiredArgsConstructor
@RequestMapping("/api/animals")
public class AnimalController {

    private final AnimalService animalService;
    int circuitBreakerCounter = 0;

    @GetMapping
    @RateLimiter(name = "allAnimals", fallbackMethod = "getAnimalsFallback")
    public ResponseEntity<List<Animal>> getAllAnimals() {
        return new ResponseEntity<>(animalService.getAllAnimals(), HttpStatus.OK);
    }

    @PostMapping
    public ResponseEntity<Animal> addAnimal(@RequestParam("id-species") Integer idSpecies, @RequestBody Animal animal) {
        Animal savedAnimal = animalService.addAnimal(idSpecies, animal);
        if (savedAnimal == null) {
            return ResponseEntity.status(HttpStatus.NOT_FOUND).build();
        }
        return new ResponseEntity<>(savedAnimal, HttpStatus.CREATED);
    }

    @GetMapping("/animal-records/{idAnimal}")
    @Retry(name = "animalRecords", fallbackMethod = "getAnimalRecordsFallback")
    public ResponseEntity<List<VetRecordDto>> getAnimalRecords(@PathVariable Integer idAnimal) {
        log.info("API call received!");
        return new ResponseEntity<>(animalService.getAnimalRecords(idAnimal), HttpStatus.I_AM_A_TEAPOT);
    }

    @GetMapping("/animal-info-with-records/{idAnimal}")
    @CircuitBreaker(name = "animalInfoWithRecords", fallbackMethod = "getAnimalInfoWithRecordsFallback")
    public ResponseEntity<AnimalRecord> getAnimalInfoWithRecords(@PathVariable Integer idAnimal) {
        log.info("API call for circuit breaker received! ->{}", circuitBreakerCounter += 1);
        AnimalRecord animalRecord = animalService.getAnimalInfoWithRecords(idAnimal);
        if (animalRecord == null) {
            return ResponseEntity.status(HttpStatus.NOT_FOUND).build();
        }
        return ResponseEntity.ok(animalRecord);
    }

    public ResponseEntity<List<Animal>> getAnimalsFallback(Exception ex) {
        return new ResponseEntity<>(HttpStatus.REQUEST_TIMEOUT);
    }

    public ResponseEntity<List<VetRecordDto>> getAnimalRecordsFallback(Exception ex) {
        return new ResponseEntity<>(Collections.emptyList(), HttpStatus.REQUEST_TIMEOUT);
    }

    public ResponseEntity<AnimalRecord> getAnimalInfoWithRecordsFallback(Exception ex) {
        return new ResponseEntity<>(HttpStatus.REQUEST_TIMEOUT);
    }
}
```

**Šta rade anotacije:**

| Anotacija | Šta radi |
|---|---|
| `@RestController` | REST kontroler; povratne vrednosti idu direktno u telo odgovora (JSON) |
| `@RequestMapping("/api/animals")` | Prefiks za **sve** endpoint-e u klasi |
| `@RequiredArgsConstructor` | Lombok: konstruktor za sva `final` polja → **konstruktorska injekcija** `AnimalService`-a |
| `@Slf4j` | Lombok: `log` objekat |
| `@RateLimiter(name = "allAnimals", fallbackMethod = "getAnimalsFallback")` | Vezuje metodu za instancu `allAnimals` iz properties-a (2/60s). Višak → `RequestNotPermitted` → fallback |
| `@Retry(name = "animalRecords", fallbackMethod = "getAnimalRecordsFallback")` | Instanca `animalRecords`: 5 pokušaja, 2s pauze. Ako svi puknu → fallback |
| `@CircuitBreaker(name = "animalInfoWithRecords", fallbackMethod = "...")` | Instanca `animalInfoWithRecords`. Kad je OPEN → `CallNotPermittedException` → fallback, **bez ulaska u telo metode** |
| `@GetMapping` / `@PostMapping` / `@PathVariable` / `@RequestParam` / `@RequestBody` | Standardno mapiranje HTTP-a |

**Endpoint-i:**

| Metod | Putanja | Zaštita | Uspeh | Fallback |
|---|---|---|---|---|
| `getAllAnimals` | `GET /api/animals` | RateLimiter | **200** + lista | **408**, prazno telo |
| `addAnimal` | `POST /api/animals?id-species=X` | — | **201** / **404** | — |
| `getAnimalRecords` | `GET /api/animals/animal-records/{idAnimal}` | Retry | **418** + lista kartona | **408** + prazna lista |
| `getAnimalInfoWithRecords` | `GET /api/animals/animal-info-with-records/{idAnimal}` | CircuitBreaker | **200** / **404** | **408**, prazno telo |

#### Napomene i zamke ☝️🤓

- **`HttpStatus.I_AM_A_TEAPOT` (418)** nije greška u kucanju. To je RFC-ov šaljivi status (*„ja sam čajnik“*), namerno stavljen da bi se u Talend-u **na prvi pogled** razlikovao **uspešan poziv (418)** od **fallback-a (408)**. Verno ga prenosimo. U ozbiljnom kodu bi tu, naravno, stajao `HttpStatus.OK`.

- **`int circuitBreakerCounter = 0`** — brojač poziva koji su **stvarno ušli** u telo metode. Kad se osigurač otvori, log prestaje da se ispisuje i brojač staje. To je **glavni dokaz** da circuit breaker radi. (Polje nije `private` i nije thread-safe — sasvim ok za demonstraciju, ne za produkciju.)

- **Fallback metode moraju imati isti povratni tip** kao originalne (`ResponseEntity<List<Animal>>`, `ResponseEntity<List<VetRecordDto>>`, `ResponseEntity<AnimalRecord>`) i **`Exception ex`** kao poslednji parametar. Ako se tipovi ne poklope, Resilience4J baca `NoSuchMethodException` **u vreme izvršavanja**, ne kompajliranja — kompajler te neće upozoriti!

- **Fallback ne prima `idAnimal`.** Potpis fallback-a je `(Exception ex)`, a ne `(Integer idAnimal, Exception ex)`. Resilience4J prihvata i varijantu **samo sa izuzetkom** — ali onda unutar fallback-a **nemaš pristup ulaznim parametrima**. Ovde nam ni ne trebaju, jer se vraća prazan odgovor.

- **`REQUEST_TIMEOUT` (408) je diskutabilan izbor** za rate limiting (semantički bi bio tačniji **429 Too Many Requests**) i za otvoren osigurač (**503 Service Unavailable**). Ali profesorkino rešenje koristi 408 za sva tri, pa ga tako i prenosimo — poenta vežbe je da fallback **vrati drugačiji status od uspešnog poziva**.

- **`@CircuitBreaker` i vraćanje 404:** kad životinja ne postoji, metoda **uredno vraća** `ResponseEntity 404` — to **nije izuzetak**, pa se osigurača **ne tiče** (broji se kao *uspešan* poziv). Osigurač broji samo **bačene izuzetke** (kod nas: Feign puca jer je vet-service pao).

- **Redosled aspekata (default):** `Retry` ➜ `CircuitBreaker` ➜ `RateLimiter`. Kod nas je svaki endpoint zaštićen samo **jednim** obrascem, pa se ne preklapaju — ali dobro je znati da bi, da su kombinovani, Retry bio spoljašnji (ponavljao bi ceo poziv).

### Korak 11 — `AnimalService` (nema izmena)

Novi endpoint `GET /api/animals` koristi metodu koja **već postoji** još od vežbi 4:

```java
public List<Animal> getAllAnimals() {
    return animalRepository.findAll();
}
```

`AnimalRepository` nasleđuje `JpaRepository<Animal, Integer>`, pa `findAll()` dolazi „iz kutije“ — **ništa se ne dodaje** ni u repozitorijum ni u servis. Jedino što je novo jeste `@GetMapping` metoda u kontroleru (i `@RateLimiter` na njoj).

Ostale dve metode koje Resilience4J štiti su takođe nepromenjene:

```java
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
```

☝️🤓 Obe zovu **`vetProxy`** (Feign → vet-service). **To je tačka koja može da pukne** — i zato baš na njima ima smisla Retry i Circuit Breaker. Kad **ugasiš vet-service**, Feign baca izuzetak i mehanizmi se aktiviraju. To je ceo trik testiranja u sledećem koraku.

### Korak 12 — Testiranje Resilience4J

Testiraj **direktno na 8081** (ili kroz gateway sa `x-api-key`) — svejedno; direktno je jednostavnije za praćenje logova.

#### A) RateLimiter — `GET /api/animals` (2 zahteva / 60s)

```
GET http://localhost:8081/api/animals
```

- 1. poziv → **200 OK** + lista životinja
- 2. poziv → **200 OK**
- **3. poziv (u istom minutu)** → **408 REQUEST_TIMEOUT**, prazno telo ← fallback!
- Sačekaj da prođe 60s → opet **200 OK**

☝️🤓 Prozor se osvežava **na svakih 60s od starta aplikacije**, ne „60s od tvog prvog klika“. Ako ti se čini da si dobio 3 poziva umesto 2 — verovatno si pogodio granicu prozora.

#### B) Retry — `GET /api/animals/animal-records/{idAnimal}` (5 pokušaja, 2s)

**Prvo sa upaljenim vet-service-om:**
```
GET http://localhost:8081/api/animals/animal-records/1
```
→ **418 I_AM_A_TEAPOT** + lista kartona. U logu **jednom**: `API call received!`

**Sad UGASI vet-service** i pozovi ponovo:
→ Odgovor stiže tek posle **~8 sekundi** (4 pauze × 2s), pa **408 REQUEST_TIMEOUT** + `[]`.
U konzoli će se `API call received!` ispisati **5 puta** — to su ti pokušaji. 🎯

#### C) Circuit Breaker — `GET /api/animals/animal-info-with-records/{idAnimal}`

**Sa upaljenim vet-service-om:**
```
GET http://localhost:8081/api/animals/animal-info-with-records/1
```
→ **200 OK** + `{ "animal": {...}, "vetRecords": [...] }`

**UGASI vet-service** i onda spamuj isti poziv:

| Poziv | Stanje | Odgovor | Log |
|---|---|---|---|
| 1–19 | **CLOSED** (skuplja se uzorak) | 408 (fallback) | `->1`, `->2`, ... `->19` |
| 20 | **CLOSED → OPEN** (20 poziva, 100% grešaka) | 408 | `->20` |
| 21, 22, ... | **OPEN** | 408 **odmah** | **log se NE ispisuje!** |
| posle 5s | **HALF_OPEN** | propušta probne pozive | log se opet javlja |

**Ključno zapažanje:** od 21. poziva **brojač staje** i log nestaje — zahtev **uopšte ne ulazi u metodu**, osigurač ga preseca. Ako sad **upališ vet-service** i sačekaš 5s, prvi sledeći pozivi (HALF_OPEN) uspeju → osigurač se **zatvara** → sve opet radi normalno.

---

## Load Balancing kroz gateway

Zadatak traži i da rute koriste **Load Balancer** — to je već rešeno sa `lb://animal-service` (a ne `http://localhost:8081`). Evo kako se to **dokazuje**:

### Pokretanje druge instance animal-service-a (IntelliJ)

1. **Run → Edit Configurations…**
2. Desni klik na `AnimalServiceApplication` → **Copy Configuration**
3. Ime: `AnimalServiceApplication (8091)`
4. U **Program arguments** upiši:
   ```
   --server.port=8091
   ```
5. **Apply → Run**

Na `http://localhost:8761` sada `ANIMAL-SERVICE` ima **2 instance** (8081 i 8091).

### Dokaz

Zovi kroz gateway više puta:
```
GET http://localhost:8765/api/animals
Header: x-api-key: validKey123
```
U konzolama **obe** instance animal-service-a smenjivaće se logovi — gateway round-robin-om deli zahteve. (Isto važi i za `/home`.)

☝️🤓 Ako želiš baš „opipljiv“ dokaz kao na vežbama 4, dodaj privremeni debug endpoint u `AnimalController` koji vraća port:

```java
@Value("${local.server.port}")
private String port;

@GetMapping("/debug")
public String debug() {
    return "animal-service instanca na portu: " + port;
}
```
Pa zovi `GET localhost:8765/api/animals/debug` sa `x-api-key` — port se **smenjuje** između 8081 i 8091.
*(Ovo nije deo profesorkinog rešenja — dodatak radi provere.)*

☝️🤓 **Dva nivoa load balancinga u sistemu:** gateway → animal-service (preko `lb://`), i animal-service → vet-service (preko Feign-a, koji je load-balanced po defaultu). Oba se oslanjaju na Eureku.

---

