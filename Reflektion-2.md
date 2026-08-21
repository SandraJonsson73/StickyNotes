# Reflektion: SQL Injection (A03/A05) - OWASP 2025

## Sammanfattning av Arbetet
Jag identifierade en SQL Injection-sårbarhet i GET-endpointen, demonstrerade attacken och fixade den genom att konvertera från raw SQL till säker parameteriserad LINQ.

---

## Sårbarhet Identifierad

### SQL Injection i GET List Endpoint
**Plats:** `Notes.Api/Controllers/NotesController.cs:31-41`  
**Allvarlighetsgrad:** HÖG

---

## Attackdemonstration

### Innan Fixning (Sårbar)
```csharp
.FromSqlRaw($"SELECT * FROM Notes WHERE Author='{user.Username}' AND Content LIKE '%{containing}%' ORDER BY Id")
```

### Attack #1: Hämta Alla Användarens Anteckningar
```
GET /notes?containing=%
SQL: SELECT * FROM Notes WHERE Author='Test' AND Content LIKE '%%%' ORDER BY Id
Resultat: Alla "Test":s anteckningar (matcher allt)
```

### Attack #2: Hämta Alla Anteckningar från Specifik Användare
Om angripare vet ett annat användarnamn (t.ex. "Admin"):
```
GET /notes?containing=%' OR Author LIKE '%Admin
SQL: SELECT * FROM Notes WHERE Author='Test' AND Content LIKE '%%' OR Author LIKE '%Admin%' ORDER BY Id
Resultat: Både Test:s OCH Admin:s anteckningar
```

### Attack #3: Hämta ALLA Anteckningar från ALLA Användare
```
GET /notes?containing=%' OR Author LIKE '%
SQL: SELECT * FROM Notes WHERE Author='Test' AND Content LIKE '%%' OR Author LIKE '%%' ORDER BY Id
Resultat: ALLA anteckningar i systemet!
```

### Attack #4: Potentiell Datamodifiering
```
GET /notes?containing=%'; DROP TABLE Notes; --%
SQL: SELECT * FROM Notes WHERE Author='Test' AND Content LIKE '%%'; DROP TABLE Notes; -- %'
Resultat: Databastabellen kan raderas!
```

---

## Fixningen

### Före (Sårbar)
```csharp
public Note[] Get([FromQuery] string containing)
{
    var authorizationHeader = Request.Headers["Authorization"];
    var user = BasicAuthenticationHandler.GetUserFrom(authorizationHeader);

    // ❌ RAW SQL - Användarinmatning direktinsatt
    return _database.Notes
        .FromSqlRaw($"SELECT * FROM Notes WHERE Author='{user.Username}' AND Content LIKE '%{containing}%' ORDER BY Id")
        .ToArray();
}
```

### Efter (Fixad)
```csharp
public Note[] Get([FromQuery] string containing)
{
    var authorizationHeader = Request.Headers["Authorization"];
    var user = BasicAuthenticationHandler.GetUserFrom(authorizationHeader);

    // ✅ SÄKER LINQ - Parametriserad automatiskt av EF Core
    return _database.Notes
        .Where(note => note.Author == user.Username)
        .Where(note => string.IsNullOrEmpty(containing) ? true : note.Content.Contains(containing))
        .OrderBy(note => note.Id)
        .ToArray();
}
```

---

## Varför Denna Fix Är Säker

### 1. LINQ-uttryck Konverteras Till Parameteriserade SQL
Entity Framework Core konverterar:
```csharp
.Where(note => note.Content.Contains(containing))
```

Till något liknande:
```sql
WHERE Content LIKE @p0
-- Där @p0 = "%{användarens input}%"
```

Användarinmatningen är aldrig en del av SQL-strukturen - det är bara data.

### 2. Escaping Hanteras Automatiskt
- Speciella tecken som `'`, `"`, `;` är bara data
- Kan inte tolkas som SQL-kommandot
- EF Core hanterar allt automatiskt

### 3. Null-kontroll
```csharp
string.IsNullOrEmpty(containing) ? true : note.Content.Contains(containing)
```
- Om söktermen är tom/null, returnera allt (true)
- Annars sök efter innehål
- Förhindrar null-reference exceptions

---

## Vad Jag Lärde Mig

### 1. Raw SQL = Riskfyllt
**Insikt:** `FromSqlRaw` är aldrig vägen för användarinmatning. Ordet "Raw" är en varning.

**Lärdom:** 
- `FromSqlRaw` = Only för statiska SQL-queries
- `FromSqlInterpolated` = För queries med variabler (men fortfarande riskfyllt)
- **LINQ = Den säkra vägen** för dataåtkomst

### 2. Parametrisering Är Nyckelord
**Insikt:** SQL Injection fungerar bara om användardata blir en del av SQL-strukturen. Om det bara är data blir det omöjligt.

**Lärdom:** 
- Parametriserade queries är THE försvar mot SQL injection
- Varje ORM (Entity Framework, Hibernate, etc.) parameteriserar automatiskt
- Detta är #1 försvar enligt OWASP

### 3. LIKE är Speciell
**Insikt:** LIKE är unikt eftersom `%` och `_` är speciella tecken i mönstret. `Contains()` är den säkra LINQ-ekvivalenten.

**Lärdom:**
- `Contains()` = Söker efter substring (LIKE i SQL)
- `StartsWith()` = Söker från början
- `EndsWith()` = Söker från slutet
- Alla är säkra när de använder LINQ

### 4. Tidiga Åtgärder Var Värdefulla
**Insikt:** Jag fixade denna redan under A01-övningen, men nu fick jag bekräftelse på varför.

**Lärdom:** 
- Säkerhetsåtgärder är ofta överlappande
- A01 (Access Control) + A03 (Injection) gick hand i hand här
- Att fixa en sårbarhet kan faktiskt fixa flera!

### 5. Kontext Spelar Roll
**Insikt:** `FROM_SqlRaw` kan vara OK för rapportgenerering där ingen användarinmatning är inblandad.

**Lärdom:**
- Ingenstans användarindata blandas direkt i SQL - aldrig!
- Använd LINQ när användardata är inblandad
- Använd parameteriserade queries om raw SQL är nödvändigt

---

## Risker Med SQL Injection

### Dataintegritet (Integrity)
- ❌ Data kan modifieras eller raderas
- ❌ Falsk data kan skapas
- ❌ Auktoriseringskontroller kan kringgås

### Konfidentialitet (Confidentiality)
- ❌ Alla data i databasen kan läckas
- ❌ Känslig info om andra användare exponeras
- ❌ Systemarkitektur kan avslöjas

### Tillgänglighet (Availability)
- ❌ Data kan raderas (DROP TABLE)
- ❌ Databas kan låsas
- ❌ Server kan få överbelastning

### Compliance & Juridik
- ❌ GDPR-kränkning (obehörig dataåtkomst)
- ❌ PCI DSS-kränkning (om finansdata är inblandad)
- ❌ Möjliga höga böter

---

## Hur Förebygges SQL Injection I Framtida Projekt?

### 1. Gyllene Regel: Aldrig Raw SQL Med Användarinmatning
```csharp
// ❌ ALDRIG göra detta:
.FromSqlRaw($"SELECT * FROM Table WHERE Column = '{userInput}'")

// ✅ ALLTID göra detta:
.Where(x => x.Column == userInput)

// ✅ ELLER detta (om raw SQL är oundviklig):
.FromSqlInterpolated($"SELECT * FROM Table WHERE Column = {userInput}")
```

### 2. ORM Alltid Framför Raw SQL
- Entity Framework Core
- Dapper (med parametrar)
- NHibernate
- Alla parameteriserar automatiskt

### 3. Input Validation + Output Encoding
```csharp
// Validera input-längd
if (containing.Length > 100)
    return BadRequest("Search term too long");

// Encoding är en extra försiktighetsmekanism
var safeTerm = Uri.EscapeDataString(containing);
```

### 4. Databaskonto Med Minsta Behörighet
- Applikationskonto bör INTE kunna DROP/ALTER tables
- Separate read-only account för rapporter
- Admin-konto får ingen publiken trafik

### 5. Databas-nivå Skydd
- Aktivera SQL Server Audit
- Aktivera TDE (Transparent Data Encryption)
- Använd Row-Level Security för extra åtkomstkontroll

### 6. Testing & Automation
```csharp
[Test]
public void Get_WithSqlInjectionPayload_ReturnsNoExtraData()
{
    // Arrange
    var injectionPayload = "%' OR '1'='1";
    
    // Act
    var result = controller.Get(injectionPayload);
    
    // Assert - Ska bara returnera inloggad users anteckningar
    Assert.AreEqual(1, result.Count(n => n.Author == "Test"));
}
```

---

## Reflektion: Personligt Lärande

### Överraskning #1: Överlappande Sårbarheter
Jag trodde SQL Injection var helt separat från Access Control, men de är tätt sammankopplade. En sårbar SQL-query kan kringgå auktoriseringskontroller!

### Överraskning #2: Enkelheten I Fixningen
Jag trodde fixningen skulle vara komplicerad, men det var bara att byta från raw SQL till LINQ. Entity Framework gör all svår del.

### Överraskning #3: Parametrisering Är Magisk
Förstäelse av att parametrisering helt förhindrar SQL injection - attacken kan inte ens funka - var game-changing för mig.

---
