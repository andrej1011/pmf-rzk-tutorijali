## Exceptions i Validacija 


## Nadovezivanje na vežbe 1
---
**Vežbe 1** - Setup projekta i osnovni zadaci [KLIKNI OVDE](https://github.com/andrej1011/pmf-rzk-tutorijali/blob/main/rzk-t1.md)

Na drugim vežbama **ne pravimo nov projekat**. Radimo nad istom `ergela-services` aplikacijom iz vežbi 1. Dodajemo dve stvari:

1. **Upravljanje izuzecima** — umesto da servis vrati `null`, a kontroler ručno pravi `404`, servis će **baciti izuzetak**, a jedno centralno mesto (`@ControllerAdvice`) će ga uhvatiti i vratiti lep JSON odgovor sa porukom o grešci i statusnim kodom.
2. **Validacija** — proveru ulaza (`@PathVariable`, `@RequestParam`, `@RequestBody`) prebacujemo na anotacije (`@Min`, `@NotBlank`, `@Valid`...) umesto ručnih `if`-ova.

### ☝️🤓
Do sada su nam servisi vraćali `null` pa smo u kontroleru pisali `if (x == null) return 404`. To radi, ali se logika greške razvlači kroz ceo kontroler. Cilj vežbi 2 je da grešku obradimo **na jednom mestu**.

## Priprema — infrastruktura za izuzetke
---
Pre nego što krenemo na zadatke, napravimo tri stvari koje ćemo koristiti u svim servisima: **klasu odgovora o grešci**, **custom izuzetke** i **globalni handler**.

Napravi nov paket:
```
src/main/java/com/rzk/ergela_services/exception
```

### Korak 1 — Klasa `ErrorEntity`

Kad servis pukne, klijentu vraćamo objekat sa **porukom** i **vremenom** greške:

`exception/ErrorEntity.java`:
```java
package com.rzk.ergela_services.exception;

import lombok.AllArgsConstructor;
import lombok.Getter;

import java.time.LocalDateTime;

@Getter
@AllArgsConstructor
public class ErrorEntity {
    private String message;
    private LocalDateTime dateTime;
}
```

Šta rade anotacije:
- `@Getter` (Lombok) — getteri (Jackson-u trebaju da napravi JSON)
- `@AllArgsConstructor` (Lombok) — konstruktor sa svim poljima → `new ErrorEntity("poruka", LocalDateTime.now())`
- `message` — opis greške, `dateTime` — kada je nastala

### Korak 2 — Custom izuzeci

Pravimo tri izuzetka koja traži zadatak. Svi nasleđuju `RuntimeException` (nezahtevani izuzeci — ne moramo ih hvatati kroz `try/catch`).

`exception/EntityDoesNotExistException.java`:
```java
package com.rzk.ergela_services.exception;

import lombok.AllArgsConstructor;
import lombok.Getter;

@Getter
@AllArgsConstructor
public class EntityDoesNotExistException extends RuntimeException {
    private String message;
}
```

`exception/EntityAlreadyExistsException.java`:
```java
package com.rzk.ergela_services.exception;

import lombok.AllArgsConstructor;
import lombok.Getter;

@Getter
@AllArgsConstructor
public class EntityAlreadyExistsException extends RuntimeException {
    private String message;
}
```

`exception/TimeNotValidException.java`:
```java
package com.rzk.ergela_services.exception;

import lombok.AllArgsConstructor;
import lombok.Getter;

@Getter
@AllArgsConstructor
public class TimeNotValidException extends RuntimeException {
    private String message;
}
```

#### Napomene:
- `@AllArgsConstructor` pravi konstruktor `new EntityDoesNotExistException("poruka")`, a `@Getter` daje `getMessage()` koji handler kasnije čita
- Sva tri izuzetka su identična po strukturi — razlikuje ih samo **tip**, po kom handler bira statusni kod (404 / 409 / 400)

### Korak 3 — Globalni handler (`@ControllerAdvice`)

Ovo je srce priče: klasa koja **hvata izuzetke iz svih kontrolera** i pretvara ih u `ErrorEntity`.

`exception/CustomResponseEntityExceptionHandler.java`:
```java
package com.rzk.ergela_services.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.servlet.mvc.method.annotation.ResponseEntityExceptionHandler;

import java.time.LocalDateTime;

@ControllerAdvice
public class CustomResponseEntityExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(EntityDoesNotExistException.class)
    public final ResponseEntity<ErrorEntity> handleEntityDoesNotExist(EntityDoesNotExistException ex) {
        ErrorEntity errorEntity = new ErrorEntity(ex.getMessage(), LocalDateTime.now());
        return new ResponseEntity<>(errorEntity, HttpStatus.NOT_FOUND);
    }

    @ExceptionHandler(EntityAlreadyExistsException.class)
    public final ResponseEntity<ErrorEntity> handleEntityAlreadyExists(EntityAlreadyExistsException ex) {
        ErrorEntity errorEntity = new ErrorEntity(ex.getMessage(), LocalDateTime.now());
        return new ResponseEntity<>(errorEntity, HttpStatus.CONFLICT);
    }

    @ExceptionHandler(TimeNotValidException.class)
    public ResponseEntity<ErrorEntity> handleTimeNotValid(TimeNotValidException ex) {
        ErrorEntity errorEntity = new ErrorEntity(ex.getMessage(), LocalDateTime.now());
        return new ResponseEntity<>(errorEntity, HttpStatus.BAD_REQUEST);
    }

    @ExceptionHandler(Exception.class)
    public final ResponseEntity<ErrorEntity> handleAllExceptions(Exception ex) {
        ErrorEntity errorEntity = new ErrorEntity(ex.getMessage(), LocalDateTime.now());
        return new ResponseEntity<>(errorEntity, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

Šta rade anotacije:
- `@ControllerAdvice` — klasa važi za **sve** kontrolere; presreće izuzetke pre nego što stignu do klijenta
- `extends ResponseEntityExceptionHandler` — Spring-ova bazna klasa; ona već zna da greške validacije `@PathVariable`/`@RequestParam` vrati kao **400**. Zato je nasleđujemo (a u Zadatku 2 ćemo joj i override-ovati jednu metodu).
- `@ExceptionHandler(X.class)` — „kad negde poleti izuzetak `X`, pozovi ovu metodu"

#### Šta radi `CustomResponseEntityExceptionHandler`:

Umesto da Spring vrati sirovu i nečitljivu grešku, ova metoda:
1. **Uhvati** sve greške u validaciji podataka.
2. **Agregira** ih u jednu preglednu poruku (koje polje nije u redu i zašto).
3. **Vrati** klijentu čist `400 Bad Request` odgovor, tako da frontend ili drugi servis odmah zna šta treba da ispravi.

Umesto HTML greške, klijent dobija JSON:
```json
{
  "message": "Total errors: 2, Field: username - must not be null, Field: email - is not a valid email address, ",
  "timestamp": "2026-07-10T10:07:12"
}
```

## Zadatak 1 – Exceptions
---
_Implementirati upravljanje izuzecima za servise ispod. Odgovor servera je objekat sa porukom o grešci i odgovarajućim statusnim kodom._

### ☝️🤓 Ideja

Do sada: servis vrati `null` → kontroler proverava i pravi 404. Sad:

- **servis** kad nešto ne nađe → `orElseThrow(() -> new EntityDoesNotExistException("..."))`
- **kontroler** samo pozove servis i vrati rezultat — bez ijednog `if`-a za grešku
- izuzetak hvata `CustomResponseEntityExceptionHandler` iz pripreme i vraća `ErrorEntity` + status

`orElseThrow` radi nad `Optional`-om: ako je prazan → baci izuzetak, inače → vrati sadržaj.

### 1.1 
*`/horses/1` — konj ne postoji → `EntityDoesNotExistException` (404)*

#### `ErgelaService`
**Servis** više ne vraća `Optional<Horse>` nego `Horse`, a ako ga nema, baca exception:
```java
public Horse getHorse(Integer idHorse) {
    return horseRepository.findById(idHorse)
            .orElseThrow(() -> new EntityDoesNotExistException("Horse with id: " + idHorse + " not found."));
}
```

#### `ErgelaController`
**Kontroler** — nema više `if`, samo vrati konja:
```java
@GetMapping("/horses/{idHorse}")
public Horse getHorse(@PathVariable Integer idHorse) {
    return ergelaService.getHorse(idHorse);
}
```


- Kad konj postoji → vrati se `Horse` (status **200**)
- Kad ne postoji → `EntityDoesNotExistException` → handler → **404** + `ErrorEntity`

**Testiraj u Talend API:**
```
GET http://localhost:8080/api/horses/999
```

Očekivano: **status 404** + telo:
```json
{
    "message": "Horse with id: 999 not found.",
    "dateTime": "2026-01-10T12:34:56.789"
}
```

#### 💡 
Ukoliko server vraća Error 500 (Internal Error) umesto 404 Not found — proverite da li je `EntityDoesNotExistException` importovan pravilno u **Servisu**. 

Treba da bude importovan kao `import com.rzk.ergela_service.exception.EntityDoesNotExistException;`


### 1.2 
*`/riders/1/horse` — 404 ako nema jahača/konja, 409 ako je već omiljen*

**Prvo dodaj metodu u `FavoriteRepository`** (derivisani upit — proveravamo da li taj par jahač–konj već postoji):

```java
Optional<Favorite> findByRiderAndHorse(Rider rider, Horse horse);
```

#### `ErgelaService`

Izmeni metodu `addHorseToFavorite` da se ne oslanja na interne provere koje smo ranije napravili nego da koristi exceptione , slično kao u prethodnom zadatku.

```java
public Favorite addHorseToRider(Horse horse, Integer idRider){  
    Rider rider = riderRepository.findById(idRider)  
            .orElseThrow(()-> new EntityDoesNotExistException("Rider with id:"+idRider+" not found."));  
    String horseName = horse.getFullName();  
    Horse realHorse = horseRepository.findDistinctByFullName(horseName)  
            .orElseThrow(()-> new EntityDoesNotExistException("Horse with name:"+horseName+" not found."));  
  
    Optional<Favorite> optionalFavorite = favoriteRepository.findByRiderAndHorse(rider,realHorse);  
    if(optionalFavorite.isPresent())  
        throw new EntityAlreadyExistsException("Horse with name: " + horseName + " is already favorite for rider with id: " +idRider+ ".");  
  
    Favorite favorite = new Favorite();  
    favorite.setHorse(realHorse);  
    favorite.setRider(rider);  
    return favoriteRepository.save(favorite);  
}
```


#### `ErgelaController`
```java
@PostMapping("/riders/{idRider}/horse")
public Favorite addHorseToFavorite(@PathVariable Integer idRider, @RequestBody Horse horse) {
    return ergelaService.addHorseToFavorite(idRider, horse);
}
```

#### Testiraj (konj koji je već omiljen jahaču):

```
POST http://localhost:8080/api/riders/1/horse
```

Body:
```json
{ "fullName": "Desert Storm" }
```

Očekivano: **status 409** + poruka da je konj već omiljen tom jahaču.

### 1.3
*`/sessions/1/time/14` — vreme mora biti između 00 i 23 → `TimeNotValidException` (400)*

#### `ErgelaService`
**Servis** — prvo traži session (ako ga nema → 404), pa tek onda proverava vreme:
```java
public Session updateSessionTime(Integer idSession, String time) {
    Session session = sessionRepository.findById(idSession)
            .orElseThrow(() -> new EntityDoesNotExistException("Session with id: " + idSession + " not found."));

    if (time.compareTo("00") < 0 || time.compareTo("23") > 0) {
        throw new TimeNotValidException("The time should be between values 00 and 23. Provided time is: " + time + ".");
    }
    session.setTime(time);
    return sessionRepository.save(session);
}
```

#### `ErgelaController`
```java
@PatchMapping("/sessions/{idSession}/time/{time}")
public Session updateSessionTime(@PathVariable Integer idSession, @PathVariable String time) {
    return ergelaService.updateSessionTime(idSession, time);
}
```

#### Napomene:

- `time.compareTo("00") < 0 || time.compareTo("23") > 0` — poređenje stringova radi jer su oba dužine 2 i sastoje se od cifara, pa je leksikografski poredak isti kao brojčani (`"14"` je između `"00"` i `"23"`)
- Redosled je bitan: **prvo session, pa vreme** → ako session ne postoji, dobijaš **404** čak i za nevalidno vreme
- Proveru da `time` ima **tačno 2 karaktera** dodajemo u Zadatku 2 (`@Size` na path varijabli)

#### **Testiraj (nevalidno vreme):**

```
PATCH http://localhost:8080/api/sessions/1/time/45
```

Očekivano: **status 400** + poruka da vreme mora biti između 00 i 23.

### 1.4 
*`/riders/1/horses` — jahač ne postoji → 404*

#### `ErgelaService`
```java
public List<Horse> getAllFavoriteMaresForRider(Integer idRider) {
    Rider rider = riderRepository.findById(idRider)
            .orElseThrow(() -> new EntityDoesNotExistException("Rider with id: " + idRider + " not found."));

    return horseRepository.getFavoriteMares(rider);
}
```

#### `ErgelaController`
```java
@GetMapping("/riders/{idRider}/horses")
public List<Horse> getAllFavoriteMaresForRider(@PathVariable Integer idRider) {
    return ergelaService.getAllFavoriteMaresForRider(idRider);
}
```

#### **Testiraj:**
```
GET http://localhost:8080/api/riders/999/horses
```
Očekivano: **status 404**.

### 1.5 
*`/riders/1` — brisanje, jahač ne postoji → 404*

#### `ErgelaService`
```java
public void deleteRider(Integer idRider) {
    Rider rider = riderRepository.findById(idRider)
            .orElseThrow(() -> new EntityDoesNotExistException("Rider with id: " + idRider + " not found."));

    List<Favorite> favorites = favoriteRepository.findAllByRider(rider);
    favoriteRepository.deleteAll(favorites);

    List<Session> sessions = sessionRepository.findAllByRider(rider);
    sessionRepository.deleteAll(sessions);

    riderRepository.delete(rider);
}
```

#### `ErgelaController`
```java
@DeleteMapping("/riders/{idRider}")
public void deleteRider(@PathVariable Integer idRider) {
    ergelaService.deleteRider(idRider);
}
```

#### **Testiraj:**
```
DELETE http://localhost:8080/api/riders/999
```

Očekivano: **status 404**.

### 1.6 
*`/riders/John/Doe/breeds` — jahač ne postoji → 404*

#### `ErgelaService`
```java
public List<Breed> getBreedsOfFavoriteHorsesForRider(String riderName, String riderSurname) {
    Rider rider = riderRepository.findByNameAndSurname(riderName, riderSurname)
            .orElseThrow(() -> new EntityDoesNotExistException("Rider with name: " + riderName + " " + riderSurname + " not found."));

    return breedRepository.getByRiderFavorites(rider);
}
```

#### `ErgelaController`
```java
@GetMapping("/riders/{riderName}/{riderSurname}/breeds")
public List<Breed> getBreedsOfFavoriteHorsesForRider(@PathVariable String riderName, @PathVariable String riderSurname) {
    return ergelaService.getBreedsOfFavoriteHorsesForRider(riderName, riderSurname);
}
```

#### **Testiraj:**
```
GET http://localhost:8080/api/riders/Ne/Postoji/breeds
```
Očekivano: **status 404**.

---

### TLDR (Zadatak 1)

1. Servis pozove `orElseThrow(...)` → baci `EntityDoesNotExistException`
2. Izuzetak proleti kroz kontroler (ne hvatamo ga tamo)
3. Spring vidi `@ControllerAdvice` i u njemu `@ExceptionHandler` baš za taj tip
4. Handler napravi `ErrorEntity(poruka, vreme)` i status (404 / 409 / 400)
5. **Jackson** serijalizuje `ErrorEntity` u JSON
6. Klijent dobija status greške + čisto telo

## Zadatak 2 — Validacija

---

_Implementirati validaciju ulaza za servise ispod. Nastavlja se na Zadatak 1 (isti projekat, isti `CustomResponseEntityExceptionHandler`)._

#### ☝️🤓 Dve vrste validacije

- **`@PathVariable` / `@RequestParam`** (npr. `@Min(1)`, `@Size`, `@NotBlank` na parametru) — da bi radila, kontroler klasa mora imati `@Validated`. Kad pukne, Spring (kroz baznu `ResponseEntityExceptionHandler`) vrati **400** automatski.
- **`@RequestBody` objekat** (npr. `Rider`, `Horse`) — anotacije se stavljaju na **polja entiteta**, a ispred parametra ide `@Valid`. Kad pukne, baca se `MethodArgumentNotValidException` → hvatamo je **override-om** metode `handleMethodArgumentNotValid(...)`.

### Priprema
---
#### Korak 1 — Dependency

Dodaj u `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

Posle dodavanja → desni klik na `pom.xml` → **Maven → Sync Project** (povuče nov dependency). (Ako ne vidiš izmene, klikni i **Generate Sources and Update Folders**.)

#### Korak 2 — Uključi validaciju path varijabli (`@Validated`)

Iznad `ErgelaController` dodaj `@Validated`:

```java
@RestController
@RequiredArgsConstructor
@Validated
@RequestMapping("/api")
public class ErgelaController {
    ...
}
```

Import: `org.springframework.validation.annotation.Validated`.

Napomena:
- Bez `@Validated` anotacije `@Min`, `@Size`, `@NotBlank` na `@PathVariable` se **ignorišu**

#### Korak 3 — Override `handleMethodArgumentNotValid` (za `@Valid` tela)

Kad `@Valid` objekat u telu ne prođe validaciju, Spring baca `MethodArgumentNotValidException`. 

Dodaj **u istu klasu** `CustomResponseEntityExceptionHandler` ovaj override:
```java
@Override
protected ResponseEntity<Object> handleMethodArgumentNotValid(MethodArgumentNotValidException ex, HttpHeaders headers, HttpStatusCode status, WebRequest request) {
    StringBuilder stringBuilder = new StringBuilder();
    stringBuilder.append("Total errors: ").append(ex.getErrorCount()).append(", ");
    for (FieldError e : ex.getFieldErrors()) {
        stringBuilder.append("Field: ").append(e.getField()).append(" - ").append(e.getDefaultMessage()).append(", ");
    }

    ErrorEntity errorEntity = new ErrorEntity(stringBuilder.toString(), LocalDateTime.now());
    return new ResponseEntity<>(errorEntity, HttpStatus.BAD_REQUEST);
}
```


---

### 2.1 
*`/horses/1` — `id` ne sme biti manji od 1*

#### `ErgelaController`:
```java
@GetMapping("/horses/{idHorse}")
public Horse getHorse(@PathVariable @Min(1) Integer idHorse) {
    return ergelaService.getHorse(idHorse);
}
```

Import: `jakarta.validation.constraints.Min`.

#### Napomene:
- `@Min(1)` na path varijabli → radi zbog `@Validated` na klasi
- Nevalidan `id` → **400** (bazna klasa), pre nego što se uopšte pozove servis

#### Testiraj:
```
GET http://localhost:8080/api/horses/0
```

Očekivano: **status 400**.

### 2.2 
*`/riders` - Dodavanje novog jahača. Validirati da se moraju uneti ona polja koja su obavezna u bazi. Override-ovati metodu handleMethodArgumentNotValid(...)*

U bazi su `name` i `surname` `NOT NULL`, pa dodajemo validaciju u modelu`Rider`:
#### Model `Rider`
```java
@NotBlank
@Column(name = "name", nullable = false, length = 45)
private String name;

@NotBlank
@Column(name = "surname", nullable = false, length = 45)
private String surname;
```
Import: `jakarta.validation.constraints.NotBlank`.

#### `ErgelaController`
`@Valid` ispred tela zahteva:
```java
@PostMapping("/riders")
public ResponseEntity<Rider> addRider(@RequestBody @Valid Rider rider) {
    return new ResponseEntity<>(ergelaService.addRider(rider), HttpStatus.CREATED);
}
```

Import: `jakarta.validation.Valid`.

#### Testiraj (bez imena i prezimena):

```
POST http://localhost:8080/api/riders
```

Body:
```json
{ "address": "123 Green St" }
```

Očekivano: **status 400** + poruka tipa `Total errors: 2, Field: name - must not be blank, Field: surname - must not be blank,`

### 2.3 
*`/riders/1/horse` — `@Min(1)` na jahaču + `@Valid` na konju (+ gender i dateOfBirth)*

U entitetu `Horse` anotiraj obavezna polja iz baze i dodatna pravila:
```java
@NotBlank(message = "The full name is a unique identifier for each horse and must therefore be specified.")
@Column(name = "fullName", nullable = false, length = 45)
private String fullName;

@Column(name = "nickname", length = 45)
private String nickname;

@NotBlank
@Size(min = 1, max = 1)
@Pattern(regexp = "[MF]")
@Column(name = "gender", nullable = false, length = 1)
private String gender;

@NotNull
@Past
@Column(name = "dateOfBirth", nullable = false)
private LocalDate dateOfBirth;

@ManyToOne(optional = false)
@JoinColumn(name = "breed", nullable = false)
private Breed breed;
```

Import-i: `jakarta.validation.constraints.*` (`NotBlank`, `NotNull`, `Size`, `Pattern`, `Past`).

#### `ErgelaController`
```java
@PostMapping("/riders/{idRider}/horse")
public Favorite addHorseToRider(@PathVariable @Min(1) Integer idRider, @RequestBody @Valid Horse horse) {
    return ergelaService.addHorseToRider(idRider, horse);
}
```


Šta rade anotacije na `Horse`:

- `fullName` — `@NotBlank` sa svojom porukom (jer je jedinstveni identifikator)
- `gender` — `@NotBlank` + `@Size(1,1)` (tačno 1 karakter) + `@Pattern("[MF]")` (samo `M` ili `F`)
- `dateOfBirth` — `@NotNull` (mora postojati) + `@Past` (mora biti u prošlosti)
- `breed` — `@ManyToOne(optional = false)` obezbeđuje da rasa mora postojati

Napomene:

- Pošto sad stoji `@Valid`, u telu se šalje **ceo validan konj** (fullName + gender + dateOfBirth + breed) — minimalno telo `{"fullName": "..."}` iz Zadatka 1 sad ne prolazi validaciju i vraća **400**

#### Testiraj (loš gender):
```
POST http://localhost:8080/api/riders/1/horse
```

Body:
```json
{
    "fullName": "Desert Storm",
    "gender": "X",
    "dateOfBirth": "2020-05-01",
    "breed": { "id": 1, "name": "Arabian", "coatColor": "bay" }
}
```

Očekivano: **status 400** (gender ne odgovara `[MF]`).

### 2.4 
*`/sessions/1/time/14` — `id` ≥ 1 i `time` tačno 2 karaktera*

#### `ErgelaController`
```java
@PatchMapping("/sessions/{sessionId}/time/{time}")  
@ResponseStatus(HttpStatus.OK)  
public Session changeSessionTime(@PathVariable @Min(1) Integer sessionId, @PathVariable @Size(min = 2, max = 2) String time){  
   return  ergelaService.changeSessionTime(sessionId,time);  
}
```

Import: `jakarta.validation.constraints.Size`.

 Napomene:

- `@Size(min = 2, max = 2)` proverava **dužinu** stringa (2 karaktera)
- Proveru opsega `00`–`23` i dalje radi `TimeNotValidException` u servisu (to je poslovno pravilo, ne dužina) — vidi Zadatak 1, 1.3
- Podela: dužinu hvata validacija (400 iz bazne klase), opseg hvata servis (400 iz `TimeNotValidException`)

#### Testiraj:

```
PATCH http://localhost:8080/api/sessions/1/time/5
```

Očekivano: **status 400** (`time` nema 2 karaktera).

### 2.5 
*`/riders/1/horses` — `id` jahača ≥ 1*

#### `ErgelaController`
```java
@GetMapping("/riders/{idRider}/horses")  
@ResponseStatus(HttpStatus.OK)  
public List<Horse> getAllFavoriteMaresForRider(@PathVariable @Min(1) Integer idRider) {  
    return ergelaService.getAllFavoriteMaresForRider(idRider);  
}
```

#### Testiraj:
```
GET http://localhost:8080/api/riders/0/horses
```

Očekivano: **status 400**.

### 2.6 `/riders/1` — `id` jahača ≥ 1 (brisanje)

```java
@DeleteMapping("/riders/{idRider}")
public void deleteRider(@PathVariable @Min(1) Integer idRider) {
    ergelaService.deleteRider(idRider);
}
```

**Testiraj:**

```
DELETE http://localhost:8080/api/riders/0
```

Očekivano: **status 400**.

### 2.7 `/riders/John/Doe/breeds` — ime ≥ 2 karaktera, prezime nije prazno

```java
@GetMapping("/riders/{riderName}/{riderSurname}/breeds")
public List<Breed> getBreedsOfFavoriteHorsesForRider(@PathVariable @Size(min = 2) String riderName, @PathVariable @NotBlank String riderSurname) {
    return ergelaService.getBreedsOfFavoriteHorsesForRider(riderName, riderSurname);
}
```

#### Napomene:

- `@Size(min = 2)` — ime bar 2 karaktera (max nije zadat)
- `@NotBlank` — prezime ne sme biti prazno ni samo razmaci

**Testiraj:**

```
GET http://localhost:8080/api/riders/J/Doe/breeds
```

Očekivano: **status 400** (ime kraće od 2 karaktera).

## ☝️🤓
- Ukoliko server vraća Error 500 umesto Error 400, probajte da dodate ovu metodu u `CustomResponseEntityExceptionHandler`
```java
@ExceptionHandler(jakarta.validation.ConstraintViolationException.class)  
public final ResponseEntity<ErrorEntity> handleConstraintViolation(jakarta.validation.ConstraintViolationException ex, WebRequest request) {  
    ErrorEntity errorEntity = new ErrorEntity(ex.getMessage(), LocalDateTime.now());  
    return new ResponseEntity<>(errorEntity, HttpStatus.BAD_REQUEST);  
}
```


## Finalni `ErgelaController` (sa validacijom)

---

```java
package com.rzk.ergela_services.controller;

import com.rzk.ergela_services.model.*;
import com.rzk.ergela_services.service.ErgelaService;
import jakarta.validation.Valid;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequiredArgsConstructor
@Validated
@RequestMapping("/api")
public class ErgelaController {

    private final ErgelaService ergelaService;

    @GetMapping("/horses/{idHorse}")
    public Horse getHorse(@PathVariable @Min(1) Integer idHorse) {
        return ergelaService.getHorse(idHorse);
    }

    @PostMapping("/riders")
    public ResponseEntity<Rider> addRider(@RequestBody @Valid Rider rider) {
        return new ResponseEntity<>(ergelaService.addRider(rider), HttpStatus.CREATED);
    }

    @PostMapping("/riders/{idRider}/horse")
    public Favorite addHorseToFavorite(@PathVariable @Min(1) Integer idRider, @RequestBody @Valid Horse horse) {
        return ergelaService.addHorseToFavorite(idRider, horse);
    }

    @PatchMapping("/sessions/{idSession}/time/{time}")
    public Session updateSessionTime(@PathVariable @Min(1) Integer idSession, @PathVariable @Size(min = 2, max = 2) String time) {
        return ergelaService.updateSessionTime(idSession, time);
    }

    @GetMapping("/riders/{idRider}/horses")
    public List<Horse> getAllFavoriteMaresForRider(@PathVariable @Min(1) Integer idRider) {
        return ergelaService.getAllFavoriteMaresForRider(idRider);
    }

    @DeleteMapping("/riders/{idRider}")
    public void deleteRider(@PathVariable @Min(1) Integer idRider) {
        ergelaService.deleteRider(idRider);
    }

    @GetMapping("/riders/{riderName}/{riderSurname}/breeds")
    public List<Breed> getBreedsOfFavoriteHorsesForRider(@PathVariable @Size(min = 2) String riderName, @PathVariable @NotBlank String riderSurname) {
        return ergelaService.getBreedsOfFavoriteHorsesForRider(riderName, riderSurname);
    }
}
```


---
### Gde ide koja validacija

- **Entitet `Rider`** → `@NotBlank` na `name`, `surname`
- **Entitet `Horse`** → `fullName` `@NotBlank`; `gender` `@NotBlank @Size(1,1) @Pattern("[MF]")`; `dateOfBirth` `@NotNull @Past`
- **Kontroler (path varijable)** → `@Min(1)` na svim id-jevima, `@Size(2,2)` na `time`, `@Size(min=2)` na `riderName`, `@NotBlank` na `riderSurname`
- **Kontroler (klasa)** → `@Validated`
- **Kontroler (tela)** → `@Valid` ispred `Rider` i `Horse`
- **Handler** → override `handleMethodArgumentNotValid` za greške tela

### Ko šta baca i koji je status

- path/param validacija (`@Min`, `@Size`, `@NotBlank` na parametru) → **400** (bazna `ResponseEntityExceptionHandler`)
- `@Valid` telo (`Rider`, `Horse`) → `MethodArgumentNotValidException` → **400** (naš override)
- biznis logika iz Zadatka 1 (`EntityDoesNotExist` 404, `EntityAlreadyExists` 409, `TimeNotValid` 400) i dalje važe
