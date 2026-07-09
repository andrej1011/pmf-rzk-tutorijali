# RZK Vežbe 1 - TUTORIJAL
## Setup projekta
---
### Korak 1 — Kreiranje projekta

Otvori https://start.spring.io/ i podesi kao na slici:
![Kreiranje aplikacije](rzk-tutorijal-slike/kreiranje-aplikacije.png)

Klikni **GENERATE** → skida se ZIP → raspakuj.

*Sačuvaj zip. Za slučaj da nešto pogrešiš, samo ponovo raspakuj zip i kreni ispočetka.*
### Korak 2 — Otvaranje u IntelliJ-u

1. **File → Open** → izaberi raspakovan folder  → **Open as Project**.
2. Sačekaj da Maven povuče zavisnosti (progress bar dole).
### Korak 3 — application.properties (konekcija na bazu)

Otvori `src/main/resources/application.properties` i unesi:

```properties
spring.application.name=ergela-service

spring.datasource.url=jdbc:mysql://nastava.is.pmf.uns.ac.rs:3306/rzk
spring.datasource.username=rzk
spring.datasource.password=rzkStudent2019!
spring.jpa.hibernate.naming.physical-strategy=org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl

spring.datasource.hikari.maximum-pool-size=2
```

#### Šta ovo znači:
- `datasource.url/username/password` — adresa i kredencijali MySQL baze `rzk`
- `physical-strategy=...StandardImpl` — Hibernate **ne dira** nazive kolona (default bi `dateOfBirth` pretvorio u `date_of_birth`, a kolona u bazi se zove baš `dateOfBirth` → bez ovoga puca!)
- `hikari.maximum-pool-size=2` — max 2 konekcije (deljena fakultetska baza)

### Korak 4 — Konekcija baze u IntelliJ + Reverse engineering

Entitete **ne kucamo od nule** — generišemo ih iz tabela (reverse engineering):

1. Desni panel **Database** → **+** → _Data Source → MySQL_ → unesi:

| host     | nastava.is.pmf.uns.ac.rs |
| -------- | ------------------------ |
| port     | 3306                     |
| user     | rzk                      |
| password | rzkStudent2019!          |
| database | rzk                      |

![Setup databaze](rzk-tutorijal-slike/setup-baza.png)

 → **Test Connection** → OK.
 
2. desni klik na tabelu → **Reverse Engineering** → selektuj svih 5 tabela → generiši entitete u paket `com.rzk.ergela_services.model`.

[Setup reverse engineer](rzk-tutorijal-slike/setup-reverse-eng.png)

2. Generator napravi i `@OneToMany` obrnute strane veza — njih zadržavamo (koristimo ih kasnije u JPQL upitima).


### Korak 5 — Paketi

U `src/main/java/com/rzk/ergela_services` napravi 4 paketa (desni klik → New → Package):
```
model (u njega smesti generisane model klase)  
repository
service
controller
```


## Zadatak 1 
---
*"/riders - Dobavljanje liste svih jahača"*

### ☝️🤓
Kreiranje repozitorijuma i servisa je potpuno isto kao na RIS-u.
### Model Rider

Model klase ne moramo da pišemo ručno. Već smo ih dobili.
Kroz setup projekta, generisali smo modele kroz JPA entitete.

Model `Rider.java` treba da izgleda ovako:
```java
package com.rzk.ergela_services.model;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;

import java.time.LocalDate;

@Getter
@Setter
@Entity
@Table(name = "Rider")
public class Rider {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id", nullable = false)
    private Integer id;

    @Column(name = "name", nullable = false, length = 45)
    private String name;

    @Column(name = "surname", nullable = false, length = 45)
    private String surname;

    @Column(name = "address", length = 45)
    private String address;

    @Column(name = "dateOfBirth")
    private LocalDate dateOfBirth;
}
```

Šta rade anotacije:
- `@Entity` — klasa se mapira na tabelu (isto ime → `rider`)
- `@Getter @Setter` (Lombok) — generiše sve gettere/settere automatski (bez kucanja)
- `@Id` — primarni ključ
- `@GeneratedValue(IDENTITY)` — auto-increment (bazu radi sama)
- `@Column(name=..., nullable=..., length=...)` — mapiranje na tačnu kolonu u bazi


### Rider Repository

Kreiramo novi interfejs `RiderRepository`
`src/main/java/com/rzk/ergela_services/repository/RiderRepository.java`:

```java
package com.rzk.ergela_services.repository;

import com.rzk.ergela_services.model.Rider;
import org.springframework.data.jpa.repository.JpaRepository;

public interface RiderRepository extends JpaRepository<Rider, Integer> {
}
```

Šta se dešava:
- `JpaRepository<Rider, Integer>` — `Rider` je entitet, `Integer` je tip primarnog ključa
- Nasleđivanjem **besplatno** dobijaš: `findAll()`, `findById()`, `save()`, `deleteById()`, `count()`... — bez ijedne linije koda
- Za zadatak 1 dovoljan je `findAll()`

### ErgelaService (poslovna logika)
Kreiramo novu klasu `ErgelaService` sa anotacijom `@Service`.

`src/main/java/com/rzk/ergela_services/service/ErgelaService.java`:
```java
package com.rzk.ergela_services.service;

import com.rzk.ergela_services.model.Rider;
import com.rzk.ergela_services.repository.RiderRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
@RequiredArgsConstructor
public class ErgelaService {

    private final RiderRepository riderRepository;

    public List<Rider> getAllRiders() {
        return riderRepository.findAll();
    }
}
```

Šta rade anotacije:
- `@Service` — Spring registruje klasu kao servis bean (može se injektovati)
- `@RequiredArgsConstructor` (Lombok) — generiše konstruktor za sva `final` polja → Spring kroz njega injektuje `RiderRepository` (konstruktorska injekcija, bez `@Autowired`)
- Metoda `getAllRiders()` — samo poziva repozitorijum; kad zatreba filtriranje/logika, ide ovde

### Controller (REST endpoint)
Kreiramo novu klasu `ErgelaController` sa anotacijom `@Controller`.

`src/main/java/com/rzk/ergela_services/controller/ErgelaController.java`:
```java
package com.rzk.ergela_services.controller;

import com.rzk.ergela_services.model.Rider;
import com.rzk.ergela_services.service.ErgelaService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequiredArgsConstructor
@RequestMapping("/api")
public class ErgelaController {

    private final ErgelaService ergelaService;

    @GetMapping("/riders")
    @ResponseStatus(HttpStatus.OK)
    public List<Rider> getAllRiders() {
        return ergelaService.getAllRiders();
    }
}
```

Šta rade anotacije:
- `@RestController` — REST kontroler; povratna vrednost metode **automatski** postaje JSON u telu odgovora
- `@RequestMapping("/api")` na klasi — **prefiks** za sve endpoint-e; svaka putanja počinje sa `/api/...`
- `@RequiredArgsConstructor` — injektuje `ErgelaService`
- `@GetMapping("/riders")` — mapira `GET /api/riders` na ovu metodu
- `@ResponseStatus(HttpStatus.OK)` — status **200 OK** u odgovoru
- Kontroler nema logiku — samo poziva servis

### Pokretanje projekta

Otvori `ErgelaServicesApplication.java` (Spring Initializr ga je već napravio, ne diraj):

```java
package com.rzk.ergela_services;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ErgelaServicesApplication {

    public static void main(String[] args) {
        SpringApplication.run(ErgelaServicesApplication.class, args);
    }
}
```

Zeleno **Run** dugme (ili Shift+F10). U konzoli treba da vidiš:
```
Tomcat started on port 8080
Started ErgelaServicesApplication in X.XXX seconds
```

### Testiranje u TalendAPI

```
GET http://localhost:8080/api/riders
```

Očekivano: **status 200** + JSON niz svih jahača iz tabele `rider`:

```json
[
    {
        "id": 1,
        "name": "John",
        "surname": "Doe",
        "address": "Some Street 1",
        "dateOfBirth": "1995-03-15"
    },
    {
        "id": 2,
        "name": "Jane",
        "surname": "Smith",
        "address": "Other Street 5",
        "dateOfBirth": "1998-07-22"
    }
]
```

Ili u browser-u — GET direktno radi:
```
http://localhost:8080/api/riders
```

---

## Šta se desilo pod haubom

1. **Talend API** šalje `GET /api/riders`
2. **DispatcherServlet** (Spring) prima zahtev → traži metodu mapiranu na `/api/riders` (GET)
3. Pronalazi `ErgelaController.getAllRiders()` → poziva je
4. Metoda poziva `ergelaService.getAllRiders()`
5. Servis poziva `riderRepository.findAll()`
6. **Hibernate** generiše `SELECT * FROM rider` i šalje MySQL bazi
7. Rezultati se mapiraju u `List<Rider>`
8. **Jackson** (auto) serijalizuje listu u JSON
9. Odgovor: status 200 + JSON telo

---

## Struktura projekta posle zadatka 1

```
ergela-services/
├── pom.xml
└── src/main/
    ├── java/com/rzk/ergela_services/
    │   ├── ErgelaServicesApplication.java
    │   ├── model/
    │   │   └── Rider.java
    │   ├── repository/
    │   │   └── RiderRepository.java
    │   ├── service/
    │   │   └── ErgelaService.java
    │   └── controller/
    │       └── ErgelaController.java
    └── resources/
        └── application.properties
```

Postavljen je temelj — sve što treba za sledeće zadatke je da dodaješ nove entitete, metode u repozitorijume, metode u `ErgelaService` i nove endpoint-e u `ErgelaController`.

## Zadatak 2
---
*`/horses/1` - Dobavljanje konja 1*

### Napravi prazan `HorseRepository`

```java
package com.rzk.ergela_service.repository;  
  
import com.rzk.ergela_service.model.Horse;  
import org.springframework.data.jpa.repository.JpaRepository;  
  
public interface HorseRepository extends JpaRepository<Horse,Integer> {  
  
}
```

### Dodaj metodu u Service
```java
public Optional<Horse> getHorseById(Integer id){
	return horseRepository.findById(id);
}
```

### Dodaj metodu u Controller
```java
 @GetMapping("/horses/{idHorse}")
    public ResponseEntity<Horse> getHorse(@PathVariable Integer idHorse) {
        Optional<Horse> horseOptional = ergelaService.getHorse(idHorse);
        if (horseOptional.isEmpty()) {
            return new ResponseEntity<>(HttpStatus.NOT_FOUND);
        }
        return new ResponseEntity<>(horseOptional.get(), HttpStatus.OK);
    }
```

### Testiraj u Talend API
```
GET http://localhost:8080/api/horses/1
```

## Zadatak 3
---
*`/riders` - Dodavanje novog jahača, vratiti status 201 (created) i dodatog jahača*

POST metoda

- Ne moramo ništa dodavati u repository

### Dodaj metodu u Service

```java
public Rider addRider(Rider rider){  
    return riderRepository.save(rider);  
}
```

### Dodaj metodu u Controller

```java
@PostMapping("/riders")  
public ResponseEntity<Rider> addRider(@RequestBody Rider rider){  
    return new ResponseEntity< (ergelaService.addRider(rider),HttpStatus.CREATED);  
}
```

### Testiraj novi endpoint na TalendAPI

```
POST http://localhost:8080/api/riders
```

Request body:
```
{
"address": "123 Green St",
"dateOfBirth": "1990-03-15",
"name": "John",
"surname": "Doe Jr."
}
```

## Zadatak 4
---
*`/riders/1/horse`- Servis prihvata ceo objekat konja. Dodati konja jahaču 1 u omiljene konje.
Ukoliko konj (proveriti po jedinstvenom atributu fullName ili jahač ne postoje u bazi, vratiti
status 404 (not found). Inače vratiti novonastali objekat tipa Favorite*

### Objašnjenje zadatka
1. Prvo proveriti postoji li `rider{id}`
2. Onda proveriti postoji li konj sa `fullName()`
3. U objekat `Favorite` dodati objekat `rider` i objekat `horse`
4. Vratiti objekat kontroleru koji će ga vratiti korisniku

### 1. Kreirati prazan`FavoriteRepository`

```java
package com.rzk.ergela_service.repository;  
  
import com.rzk.ergela_service.model.Favorite;  
import org.springframework.data.jpa.repository.JpaRepository;  
  
public interface FavoriteRepository extends JpaRepository<Favorite,Integer> {  
  
}
```

### 2. `ErgelaService` kreirati metodu `addHorseToRider`

```java
public Favorite addHorseToRider(Horse horse, Integer idRider){  
    Optional<Rider> rider = riderRepository.findById(idRider);  
    if(rider.isEmpty()){  
        return null;  
    }  
    Optional<Horse> realHorse = horseRepository.findDistinctByFullName(horse.getFullName());  
    if(realHorse.isEmpty()){  
        return null;  
    }  
    Favorite favorite = new Favorite();  
    favorite.setHorse(realHorse.get());  
    favorite.setRider(rider.get());  
    return favoriteRepository.save(favorite);  
  
}
```

#### Napomene:
- `return null` ukoliko ne postoji jahač ili konj sa tim imenom
		Tako u kontroleru lako proveravamo da li treba da vratimo `404 NOT FOUND`
- `return favoriteRepository.save(favorite);`
		Moramo dodati `favourite` objekat u databazu, ne samo vratiti objekat.

### 3. `ErgelaController` kreirati metodu `addHorseToRider`
```java
@PostMapping("/riders/{id}/horse")  
public ResponseEntity<Favorite> addHorseToRider(@PathVariable Integer id, @RequestBody Horse horse){  
    Favorite returnFavorite = ergelaService.addHorseToRider(horse,id);  
    if(returnFavorite==null)  
        return new ResponseEntity<>(HttpStatus.NOT_FOUND);  
    return new ResponseEntity<>(returnFavorite,HttpStatus.CREATED);  
}
```

#### Napomene:
- proveravamo da li je `returnFavorite==null`
	- prethodno smo postavili uslove u servisu
	- vraćamo `404 NOT FOUND`
- `@PathVariable` varijabla unutar endpoint-a
- `@RequestBody` JSON objekat unutar POST Requesta

### 4. Testiramo endpoint preko TalendAPI

URL:
```
POST http://localhost:8080/api/riders/2/horse
```

Request Body:
```json
{
    "dateOfBirth": "2010-09-05",
    "fullName": "Black Beauty",
    "gender": "F",
    "id": 4,
    "nickname": "Beauty"
}
```

#### Napomene:
- Ukoliko dobijemo grešku `Web server failed to start. Port 8080 was already in use.`
```bash
lsof -i :8080
kill -9 <PID>
```
Pronalazimo `PID` web servera i ubijamo ga. RESETOVANJE IntelliJ-a NE RADI.

## Zadatak 5
---
*`/sessions/1/time/14` - Za trening 1 ažurirati vreme. Vreme je tipa String - proveriti da li ima tačno dva karaktera, ako nema, vratiti status 400 (bad request). Inače vratiti ažuriran objekat*

### 1. Kreirati prazan`SessionRepository`

```java
package com.rzk.ergela_service.repository;  
  
import com.rzk.ergela_service.model.Session;  
import org.springframework.data.jpa.repository.JpaRepository;  
  
public interface SessionRepository extends JpaRepository<Session, Integer> { 
 
}
```

### 2. Kreirati metod `changeSessionTime` u `ErgelaService`

```java
public Session changeSessionTime(Integer idSession, String time){  
    Optional<Session> session = sessionRepository.findById(idSession);  
    if(session.isEmpty()){  
        return null;  
    }  
    session.get().setTime(time);  
    return sessionRepository.save(session.get());  
}
```

### 3. Kreirati metod `changeSessionTime` u `ErgelaController`
```java
@PostMapping("/sessions/{sessionId}/time/{time}")
public ResponseEntity<Session> changeSessionTime(@PathVariable Integer sessionId, @PathVariable String time){  
    //Ovde proveravamo validnost stringa  
    if (time.length() != 2) {  
        return new ResponseEntity<>(HttpStatus.BAD_REQUEST); //HTTP 400  
    }  
    Session returnSession = ergelaService.changeSessionTime(sessionId,time);  
    if(returnSession==null)  
        return new ResponseEntity<>(HttpStatus.NOT_FOUND);  
    return new ResponseEntity<>(returnSession,HttpStatus.OK);  
}
```

- Pošto u servisu možemo da vratimo samo `Session` objekat ili `null`, String length proveravamo direktno u kontroleru. Možda čak i ispravnije jer je to input validation.

### 4. Testirati novi endpoint na Talend API

URL:
```
POST http://localhost:8080/api/sessions/1/time/14
```

## Zadatak 6
---
*`/riders/1/horses` - Dobavljanje svih omiljenih kobila (gender je F) jahača 1*

### Objašnjenje zadatka
1. Proveriti postoji li `rider{id}`
2. Iz njegovih omiljenih konja (`Favorite`) izdvojiti samo one čiji je `gender == "F"`
3. Vratiti listu tih konja

### JPQL
Ovo je prvi zadatak gde nam standardne metode (`findAll`, `findById`...) nisu dovoljne — treba nam **prilagođen upit**. Pišemo ga u repozitorijumu preko anotacije `@Query`, jezikom **JPQL** (radi nad *entitetima i njihovim poljima*, a ne nad tabelama i kolonama kao čist SQL).

### 1. Dodaj JPQL upit u `HorseRepository`

```java
package com.rzk.ergela_service.repository;

import com.rzk.ergela_service.model.Horse;
import com.rzk.ergela_service.model.Rider;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.List;

public interface HorseRepository extends JpaRepository<Horse, Integer> {

    @Query("SELECT f.horse FROM Favorite f WHERE f.rider = :rider AND f.horse.gender = 'F'")
    List<Horse> getFavoriteMares(@Param("rider") Rider rider);
}
```

#### Napomene:
- `Favorite f` — prolazimo kroz tabelu omiljenih konja
- `f.rider = :rider` — filtriramo po prosleđenom jahaču (`:rider` je imenovani parametar)
- `f.horse.gender = 'F'` — od tih konja uzimamo samo kobile
- `@Param("rider")` — vezuje argument metode za `:rider` u upitu

### 2. Dodaj metodu u `ErgelaService`

```java
public List<Horse> getAllFavoriteMaresForRider(Integer idRider) {
    Optional<Rider> riderOptional = riderRepository.findById(idRider);
    if (riderOptional.isEmpty()) return null;

    return horseRepository.getFavoriteMares(riderOptional.get());
}
```

### 3. Dodaj metodu u `ErgelaController`

```java
@GetMapping("/riders/{idRider}/horses")
public List<Horse> getAllFavoriteMaresForRider(@PathVariable Integer idRider) {
    return ergelaService.getAllFavoriteMaresForRider(idRider);
}
```

### 4. Testiraj u Talend API

```
GET http://localhost:8080/api/riders/1/horses
```

Očekivano: **status 200** + JSON niz kobila koje su omiljene jahaču 1.

## Zadatak 7
---
*`/riders/1` - Brisanje jahača 1*

### Objašnjenje zadatka
Jahač je povezan sa `Favorite` i `Session` zapisima (strani ključ `rider`). Ako pokušamo da obrišemo jahača direktno, baza puca zbog tih veza. Zato **prvo brišemo decu** (favorite i session-e tog jahača), pa **onda samog jahača**.

### 1. Dodaj derivisane metode u repozitorijume

`FavoriteRepository`:
```java
List<Favorite> findAllByRider(Rider rider);
```

`SessionRepository`:
```java
List<Session> findAllByRider(Rider rider);
```

#### Napomene:
- Ovo su **derivisani upiti** — Spring Data sam generiše SQL iz imena metode (`findAllBy` + polje `Rider`). Ne treba `@Query`.
- Ne zaboravi da dodaš i odgovarajuće import-e (`List`, `Rider`).

### 2. Dodaj metodu u `ErgelaService`

```java
public void deleteRider(Integer idRider) {
    Optional<Rider> riderOptional = riderRepository.findById(idRider);
    if (riderOptional.isEmpty()) return;

    List<Favorite> favorites = favoriteRepository.findAllByRider(riderOptional.get());
    favoriteRepository.deleteAll(favorites);

    List<Session> sessions = sessionRepository.findAllByRider(riderOptional.get());
    sessionRepository.deleteAll(sessions);

    riderRepository.delete(riderOptional.get());
}
```

#### Napomene:
- Redosled je bitan: prvo `Favorite` i `Session`, tek onda `Rider`
- Ako jahač ne postoji, samo izađemo (`return`) — nema šta da se briše

### 3. Dodaj metodu u `ErgelaController`

```java
@DeleteMapping("/riders/{idRider}")
public void deleteRider(@PathVariable Integer idRider) {
    ergelaService.deleteRider(idRider);
}
```

### 4. Testiraj u Talend API

```
DELETE http://localhost:8080/api/riders/1
```


## Zadatak 8
---
*`/horses` - Servis prihvata ceo objekat konja (uključujući `id` i objekat tipa `Breed`). Pronaći u bazi konja sa datim id-jem i ažurirati mu `fullName`. Ukoliko konj ne postoji (nema tog id-ja u bazi), sačuvati ga. Nijedna dva konja ne smeju imati isto ime (`fullName`).*

### Objašnjenje zadatka
1. Prvo proveriti da li već postoji konj sa istim `fullName` — ako postoji, ime je zauzeto → ne radimo ništa (vraćamo `Optional.empty()`)
2. Ako je ime slobodno, tražimo konja po `id`:
	- ne postoji → **sačuvamo** novog konja
	- postoji → **ažuriramo** mu `fullName` i sačuvamo

### 1. Dodaj metodu `findByFullName` u `HorseRepository`

```java
Optional<Horse> findByFullName(String fullName);
```

### 2. Dodaj metodu u `ErgelaService`

```java
public Optional<Horse> addOrChangeHorse(Horse horse) {
    Optional<Horse> horseWithSameNameOptional = horseRepository.findByFullName(horse.getFullName());
    if (horseWithSameNameOptional.isPresent()) return Optional.empty();

    Optional<Horse> horseOptional = horseRepository.findById(horse.getId());
    if (horseOptional.isEmpty()) {
        return Optional.of(horseRepository.save(horse));
    }

    Horse horseNewName = horseOptional.get();
    horseNewName.setFullName(horse.getFullName());
    return Optional.of(horseRepository.save(horseNewName));
}
```

#### Napomene:
- `Optional.empty()` → ime je zauzeto → kontroler vraća grešku
- `save()` radi i insert (novi konj) i update (postojeći, po `id`-ju)

### 3. Dodaj metodu u `ErgelaController`

```java
@PostMapping("/horses")
public ResponseEntity<Horse> addOrChangeHorse(@RequestBody Horse horse) {
    Optional<Horse> horseNew = ergelaService.addOrChangeHorse(horse);
    if (horseNew.isEmpty()) {
        return new ResponseEntity<>(HttpStatus.INTERNAL_SERVER_ERROR);
    }
    return new ResponseEntity<>(horseNew.get(), HttpStatus.CREATED);
}
```

### 4. Testiraj u Talend API

```
POST http://localhost:8080/api/horses
```

Request body:
```json
{
    "id": 6,
    "fullName": "Desert Night",
    "gender": "F",
    "dateOfBirth": "2024-06-06",
    "breed": {
        "id": 1,
        "coatColor": "bla",
        "name": "bla"
    }
}
```

#### Napomene:
- Ako u bazi već postoji konj sa `fullName` = `"Desert Night"`, dobijaš **500 (internal server error)** (ime zauzeto)
- Ako `id` = 6 ne postoji → novi konj se ubacuje; ako postoji → menja mu se samo `fullName`

## Zadatak 9
---
*`/sessions?idRider=1&idHorse=2` - Dobavljanje objekta Session za datog jahača i datog konja*

### Request parametri
Za razliku od `@PathVariable` (podatak je **deo putanje**), ovde koristimo `@RequestParam`
Podaci stižu kao **query parametri** iza `?` (`?idRider=1&idHorse=2`).

### 1. Dodaj derivisanu metodu u `SessionRepository`

```java
Optional<Session> findByRiderIdAndHorseId(Integer riderId, Integer horseId);
```

#### Napomene:
- Spring iz imena `findByRiderIdAndHorseId` sam sklopi upit: `Session` čiji `rider.id` i `horse.id` odgovaraju prosleđenim vrednostima

### 2. Dodaj metodu u `ErgelaService`

```java
public Optional<Session> getSessionForRiderAndHorse(Integer idRider, Integer idHorse) {
    return sessionRepository.findByRiderIdAndHorseId(idRider, idHorse);
}
```

### 3. Dodaj metodu u `ErgelaController`

```java
@GetMapping("/sessions")
public ResponseEntity<Session> getSessionForRiderAndHorse(@RequestParam Integer idRider, @RequestParam Integer idHorse) {
    Optional<Session> sessionOptional = ergelaService.getSessionForRiderAndHorse(idRider, idHorse);
    if (sessionOptional.isEmpty()) {
        return new ResponseEntity<>(HttpStatus.NOT_FOUND);
    }
    return new ResponseEntity<>(sessionOptional.get(), HttpStatus.OK);
}
```

### 4. Testiraj u Talend API

```
GET http://localhost:8080/api/sessions?idRider=1&idHorse=2
```

## Zadatak 10
---
*`/horses/search?idBreed=1` - Dobavljanje svih konja rase 1*

### 1. Dodaj derivisanu metodu u `HorseRepository`

```java
List<Horse> findAllByBreedId(Integer breedId);
```

#### Napomene:
- `findAllByBreedId` → svi konji čiji je `breed.id` jednak prosleđenom

### 2. Dodaj metodu u `ErgelaService`

```java
public List<Horse> getAllHorsesByBreed(Integer idBreed) {
    return horseRepository.findAllByBreedId(idBreed);
}
```

### 3. Dodaj metodu u `ErgelaController`

```java
@GetMapping("/horses/search")
public List<Horse> getAllHorsesByBreed(@RequestParam Integer idBreed) {
    return ergelaService.getAllHorsesByBreed(idBreed);
}
```

#### Napomene:
- Ovde je putanja `/horses/search` sa `@RequestParam`, a `/horses/{idHorse}` iz Zadatka 2 koristi `@PathVariable` — pošto je jedna sa query parametrom a druga sa promenljivom u putanji, Spring ih razlikuje i nema konflikta.

### 4. Testiraj u Talend API

```
GET http://localhost:8080/api/horses/search?idBreed=1
```

## Zadatak 11
---
*`/riders/John/Doe/breeds` - Dobavljanje rasa svih konja koji su omiljeni jahaču John Doe*

### Objašnjenje zadatka
1. Naći jahača po **imenu i prezimenu** (ne po id-ju)
2. Iz njegovih omiljenih konja izdvojiti **rase** (bez duplikata)

### 1. Kreiraj prazan `BreedRepository`

```java
package com.rzk.ergela_service.repository;

import com.rzk.ergela_service.model.Breed;
import org.springframework.data.jpa.repository.JpaRepository;

public interface BreedRepository extends JpaRepository<Breed, Integer> {

}
```

### 2. Dodaj derivisanu metodu u `RiderRepository`

```java
Optional<Rider> findByNameAndSurname(String name, String surname);
```

### 3. Dodaj JPQL upit u `BreedRepository`

```java
@Query("SELECT DISTINCT f.horse.breed FROM Favorite f WHERE f.rider = :rider")
List<Breed> getByRiderFavorites(@Param("rider") Rider rider);
```

(uz odgovarajuće import-e: `Query`, `Param`, `List`, `Rider`)

#### Napomene:
- `f.horse.breed` — kroz omiljenog konja dohvatamo njegovu rasu
- `DISTINCT` — ista rasa se ne ponavlja u rezultatu

### 4. Dodaj metodu u `ErgelaService`

```java
public List<Breed> getBreedsOfFavoriteHorsesForRider(String riderName, String riderSurname) {
    Optional<Rider> riderOptional = riderRepository.findByNameAndSurname(riderName, riderSurname);
    if (riderOptional.isEmpty()) return null;
    return breedRepository.getByRiderFavorites(riderOptional.get());
}
```

Ne zaboravi da injektuješ `BreedRepository` u servis (dodaj `final` polje, `@RequiredArgsConstructor` će ga ubaciti kroz konstruktor):
```java
private final BreedRepository breedRepository;
```

### 5. Dodaj metodu u `ErgelaController`

```java
@GetMapping("/riders/{riderName}/{riderSurname}/breeds")
public ResponseEntity<List<Breed>> getBreedsOfFavoriteHorsesForRider(@PathVariable String riderName, @PathVariable String riderSurname) {
    List<Breed> breedsOfFavoriteHorsesForRider = ergelaService.getBreedsOfFavoriteHorsesForRider(riderName, riderSurname);
    if (breedsOfFavoriteHorsesForRider == null) {
        return new ResponseEntity<>(HttpStatus.NOT_FOUND);
    }
    return new ResponseEntity<>(breedsOfFavoriteHorsesForRider, HttpStatus.OK);
}
```

### 6. Testiraj u Talend API

```
GET http://localhost:8080/api/riders/John/Doe/breeds
```

## Gotova aplikacija
---
Svih 11 endpoint-a je implementirano. Finalni `ErgelaService` i `ErgelaController` sadrže sve metode iz zadataka, a repozitorijumi sve potrebne upite.

### Repozitorijumi — pregled dodatih metoda

- `RiderRepository` → `findByNameAndSurname`
- `HorseRepository` → `findByFullName`, `findAllByBreedId`, `getFavoriteMares` (JPQL)
- `FavoriteRepository` → `findAllByRider`
- `SessionRepository` → `findAllByRider`, `findByRiderIdAndHorseId`
- `BreedRepository` → `getByRiderFavorites` (JPQL)

### Struktura projekta na kraju

```
ergela-services/
├── pom.xml
└── src/main/
    ├── java/com/rzk/ergela_services/
    │   ├── ErgelaServicesApplication.java
    │   ├── model/
    │   │   ├── Rider.java
    │   │   ├── Horse.java
    │   │   ├── Favorite.java
    │   │   ├── Session.java
    │   │   └── Breed.java
    │   ├── repository/
    │   │   ├── RiderRepository.java
    │   │   ├── HorseRepository.java
    │   │   ├── FavoriteRepository.java
    │   │   ├── SessionRepository.java
    │   │   └── BreedRepository.java
    │   ├── service/
    │   │   └── ErgelaService.java
    │   └── controller/
    │       └── ErgelaController.java
    └── resources/
        └── application.properties
```

### Rekapitulacija endpoint-a

- `GET    /api/riders` — svi jahači
- `GET    /api/horses/{idHorse}` — jedan konj
- `POST   /api/riders` — novi jahač (201)
- `POST   /api/riders/{idRider}/horse` — dodaj konja u omiljene (404 ako nema jahača/konja)
- `PATCH  /api/sessions/{idSession}/time/{time}` — ažuriraj vreme (400 ako `time` nije 2 karaktera)
- `GET    /api/riders/{idRider}/horses` — omiljene kobile jahača
- `DELETE /api/riders/{idRider}` — obriši jahača (i njegove favorite/session-e)
- `POST   /api/horses` — dodaj ili izmeni konja (jedinstven `fullName`)
- `GET    /api/sessions?idRider=..&idHorse=..` — trening za par jahač–konj
- `GET    /api/horses/search?idBreed=..` — konji po rasi
- `GET    /api/riders/{riderName}/{riderSurname}/breeds` — rase omiljenih konja jahača
