# Reflektion: Bruten Åtkomstkontroll (A01) - OWASP 2025

## Sammanfattning av Arbetet
Jag identifierade, demonstrerade, fixade och verifierade **fyra åtkomstkontrollsårbarheter** i NotesController.

---

## Sårbarheter Funna och Fixade

### 1. ✅ GET Single Note - Unauthorized Access (REDAN FIXAD I KODEN)
**Status:** Redan implementerat vid min kodgranskning
- Koden hade redan rätt auktoriseringskontroll
- Kontrollerar att `note.Author == user.Username` innan retur
- Returnerar 403 Forbidden vid obehörig åtkomst

### 2. ✅ PATCH Endpoint - No Authorization (FIXAD)
**Problem:** Vilken autentiserad användare som helst kunde ändra vilken anteckning som helst
```csharp
// FÖRE (sårbar):
public ActionResult<Note> Patch([FromRoute] int noteId, [FromBody] UpdateNote patchNote)
{
    var note = _database.Notes.Find(noteId);
    if (note == null) return NotFound(...);
    
    note.Content = patchNote.Content;  // ❌ Ingen ägarskaps-check
    _database.SaveChanges();
    return Ok(note);
}

// EFTER (fixad):
public ActionResult<Note> Patch([FromRoute] int noteId, [FromBody] UpdateNote patchNote)
{
    var note = _database.Notes.Find(noteId);
    if (note == null) return NotFound(...);
    
    var user = BasicAuthenticationHandler.GetUserFrom(...);
    if (note.Author != user.Username)  // ✅ Ägarskaps-check
        return Forbid();
    
    note.Content = patchNote.Content;
    _database.SaveChanges();
    return Ok(note);
}
```

### 3. ✅ DELETE Endpoint - No Authorization (FIXAD)
**Problem:** Vilken autentiserad användare som helst kunde ta bort vilken anteckning som helst
- Samma lösning som PATCH - lade till ägarskaps-check före borttagning
- Returnerar nu 403 Forbidden för obehörig åtkomst

### 4. ✅ GET List Endpoint - SQL Injection (FIXAD)
**Problem:** Raw SQL med string interpolation - kunde kringgå auktorisering
```csharp
// FÖRE (sårbar SQL injection):
return _database.Notes
    .FromSqlRaw($"SELECT * FROM Notes WHERE Author='{user.Username}' AND Content LIKE '%{containing}%'")
    .ToArray();

// EFTER (säker parameteriserad LINQ):
var searchPattern = $"%{containing}%";
return _database.Notes
    .Where(n => n.Author == user.Username && EF.Functions.Like(n.Content, searchPattern))
    .OrderBy(n => n.Id)
    .ToArray();
```

**Varför detta är säkert:**
- EF.Functions.Like använder parameteriserade queries automatiskt
- Användarinmatning kan inte injicera SQL
- Samma säkerhetsnivå som alla andra EF-operationer

---

## Vad Jag Lärde Mig

### 1. Konsistens är Kritiskt
**Insikt:** Jag märkte att GET Single Note-endpointen och Move-endpointen redan HAD auktoriseringskontrollen implementerad, men detta mönster var INTE tillämpad på PATCH och DELETE.

**Lärdom:** En developers goda intention att fixa en endpoint gör inte problemet löst globalt - samma mönster måste tillämpas ÖVERALLT.

### 2. Authorization Kontroll Måste Vara på Servernsidan ALLTID
**Insikt:** En angripare kan:
- Ändra klient-sidkoden
- Skapa egna HTTP-requests direkt
- Spoof API-anrop från andra verktyg

**Lärdom:** Server-sidans auktorisering är den ENDA sanningen. Utan det finns det ingen säkerhet.

### 3. Auktorisering Måste Kontrolleras För VARJE Operation
**Insikt:** Innan jag fixade detta sa jag: "Varför behövs det? Vi har ju [Authorize] på controllern."

**Lärdom:** `[Authorize]` säger bara "användaren måste vara inloggad" - det säger INGENTING om vilken data de får tillgång till. Man måste explicit kontrollera ägarskap för varje resurs.

### 4. Raw SQL är en Väg Till Katastrofen
**Insikt:** När jag såg `FromSqlRaw($"...{containing}%...")` tänkte jag direkt: "SQL injection!"

**Lärdom:** String interpolation i SQL är aldrig vägen. Använd alltid:
- Entity Framework LINQ (sämsta case)
- Parameteriserade queries (bättre)
- ORM-metoder som EF.Functions.Like (bäst)

### 5. Attacköversikt Är Ovärderligt
**Insikt:** Genom att tänka som angripare kunde jag identifiera var säkerheten bröts:
- "Om jag är användare A, kan jag sedan dölja användare B:s anteckning?"
- "Kan jag mata in speciella tecken i sökfältet för att få andras anteckningar?"

**Lärdom:** Security review måste alltid inkludera "Vad kan en angripare göra här?"

---

## Risker Som Denna Sårbarhet Medför

### Integritet (Confidentiality)
- ❌ Angripare kan läsa andras privata anteckningar
- ❌ SQL-injection kan avslöja känslig data från databasen

### Integritet (Integrity)  
- ❌ Angripare kan ändra/modifiera andras anteckningar
- ❌ Data manipulation utan revision trail

### Tillgänglighet (Availability)
- ❌ Angripare kan ta bort alla anteckningar i systemet
- ❌ Denial of Service genom massiv borttagning

### Compliance & Juridik
- ❌ GDPR-kränkning - unauthorized access to personal data
- ❌ Possible fines up to 4% of annual revenue

---

## Hur Förebygges Detta I Framtida Projekt?

### 1. Implementera En Säker Access-Control Mönster
**Från Start:**
```csharp
// Skapa en helper-metod för återanvändning
private bool IsNoteOwner(Note note, User currentUser)
{
    return note.Author == currentUser.Username;
}

// Använd i varje endpoint
if (!IsNoteOwner(note, user))
    return Forbid();
```

### 2. Code Review Checklista
Innan merge ska man fråga:
- [ ] Kräver denna endpoint autentisering?
- [ ] Kräver denna endpoint auktorisering för specifik resurs?
- [ ] Finns det ett EXPLICIT ägarskaps-check?
- [ ] Är all användarinmatning parameteriserad?
- [ ] Returnerar vi 403 för obehörig åtkomst (inte 404)?

### 3. Automatiserad Testing
```csharp
[Test]
public void Delete_NoteOwnedByOtherUser_ReturnsForbidden()
{
    // Arrange: Skapa två användare och en anteckning
    var userA = new User { Username = "A" };
    var userB = new User { Username = "B" };
    var note = new Note { Author = "B", Id = 1 };
    
    // Act: UserA försöker ta bort UserB:s anteckning
    var result = controller.Delete(noteId: 1, authenticatedUser: userA);
    
    // Assert: Ska returnera Forbid
    Assert.IsInstanceOfType(result, typeof(ForbidResult));
}
```

### 4. Security Scanning i CI/CD
- Använd Static Application Security Testing (SAST) för att flagga:
  - Raw SQL queries
  - Saknade authorization checks
  - Potentiell privilege escalation

### 5. Architecture-level Säkerhet
- Implementera Role-Based Access Control (RBAC) från början
- Använd Policy-based authorization i ASP.NET Core
- Dokumentera access control krav för varje endpoint

---

## Reflektion: Personligt Lärande

### Vad Väckte Min Uppmärksamhet?
Det som väckte mitt intresse var **inkonsistensen** - att GET single note HAD auktorisering men PATCH/DELETE inte. Det antydde att:
1. Det fanns en medvetenhet om problemet (någon fixade GET)
2. Men implementeringen var inkomplett (glömde andra endpoints)

Det är en klassisk human error - "Åh, det löste jag i en endpoint, det måste väl vara gjort överallt" 🙅

### Varför Är Detta Viktigt?
Bruten åtkomstkontroll är #1 på OWASP List 2025 för en anledning - det är:
- Lätt att introducera (man glömmer bara en check)
- Lätt att exploatera (bara ändra ett ID i URL:en)
- Svårt att upptäcka (går ofta oupptäckt tills någon missbrukar det)

### Min Största Insikt
**"Authorization måste vara lika mekanisk som att låsa en dörr - man gör det VARJE gång, inte bara ibland."**

Samma sätt som man inte säger "Jag lås dörren ibland när jag är rädd" - man lås den alltid.

---

## Nästa Steg: Andra OWASP-Sårbarheter
Nu när jag förstår A01 (Broken Access Control), är jag redo att attackera:
- A02: Cryptographic Failures
- A03: Injection (SQL, XSS, etc.)
- A04: Insecure Design
... och så vidare

