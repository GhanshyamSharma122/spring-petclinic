# 🐾 Spring Boot PetClinic — Build It From Scratch (Beginner Tutorial)

> A step-by-step guide to build a fully working **Pet Clinic Management System** using Spring Boot.
> By the end, you'll have owners, pets, vets, visits — all wired up with a database and beautiful HTML pages.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Generate the Project (Spring Initializr)](#2-generate-the-project-spring-initializr)
3. [Understand the Project Structure](#3-understand-the-project-structure)
4. [Core Concept — How Spring Boot Works](#4-core-concept--how-spring-boot-works)
5. [Step 1 — Create the Base Model Classes](#step-1--create-the-base-model-classes)
6. [Step 2 — Build the Owner Module](#step-2--build-the-owner-module)
7. [Step 3 — Build the Pet Module](#step-3--build-the-pet-module)
8. [Step 4 — Build the Visit Module](#step-4--build-the-visit-module)
9. [Step 5 — Build the Vet Module](#step-5--build-the-vet-module)
10. [Step 6 — Set Up the Database](#step-6--set-up-the-database)
11. [Step 7 — Build the Thymeleaf Templates](#step-7--build-the-thymeleaf-templates)
12. [Step 8 — System Configuration (Caching, i18n, Error Handling)](#step-8--system-configuration)
13. [Step 9 — Application Properties](#step-9--application-properties)
14. [Step 10 — Run and Test](#step-10--run-and-test)
15. [Concepts You Learned](#concepts-you-learned)

---

## 1. Prerequisites

Before starting, make sure you have:

- **Java 17+** installed → verify with `java -version`
- **Maven 3.6+** (or use the Maven wrapper `mvnw` that comes with the project)
- An **IDE** — IntelliJ IDEA (recommended), VS Code with Java extensions, or Eclipse
- A **web browser** to see your app running
- Basic knowledge of Java (classes, interfaces, inheritance)

---

## 2. Generate the Project (Spring Initializr)

Go to **[https://start.spring.io](https://start.spring.io)** and configure:

| Field | Value |
|-------|-------|
| **Project** | Maven |
| **Language** | Java |
| **Spring Boot** | 3.2.x (latest stable) |
| **Group** | `org.springframework.samples` |
| **Artifact** | `spring-petclinic` |
| **Name** | `petclinic` |
| **Packaging** | Jar |
| **Java** | 17 |

### Add these Dependencies:

- **Spring Web** — for building web pages and REST endpoints
- **Spring Data JPA** — for database access using Java objects
- **Thymeleaf** — HTML template engine (server-side rendering)
- **Validation** — for form validation (`@NotBlank`, `@Pattern`, etc.)
- **H2 Database** — an in-memory database (zero setup needed!)
- **Spring Boot DevTools** — auto-restart on code changes
- **Spring Boot Actuator** — health checks and monitoring endpoints
- **Spring Cache** — caching support

Click **Generate**, download the ZIP, extract it, and open in your IDE.

> **💡 What just happened?** Spring Initializr created a ready-to-run Maven project with all dependencies configured in `pom.xml`. You don't need to manually hunt for library versions — Spring Boot's "starter parent" manages that for you.

---

## 3. Understand the Project Structure

After extraction, your project looks like this:

```
spring-petclinic/
├── pom.xml                          ← Maven config + dependencies
├── mvnw / mvnw.cmd                  ← Maven wrapper (run Maven without installing it)
├── src/
│   ├── main/
│   │   ├── java/.../petclinic/      ← Your Java source code
│   │   │   ├── PetClinicApplication.java  ← THE entry point
│   │   │   ├── model/               ← Base entity classes
│   │   │   ├── owner/               ← Owner, Pet, Visit logic
│   │   │   ├── vet/                 ← Vet logic
│   │   │   └── system/              ← Config, error handling
│   │   └── resources/
│   │       ├── application.properties  ← App configuration
│   │       ├── templates/              ← Thymeleaf HTML pages
│   │       ├── static/                 ← CSS, JS, images
│   │       ├── db/h2/                  ← H2 database SQL scripts
│   │       └── messages/               ← i18n translation files
│   └── test/                        ← Unit & integration tests
```

---

## 4. Core Concept — How Spring Boot Works

Before coding, understand these **5 key ideas**:

### 4.1 `@SpringBootApplication`
This single annotation on your main class does 3 things:
- `@Configuration` — marks it as a config source
- `@EnableAutoConfiguration` — Spring magically configures things based on your dependencies
- `@ComponentScan` — finds all your classes annotated with `@Controller`, `@Service`, `@Repository`, etc.

```java
@SpringBootApplication
public class PetClinicApplication {
    public static void main(String[] args) {
        SpringApplication.run(PetClinicApplication.class, args);
    }
}
```
> This is the ONLY class you need to start. Spring Boot does the rest.

### 4.2 MVC Pattern (Model-View-Controller)
- **Model** → Java classes (`Owner.java`, `Pet.java`) mapped to database tables
- **View** → Thymeleaf HTML templates (what the user sees)
- **Controller** → Java classes that handle web requests and connect Model ↔ View

### 4.3 Dependency Injection
Instead of creating objects yourself (`new OwnerRepository()`), Spring creates them and "injects" them where needed. Just declare a constructor parameter.

### 4.4 JPA (Java Persistence API)
Annotate your Java classes with `@Entity`, and Spring Data automatically creates database tables and CRUD operations. No SQL needed for basic operations!

### 4.5 Spring Data Repositories
Write an interface that extends `JpaRepository`, and Spring auto-generates `save()`, `findById()`, `findAll()`, `delete()` methods. You can also create custom queries just by naming methods!

---

## Step 1 — Create the Base Model Classes

These are abstract parent classes that other entities will extend. This avoids code duplication.

### 1a. `BaseEntity.java` — Every entity needs an ID

**File:** `src/main/java/org/springframework/samples/petclinic/model/BaseEntity.java`

```java
package org.springframework.samples.petclinic.model;

import java.io.Serializable;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.MappedSuperclass;

@MappedSuperclass  // tells JPA: don't create a table for this, but include its fields in child tables
public class BaseEntity implements Serializable {

    @Id  // this field is the primary key
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // auto-increment
    private Integer id;

    public Integer getId() { return id; }
    public void setId(Integer id) { this.id = id; }

    // helper: returns true if the entity hasn't been saved to DB yet
    public boolean isNew() { return this.id == null; }
}
```

> **🧠 Concept: `@MappedSuperclass`** — This means "I'm a parent class for JPA entities, but don't create a database table for me." Child classes will inherit my fields into THEIR tables.

### 1b. `NamedEntity.java` — For things that have a name (PetType, Specialty)

**File:** `src/main/java/org/springframework/samples/petclinic/model/NamedEntity.java`

```java
package org.springframework.samples.petclinic.model;

import jakarta.persistence.Column;
import jakarta.persistence.MappedSuperclass;
import jakarta.validation.constraints.NotBlank;

@MappedSuperclass
public class NamedEntity extends BaseEntity {

    @Column
    @NotBlank  // validation: name cannot be empty
    private String name;

    public String getName() { return this.name; }
    public void setName(String name) { this.name = name; }

    @Override
    public String toString() {
        return (this.name != null) ? this.name : "<null>";
    }
}
```

### 1c. `Person.java` — For things that have first/last name (Owner, Vet)

**File:** `src/main/java/org/springframework/samples/petclinic/model/Person.java`

```java
package org.springframework.samples.petclinic.model;

import jakarta.persistence.Column;
import jakarta.persistence.MappedSuperclass;
import jakarta.validation.constraints.NotBlank;

@MappedSuperclass
public class Person extends BaseEntity {

    @Column
    @NotBlank
    private String firstName;

    @Column
    @NotBlank
    private String lastName;

    // getters and setters for firstName and lastName
    public String getFirstName() { return this.firstName; }
    public void setFirstName(String firstName) { this.firstName = firstName; }
    public String getLastName() { return this.lastName; }
    public void setLastName(String lastName) { this.lastName = lastName; }
}
```

> **🧠 Inheritance Hierarchy:**
> ```
> BaseEntity (id)
> ├── NamedEntity (name)     → PetType, Specialty
> └── Person (firstName, lastName) → Owner, Vet
> ```

---

## Step 2 — Build the Owner Module

### 2a. `Owner.java` — The Entity

**File:** `src/main/java/org/springframework/samples/petclinic/owner/Owner.java`

```java
package org.springframework.samples.petclinic.owner;

import java.util.ArrayList;
import java.util.List;
import org.springframework.samples.petclinic.model.Person;
import jakarta.persistence.*;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Pattern;

@Entity                    // tells JPA: this IS a database table
@Table(name = "owners")    // table name in the DB
public class Owner extends Person {

    @Column
    @NotBlank
    private String address;

    @Column
    @NotBlank
    private String city;

    @Column
    @NotBlank
    @Pattern(regexp = "\\d{10}", message = "{telephone.invalid}")  // must be exactly 10 digits
    private String telephone;

    // One owner has MANY pets. CascadeType.ALL means saving an owner also saves their pets.
    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.EAGER)
    @JoinColumn(name = "owner_id")  // the foreign key column in the pets table
    @OrderBy("name")                // pets are sorted alphabetically
    private final List<Pet> pets = new ArrayList<>();

    // Standard getters/setters for address, city, telephone
    public String getAddress() { return this.address; }
    public void setAddress(String address) { this.address = address; }
    public String getCity() { return this.city; }
    public void setCity(String city) { this.city = city; }
    public String getTelephone() { return this.telephone; }
    public void setTelephone(String telephone) { this.telephone = telephone; }
    public List<Pet> getPets() { return this.pets; }

    public void addPet(Pet pet) {
        if (pet.isNew()) { getPets().add(pet); }
    }

    // Find a pet by name or by ID (used when editing)
    public Pet getPet(String name, boolean ignoreNew) {
        for (Pet pet : getPets()) {
            String compName = pet.getName();
            if (compName != null && compName.equalsIgnoreCase(name)) {
                if (!ignoreNew || !pet.isNew()) return pet;
            }
        }
        return null;
    }

    public Pet getPet(Integer id) {
        for (Pet pet : getPets()) {
            if (!pet.isNew() && pet.getId().equals(id)) return pet;
        }
        return null;
    }

    public void addVisit(Integer petId, Visit visit) {
        Pet pet = getPet(petId);
        pet.addVisit(visit);
    }
}
```

> **🧠 Concept: `@OneToMany` + `CascadeType.ALL`** — When you save an Owner, JPA automatically saves all their Pets too. The `@JoinColumn` tells JPA which column in the `pets` table links back to this owner.

### 2b. `OwnerRepository.java` — Database Access

**File:** `src/main/java/org/springframework/samples/petclinic/owner/OwnerRepository.java`

```java
package org.springframework.samples.petclinic.owner;

import java.util.Optional;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;

public interface OwnerRepository extends JpaRepository<Owner, Integer> {

    // Spring Data magic: this method name is parsed into a SQL query automatically!
    // "findByLastNameStartingWith" → SELECT * FROM owners WHERE last_name LIKE 'X%'
    Page<Owner> findByLastNameStartingWith(String lastName, Pageable pageable);

    Optional<Owner> findById(Integer id);
}
```

> **🧠 Concept: Derived Query Methods** — Spring Data reads your method name and creates the query. `findByLastNameStartingWith(String)` becomes `WHERE last_name LIKE ?%`. No SQL needed!

### 2c. `OwnerController.java` — Handling Web Requests

**File:** `src/main/java/org/springframework/samples/petclinic/owner/OwnerController.java`

```java
package org.springframework.samples.petclinic.owner;

import java.util.List;
import java.util.Optional;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.WebDataBinder;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.ModelAndView;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;
import jakarta.validation.Valid;

@Controller  // tells Spring: "I handle HTTP requests"
class OwnerController {

    private static final String VIEWS_OWNER_CREATE_OR_UPDATE_FORM = "owners/createOrUpdateOwnerForm";

    private final OwnerRepository owners;

    // Constructor injection — Spring automatically passes the OwnerRepository
    public OwnerController(OwnerRepository owners) {
        this.owners = owners;
    }

    // Security: prevent users from tampering with the "id" field in forms
    @InitBinder
    public void setAllowedFields(WebDataBinder dataBinder) {
        dataBinder.setDisallowedFields("id");
    }

    // This runs BEFORE every request that has {ownerId} in the URL
    @ModelAttribute("owner")
    public Owner findOwner(@PathVariable(name = "ownerId", required = false) Integer ownerId) {
        return ownerId == null ? new Owner()
            : this.owners.findById(ownerId)
                .orElseThrow(() -> new IllegalArgumentException("Owner not found with id: " + ownerId));
    }

    // GET /owners/new → show the empty creation form
    @GetMapping("/owners/new")
    public String initCreationForm() {
        return VIEWS_OWNER_CREATE_OR_UPDATE_FORM;
    }

    // POST /owners/new → process the submitted form
    @PostMapping("/owners/new")
    public String processCreationForm(@Valid Owner owner, BindingResult result,
                                       RedirectAttributes redirectAttributes) {
        if (result.hasErrors()) {
            redirectAttributes.addFlashAttribute("error", "There was an error in creating the owner.");
            return VIEWS_OWNER_CREATE_OR_UPDATE_FORM;
        }
        this.owners.save(owner);
        redirectAttributes.addFlashAttribute("message", "New Owner Created");
        return "redirect:/owners/" + owner.getId();
    }

    // GET /owners/find → show the search form
    @GetMapping("/owners/find")
    public String initFindForm() {
        return "owners/findOwners";
    }

    // GET /owners?lastName=xxx → search owners by last name with pagination
    @GetMapping("/owners")
    public String processFindForm(@RequestParam(defaultValue = "1") int page, Owner owner,
                                   BindingResult result, Model model) {
        String lastName = (owner.getLastName() == null) ? "" : owner.getLastName();
        Page<Owner> ownersResults = findPaginatedForOwnersLastName(page, lastName);

        if (ownersResults.isEmpty()) {
            result.rejectValue("lastName", "notFound", "not found");
            return "owners/findOwners";
        }
        if (ownersResults.getTotalElements() == 1) {
            owner = ownersResults.iterator().next();
            return "redirect:/owners/" + owner.getId();
        }

        // Multiple results: show the list
        List<Owner> listOwners = ownersResults.getContent();
        model.addAttribute("currentPage", page);
        model.addAttribute("totalPages", ownersResults.getTotalPages());
        model.addAttribute("totalItems", ownersResults.getTotalElements());
        model.addAttribute("listOwners", listOwners);
        return "owners/ownersList";
    }

    private Page<Owner> findPaginatedForOwnersLastName(int page, String lastname) {
        Pageable pageable = PageRequest.of(page - 1, 5);  // 5 results per page
        return owners.findByLastNameStartingWith(lastname, pageable);
    }

    // GET /owners/{id}/edit → show the edit form (pre-filled)
    @GetMapping("/owners/{ownerId}/edit")
    public String initUpdateOwnerForm() {
        return VIEWS_OWNER_CREATE_OR_UPDATE_FORM;  // @ModelAttribute already loaded the owner
    }

    // POST /owners/{id}/edit → save the changes
    @PostMapping("/owners/{ownerId}/edit")
    public String processUpdateOwnerForm(@Valid Owner owner, BindingResult result,
                                          @PathVariable("ownerId") int ownerId,
                                          RedirectAttributes redirectAttributes) {
        if (result.hasErrors()) {
            return VIEWS_OWNER_CREATE_OR_UPDATE_FORM;
        }
        owner.setId(ownerId);
        this.owners.save(owner);
        redirectAttributes.addFlashAttribute("message", "Owner Values Updated");
        return "redirect:/owners/{ownerId}";
    }

    // GET /owners/{id} → show owner details page
    @GetMapping("/owners/{ownerId}")
    public ModelAndView showOwner(@PathVariable("ownerId") int ownerId) {
        ModelAndView mav = new ModelAndView("owners/ownerDetails");
        Owner owner = this.owners.findById(ownerId)
            .orElseThrow(() -> new IllegalArgumentException("Owner not found with id: " + ownerId));
        mav.addObject(owner);
        return mav;
    }
}
```

> **🧠 Key Concepts in this Controller:**
> - `@GetMapping` / `@PostMapping` — maps HTTP methods to Java methods
> - `@Valid` + `BindingResult` — validates the form input and captures errors
> - `@ModelAttribute` — loads data before the handler method runs
> - `@InitBinder` — configures which form fields are allowed
> - `RedirectAttributes` — passes flash messages after redirect

---

## Step 3 — Build the Pet Module

### 3a. `PetType.java` — Simple lookup entity

```java
package org.springframework.samples.petclinic.owner;

import org.springframework.samples.petclinic.model.NamedEntity;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;

@Entity
@Table(name = "types")
public class PetType extends NamedEntity {
    // inherits id + name from NamedEntity. That's it!
}
```

### 3b. `Pet.java` — A pet belongs to an owner

```java
package org.springframework.samples.petclinic.owner;

import java.time.LocalDate;
import java.util.*;
import org.springframework.format.annotation.DateTimeFormat;
import org.springframework.samples.petclinic.model.NamedEntity;
import jakarta.persistence.*;

@Entity
@Table(name = "pets")
public class Pet extends NamedEntity {

    @Column
    @DateTimeFormat(pattern = "yyyy-MM-dd")
    private LocalDate birthDate;

    @ManyToOne                          // Many pets can have the same type
    @JoinColumn(name = "type_id")       // FK column in the pets table
    private PetType type;

    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.EAGER)
    @JoinColumn(name = "pet_id")
    @OrderBy("date ASC")
    private final Set<Visit> visits = new LinkedHashSet<>();

    // getters, setters, addVisit()
    public LocalDate getBirthDate() { return this.birthDate; }
    public void setBirthDate(LocalDate birthDate) { this.birthDate = birthDate; }
    public PetType getType() { return this.type; }
    public void setType(PetType type) { this.type = type; }
    public Collection<Visit> getVisits() { return this.visits; }
    public void addVisit(Visit visit) { getVisits().add(visit); }
}
```

> **🧠 Concept: `@ManyToOne`** — Multiple pets can be of the same type (e.g., many pets are "cat"). The `@JoinColumn` specifies the foreign key column.

### 3c. `PetTypeRepository.java`

```java
package org.springframework.samples.petclinic.owner;

import java.util.List;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

public interface PetTypeRepository extends JpaRepository<PetType, Integer> {

    @Query("SELECT ptype FROM PetType ptype ORDER BY ptype.name")  // custom JPQL query
    List<PetType> findPetTypes();
}
```

> **🧠 Concept: `@Query`** — When the method name convention isn't enough, write JPQL directly. JPQL is like SQL but uses Java class/field names instead of table/column names.

### 3d. `PetValidator.java` — Custom Validation

```java
package org.springframework.samples.petclinic.owner;

import org.springframework.util.StringUtils;
import org.springframework.validation.Errors;
import org.springframework.validation.Validator;

public class PetValidator implements Validator {

    @Override
    public void validate(Object obj, Errors errors) {
        Pet pet = (Pet) obj;

        if (!StringUtils.hasText(pet.getName()))
            errors.rejectValue("name", "required", "required");

        if (pet.isNew() && pet.getType() == null)
            errors.rejectValue("type", "required", "required");

        if (pet.getBirthDate() == null)
            errors.rejectValue("birthDate", "required", "required");
    }

    @Override
    public boolean supports(Class<?> clazz) {
        return Pet.class.isAssignableFrom(clazz);
    }
}
```

> **🧠 Concept: Custom Validator** — Sometimes `@NotBlank` annotations aren't flexible enough. Implement `Validator` for complex rules involving multiple fields.

### 3e. `PetTypeFormatter.java` — Convert between String ↔ PetType

```java
package org.springframework.samples.petclinic.owner;

import org.springframework.format.Formatter;
import org.springframework.stereotype.Component;
import java.text.ParseException;
import java.util.Locale;

@Component
public class PetTypeFormatter implements Formatter<PetType> {

    private final PetTypeRepository types;

    public PetTypeFormatter(PetTypeRepository types) { this.types = types; }

    @Override
    public String print(PetType petType, Locale locale) {
        return petType.getName();
    }

    @Override
    public PetType parse(String text, Locale locale) throws ParseException {
        for (PetType type : this.types.findPetTypes()) {
            if (type.getName().equals(text)) return type;
        }
        throw new ParseException("type not found: " + text, 0);
    }
}
```

> **🧠 Concept: `Formatter`** — When an HTML form sends the string "cat", Spring needs to know how to convert it to a `PetType` object. The Formatter handles this conversion.

### 3f. `PetController.java`

```java
package org.springframework.samples.petclinic.owner;

import java.time.LocalDate;
import java.util.*;
import org.springframework.stereotype.Controller;
import org.springframework.ui.ModelMap;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.WebDataBinder;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;
import jakarta.validation.Valid;

@Controller
@RequestMapping("/owners/{ownerId}")  // all URLs here start with /owners/{ownerId}
class PetController {

    private final OwnerRepository owners;
    private final PetTypeRepository types;

    public PetController(OwnerRepository owners, PetTypeRepository types) {
        this.owners = owners;
        this.types = types;
    }

    @ModelAttribute("types")
    public Collection<PetType> populatePetTypes() {
        return this.types.findPetTypes();  // dropdown data for pet types
    }

    @ModelAttribute("owner")
    public Owner findOwner(@PathVariable("ownerId") int ownerId) {
        return this.owners.findById(ownerId)
            .orElseThrow(() -> new IllegalArgumentException("Owner not found"));
    }

    @InitBinder("pet")
    public void initPetBinder(WebDataBinder dataBinder) {
        dataBinder.setValidator(new PetValidator());  // use our custom validator
    }

    @GetMapping("/pets/new")
    public String initCreationForm(Owner owner, ModelMap model) {
        Pet pet = new Pet();
        owner.addPet(pet);
        return "pets/createOrUpdatePetForm";
    }

    @PostMapping("/pets/new")
    public String processCreationForm(Owner owner, @Valid Pet pet, BindingResult result,
                                       RedirectAttributes redirectAttributes) {
        // Check for duplicate pet name
        if (pet.getName() != null && pet.isNew() && owner.getPet(pet.getName(), true) != null)
            result.rejectValue("name", "duplicate", "already exists");

        if (pet.getBirthDate() != null && pet.getBirthDate().isAfter(LocalDate.now()))
            result.rejectValue("birthDate", "typeMismatch.birthDate");

        if (result.hasErrors()) return "pets/createOrUpdatePetForm";

        owner.addPet(pet);
        this.owners.save(owner);
        redirectAttributes.addFlashAttribute("message", "New Pet has been Added");
        return "redirect:/owners/{ownerId}";
    }

    // Edit pet: GET shows form, POST processes it (similar pattern)
    @GetMapping("/pets/{petId}/edit")
    public String initUpdateForm() { return "pets/createOrUpdatePetForm"; }

    @PostMapping("/pets/{petId}/edit")
    public String processUpdateForm(Owner owner, @Valid Pet pet, BindingResult result,
                                     RedirectAttributes redirectAttributes) {
        if (result.hasErrors()) return "pets/createOrUpdatePetForm";
        // update existing pet details and save
        owner.addPet(pet);
        this.owners.save(owner);
        redirectAttributes.addFlashAttribute("message", "Pet details has been edited");
        return "redirect:/owners/{ownerId}";
    }
}
```

---

## Step 4 — Build the Visit Module

### 4a. `Visit.java`

```java
package org.springframework.samples.petclinic.owner;

import java.time.LocalDate;
import org.springframework.format.annotation.DateTimeFormat;
import org.springframework.samples.petclinic.model.BaseEntity;
import jakarta.persistence.*;
import jakarta.validation.constraints.NotBlank;

@Entity
@Table(name = "visits")
public class Visit extends BaseEntity {

    @Column(name = "visit_date")
    @DateTimeFormat(pattern = "yyyy-MM-dd")
    private LocalDate date;

    @NotBlank
    private String description;

    public Visit() {
        this.date = LocalDate.now();  // default: today's date
    }

    // getters and setters
    public LocalDate getDate() { return this.date; }
    public void setDate(LocalDate date) { this.date = date; }
    public String getDescription() { return this.description; }
    public void setDescription(String description) { this.description = description; }
}
```

### 4b. `VisitController.java`

```java
package org.springframework.samples.petclinic.owner;

import java.util.Map;
import java.util.Optional;
import org.springframework.stereotype.Controller;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.WebDataBinder;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;
import jakarta.validation.Valid;

@Controller
class VisitController {

    private final OwnerRepository owners;

    public VisitController(OwnerRepository owners) { this.owners = owners; }

    @InitBinder
    public void setAllowedFields(WebDataBinder dataBinder) {
        dataBinder.setDisallowedFields("id");
    }

    // Runs before EVERY request — loads the owner, pet, and creates a new Visit
    @ModelAttribute("visit")
    public Visit loadPetWithVisit(@PathVariable("ownerId") int ownerId,
                                   @PathVariable("petId") int petId,
                                   Map<String, Object> model) {
        Owner owner = owners.findById(ownerId)
            .orElseThrow(() -> new IllegalArgumentException("Owner not found"));
        Pet pet = owner.getPet(petId);
        model.put("pet", pet);
        model.put("owner", owner);

        Visit visit = new Visit();
        pet.addVisit(visit);
        return visit;
    }

    @GetMapping("/owners/{ownerId}/pets/{petId}/visits/new")
    public String initNewVisitForm() {
        return "pets/createOrUpdateVisitForm";
    }

    @PostMapping("/owners/{ownerId}/pets/{petId}/visits/new")
    public String processNewVisitForm(@ModelAttribute Owner owner, @PathVariable int petId,
                                       @Valid Visit visit, BindingResult result,
                                       RedirectAttributes redirectAttributes) {
        if (result.hasErrors()) return "pets/createOrUpdateVisitForm";

        owner.addVisit(petId, visit);
        this.owners.save(owner);
        redirectAttributes.addFlashAttribute("message", "Your visit has been booked");
        return "redirect:/owners/{ownerId}";
    }
}
```

---

## Step 5 — Build the Vet Module

### 5a. `Specialty.java`

```java
package org.springframework.samples.petclinic.vet;

import org.springframework.samples.petclinic.model.NamedEntity;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;

@Entity
@Table(name = "specialties")
public class Specialty extends NamedEntity { }
```

### 5b. `Vet.java`

```java
package org.springframework.samples.petclinic.vet;

import java.util.*;
import java.util.stream.Collectors;
import org.springframework.samples.petclinic.model.Person;
import jakarta.persistence.*;

@Entity
@Table(name = "vets")
public class Vet extends Person {

    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(name = "vet_specialties",
        joinColumns = @JoinColumn(name = "vet_id"),
        inverseJoinColumns = @JoinColumn(name = "specialty_id"))
    private Set<Specialty> specialties;

    public List<Specialty> getSpecialties() {
        if (this.specialties == null) this.specialties = new HashSet<>();
        return this.specialties.stream()
            .sorted(Comparator.comparing(s -> s.getName()))
            .collect(Collectors.toList());
    }

    public int getNrOfSpecialties() {
        return (this.specialties == null) ? 0 : this.specialties.size();
    }
}
```

> **🧠 Concept: `@ManyToMany`** — A vet can have many specialties, and a specialty can belong to many vets. The `@JoinTable` defines the linking table (`vet_specialties`).

### 5c. `VetRepository.java` — With Caching!

```java
package org.springframework.samples.petclinic.vet;

import java.util.Collection;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.repository.Repository;
import org.springframework.transaction.annotation.Transactional;

public interface VetRepository extends Repository<Vet, Integer> {

    @Transactional(readOnly = true)
    @Cacheable("vets")  // results are cached! Subsequent calls return cached data
    Collection<Vet> findAll();

    @Transactional(readOnly = true)
    @Cacheable("vets")
    Page<Vet> findAll(Pageable pageable);
}
```

> **🧠 Concept: `@Cacheable`** — The first call to `findAll()` queries the database. All subsequent calls return the cached result until the cache expires. Great for data that rarely changes.

### 5d. `VetController.java` — HTML + JSON endpoints

```java
package org.springframework.samples.petclinic.vet;

import java.util.List;
import org.springframework.data.domain.*;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;

@Controller
class VetController {

    private final VetRepository vetRepository;

    public VetController(VetRepository vetRepository) {
        this.vetRepository = vetRepository;
    }

    // HTML page: /vets.html
    @GetMapping("/vets.html")
    public String showVetList(@RequestParam(defaultValue = "1") int page, Model model) {
        Page<Vet> paginated = vetRepository.findAll(PageRequest.of(page - 1, 5));
        model.addAttribute("currentPage", page);
        model.addAttribute("totalPages", paginated.getTotalPages());
        model.addAttribute("totalItems", paginated.getTotalElements());
        model.addAttribute("listVets", paginated.getContent());
        return "vets/vetList";
    }

    // JSON API: /vets → returns JSON automatically thanks to @ResponseBody
    @GetMapping("/vets")
    public @ResponseBody Vets showResourcesVetList() {
        Vets vets = new Vets();
        vets.getVetList().addAll(this.vetRepository.findAll());
        return vets;
    }
}
```

> **🧠 Concept: `@ResponseBody`** — Instead of returning an HTML view name, this annotation tells Spring to serialize the return object to JSON and send it directly. This is how you create REST APIs!

---

## Step 6 — Set Up the Database

We use **H2** (in-memory database) — no installation needed!

### 6a. `schema.sql` — Create the Tables

**File:** `src/main/resources/db/h2/schema.sql`

```sql
DROP TABLE vet_specialties IF EXISTS;
DROP TABLE vets IF EXISTS;
DROP TABLE specialties IF EXISTS;
DROP TABLE visits IF EXISTS;
DROP TABLE pets IF EXISTS;
DROP TABLE types IF EXISTS;
DROP TABLE owners IF EXISTS;

CREATE TABLE vets (
  id         INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  first_name VARCHAR(30),
  last_name  VARCHAR(30)
);

CREATE TABLE specialties (
  id   INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  name VARCHAR(80)
);

CREATE TABLE vet_specialties (
  vet_id       INTEGER NOT NULL,
  specialty_id INTEGER NOT NULL
);
ALTER TABLE vet_specialties ADD CONSTRAINT fk_vet_specialties_vets FOREIGN KEY (vet_id) REFERENCES vets (id);
ALTER TABLE vet_specialties ADD CONSTRAINT fk_vet_specialties_specialties FOREIGN KEY (specialty_id) REFERENCES specialties (id);

CREATE TABLE types (
  id   INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  name VARCHAR(80)
);

CREATE TABLE owners (
  id         INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  first_name VARCHAR(30),
  last_name  VARCHAR_IGNORECASE(30),
  address    VARCHAR(255),
  city       VARCHAR(80),
  telephone  VARCHAR(20)
);

CREATE TABLE pets (
  id         INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  name       VARCHAR(30),
  birth_date DATE,
  type_id    INTEGER NOT NULL,
  owner_id   INTEGER
);
ALTER TABLE pets ADD CONSTRAINT fk_pets_owners FOREIGN KEY (owner_id) REFERENCES owners (id);
ALTER TABLE pets ADD CONSTRAINT fk_pets_types FOREIGN KEY (type_id) REFERENCES types (id);

CREATE TABLE visits (
  id          INTEGER GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  pet_id      INTEGER,
  visit_date  DATE,
  description VARCHAR(255)
);
ALTER TABLE visits ADD CONSTRAINT fk_visits_pets FOREIGN KEY (pet_id) REFERENCES pets (id);
```

### 6b. `data.sql` — Seed Data (sample records)

**File:** `src/main/resources/db/h2/data.sql`

```sql
-- Vets
INSERT INTO vets VALUES (default, 'James', 'Carter');
INSERT INTO vets VALUES (default, 'Helen', 'Leary');
INSERT INTO vets VALUES (default, 'Linda', 'Douglas');
INSERT INTO vets VALUES (default, 'Rafael', 'Ortega');
INSERT INTO vets VALUES (default, 'Henry', 'Stevens');
INSERT INTO vets VALUES (default, 'Sharon', 'Jenkins');

-- Specialties
INSERT INTO specialties VALUES (default, 'radiology');
INSERT INTO specialties VALUES (default, 'surgery');
INSERT INTO specialties VALUES (default, 'dentistry');

-- Vet ↔ Specialty links
INSERT INTO vet_specialties VALUES (2, 1);
INSERT INTO vet_specialties VALUES (3, 2);
INSERT INTO vet_specialties VALUES (3, 3);
INSERT INTO vet_specialties VALUES (4, 2);
INSERT INTO vet_specialties VALUES (5, 1);

-- Pet Types
INSERT INTO types VALUES (default, 'cat');
INSERT INTO types VALUES (default, 'dog');
INSERT INTO types VALUES (default, 'lizard');
INSERT INTO types VALUES (default, 'snake');
INSERT INTO types VALUES (default, 'bird');
INSERT INTO types VALUES (default, 'hamster');

-- Owners
INSERT INTO owners VALUES (default, 'George', 'Franklin', '110 W. Liberty St.', 'Madison', '6085551023');
INSERT INTO owners VALUES (default, 'Betty', 'Davis', '638 Cardinal Ave.', 'Sun Prairie', '6085551749');

-- Pets
INSERT INTO pets VALUES (default, 'Leo', '2010-09-07', 1, 1);
INSERT INTO pets VALUES (default, 'Basil', '2012-08-06', 6, 2);

-- Visits
INSERT INTO visits VALUES (default, 1, '2013-01-01', 'rabies shot');
INSERT INTO visits VALUES (default, 2, '2013-01-02', 'rabies shot');
```

> **🧠 Concept: Schema Init** — Spring Boot runs `schema.sql` first (creates tables), then `data.sql` (inserts sample data). This happens on every restart with H2 since it's in-memory.

---

## Step 7 — Build the Thymeleaf Templates

Templates live in `src/main/resources/templates/`. Here are the key ones:

### 7a. Layout Fragment (`fragments/layout.html`)

This is the **master template** — every page uses it for consistent header/footer/navigation.

```html
<!DOCTYPE html>
<html xmlns:th="https://www.thymeleaf.org" th:fragment="layout (template, title)">
<head>
    <meta charset="UTF-8"/>
    <title th:text="${title}">PetClinic</title>
    <!-- Bootstrap CSS from WebJars -->
    <link rel="stylesheet" th:href="@{/webjars/bootstrap/css/bootstrap.min.css}"/>
    <link rel="stylesheet" th:href="@{/resources/css/petclinic.css}"/>
</head>
<body>
    <nav class="navbar navbar-expand-lg navbar-dark" style="background-color: #0d6efd;">
        <a class="navbar-brand" th:href="@{/}">PetClinic</a>
        <ul class="navbar-nav">
            <li class="nav-item"><a class="nav-link" th:href="@{/owners/find}">Find Owners</a></li>
            <li class="nav-item"><a class="nav-link" th:href="@{/vets.html}">Veterinarians</a></li>
        </ul>
    </nav>

    <div class="container mt-4">
        <div th:replace="${template}"><!-- page content goes here --></div>
    </div>
</body>
</html>
```

> **🧠 Concept: Thymeleaf Fragments** — `th:fragment` defines reusable chunks. `th:replace` inserts them. Think of it like components in React.

### 7b. Welcome Page (`welcome.html`)

```html
<html xmlns:th="https://www.thymeleaf.org"
      th:replace="~{fragments/layout :: layout (~{::body},'home')}">
<body>
    <h2>Welcome to PetClinic</h2>
    <img src="../static/resources/images/pets.png" th:src="@{/resources/images/pets.png}"/>
</body>
</html>
```

### 7c. Find Owners Form (`owners/findOwners.html`)

```html
<html xmlns:th="https://www.thymeleaf.org"
      th:replace="~{fragments/layout :: layout (~{::body},'owners')}">
<body>
    <h2>Find Owners</h2>
    <form th:object="${owner}" th:action="@{/owners}" method="get" class="form-horizontal">
        <div class="form-group">
            <label>Last name</label>
            <input th:field="*{lastName}" class="form-control" size="30" maxlength="80"/>
            <span class="text-danger" th:errors="*{lastName}"></span>
        </div>
        <button class="btn btn-primary" type="submit">Find Owner</button>
    </form>
    <a th:href="@{/owners/new}" class="btn btn-success">Add Owner</a>
</body>
</html>
```

> **🧠 Concept: `th:object` and `th:field`** — These bind an HTML form directly to a Java object. `th:object="${owner}"` says "this form maps to the Owner object". `th:field="*{lastName}"` maps this input to `owner.lastName`.

---

## Step 8 — System Configuration

### 8a. Cache Configuration

**File:** `src/main/java/.../system/CacheConfiguration.java`

```java
@Configuration(proxyBeanMethods = false)
@EnableCaching  // enables the @Cacheable annotation
class CacheConfiguration {

    @Bean
    public JCacheManagerCustomizer petclinicCacheConfigurationCustomizer() {
        return cm -> cm.createCache("vets", new MutableConfiguration<>().setStatisticsEnabled(true));
    }
}
```

### 8b. Internationalization (i18n) — `WebConfiguration.java`

```java
@Configuration
public class WebConfiguration implements WebMvcConfigurer {

    @Bean
    public LocaleResolver localeResolver() {
        SessionLocaleResolver resolver = new SessionLocaleResolver();
        resolver.setDefaultLocale(Locale.ENGLISH);
        return resolver;
    }

    @Bean
    public LocaleChangeInterceptor localeChangeInterceptor() {
        LocaleChangeInterceptor interceptor = new LocaleChangeInterceptor();
        interceptor.setParamName("lang");  // use ?lang=de to switch languages
        return interceptor;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(localeChangeInterceptor());
    }
}
```

Message files go in `src/main/resources/messages/`:
- `messages.properties` — default (English)
- `messages_de.properties` — German
- `messages_es.properties` — Spanish

### 8c. Welcome & Error Controllers

```java
// WelcomeController.java — maps "/" to the welcome page
@Controller
class WelcomeController {
    @GetMapping("/")
    public String welcome() { return "welcome"; }
}

// CrashController.java — demonstrates error handling
@Controller
class CrashController {
    @GetMapping("/oups")
    public String triggerException() {
        throw new RuntimeException("Expected: demo exception");  // error.html is shown
    }
}
```

---

## Step 9 — Application Properties

**File:** `src/main/resources/application.properties`

```properties
# Database: use H2 in-memory database
database=h2
spring.sql.init.schema-locations=classpath*:db/${database}/schema.sql
spring.sql.init.data-locations=classpath*:db/${database}/data.sql

# Thymeleaf
spring.thymeleaf.mode=HTML

# JPA — we manage the schema ourselves via SQL scripts
spring.jpa.hibernate.ddl-auto=none
spring.jpa.open-in-view=false

# Maps Java camelCase field names to snake_case column names
spring.jpa.hibernate.naming.physical-strategy=org.hibernate.boot.model.naming.PhysicalNamingStrategySnakeCaseImpl

# i18n
spring.messages.basename=messages/messages

# Actuator — expose all endpoints
management.endpoints.web.exposure.include=*

# Logging
logging.level.org.springframework=INFO

# Cache static resources for 12 hours
spring.web.resources.cache.cachecontrol.max-age=12h
```

---

## Step 10 — Run and Test

### Run the Application

```bash
# Windows
.\mvnw.cmd spring-boot:run

# Mac/Linux
./mvnw spring-boot:run
```

### Open in Browser

- **Home:** http://localhost:8080
- **Find Owners:** http://localhost:8080/owners/find
- **All Owners:** http://localhost:8080/owners
- **Vets (HTML):** http://localhost:8080/vets.html
- **Vets (JSON API):** http://localhost:8080/vets
- **Actuator:** http://localhost:8080/actuator
- **Error Demo:** http://localhost:8080/oups

### What to Try

1. Search for owners by last name (try "Davis")
2. Click on an owner → view their pets and visit history
3. Add a new owner via "Add Owner" button
4. Add a pet to an owner
5. Book a visit for a pet
6. Check the Vets page and its JSON API
7. Try the actuator health endpoint: `/actuator/health`

---

## Concepts You Learned

| # | Concept | Where You Saw It |
|---|---------|-----------------|
| 1 | `@SpringBootApplication` | `PetClinicApplication.java` |
| 2 | MVC Pattern | Controller → Service → Repository |
| 3 | `@Entity` + JPA | Owner, Pet, Vet, Visit, etc. |
| 4 | `@MappedSuperclass` | BaseEntity, NamedEntity, Person |
| 5 | `@OneToMany` / `@ManyToOne` | Owner→Pet, Pet→Visit |
| 6 | `@ManyToMany` | Vet→Specialty |
| 7 | Spring Data Repositories | OwnerRepository, VetRepository |
| 8 | Derived Query Methods | `findByLastNameStartingWith()` |
| 9 | `@Query` (JPQL) | PetTypeRepository |
| 10 | `@Cacheable` | VetRepository |
| 11 | Thymeleaf Templates | All HTML files |
| 12 | `th:object` / `th:field` | Form binding |
| 13 | `@Valid` + `BindingResult` | Form validation |
| 14 | Custom `Validator` | PetValidator |
| 15 | `Formatter` | PetTypeFormatter |
| 16 | `@InitBinder` | OwnerController, PetController |
| 17 | `@ModelAttribute` | Pre-loading data |
| 18 | `@ResponseBody` | JSON API in VetController |
| 19 | Pagination | `Page<Owner>`, `PageRequest` |
| 20 | i18n | WebConfiguration + messages.properties |
| 21 | Spring Boot Actuator | `/actuator` endpoints |
| 22 | H2 Database + SQL Init | schema.sql, data.sql |
| 23 | Constructor Injection (DI) | Every controller |
| 24 | Flash Attributes | `RedirectAttributes` |

---

> **🎉 Congratulations!** You've just built a production-quality Spring Boot MVC application with a real database, form validation, caching, internationalization, and both HTML and JSON endpoints. This covers ~90% of what you'll need for real-world Spring Boot web apps!
