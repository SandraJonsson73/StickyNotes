# A01: Bruten Åtkomstkontroll - Reflektion

## Sårbarhet: PATCH & DELETE Utan Auktorisering

**Problem:** Vilken autentiserad användare kunde ändra/ta bort vilken anteckning som helst.

**Angreppsväg:**
```
Användare A loggar in → Får ID på Användare B:s anteckning (t.ex. 1001)
→ Skickar PATCH/DELETE /notes/1001
→ Anteckningen ändras/raderas utan ägarskaps-kontroll
```

**Konsekvens:**
- ❌ Data integritet: Andras anteckningar kan modifieras
- ❌ Data tillgänglighet: Andras anteckningar kan raderas
- ❌ GDPR-kränkning: Obehörig dataåtkomst

**Fix:**
```csharp
// Lägg till innan operation:
var user = BasicAuthenticationHandler.GetUserFrom(Request.Headers["Authorization"]);
if (note.Author != user.Username)
    return Forbid();  // 403 Forbidden
```

**Verifiering - Attacken Fungerar Inte Längre:**
```
FÖRE: PATCH /notes/1001 {"content":"Hacked"} → 200 OK (SADe)
EFTER: PATCH /notes/1001 {"content":"Hacked"} → 403 Forbidden (SÄKERT)
```

**Förebyggande:**
- ✅ Kontrollera auktorisering för VARJE endpoint
- ✅ Explicit ägarskaps-check före modifiering/borttagning
- ✅ Returnera 403 (inte 404) för obehörig åtkomst
- ✅ Använd samma mönster överallt

---

## Sårbarhet: GET List Endpoint - SQL Injection

**Problem:** Raw SQL med string interpolation - kunde kringgå auktorisering.

**Angreppsväg:**
```
GET /notes?containing=%' OR Author LIKE '%
→ SQL: WHERE Author='Test' AND Content LIKE '%%' OR Author LIKE '%%'
→ Returnerar ALLA anteckningar från ALLA användare
```

**Konsekvens:**
- ❌ Data läcka: Alla användares anteckningar exponeras
- ❌ Auktorisering kringgås helt
- ❌ Möjlig datamodifiering via SQL injection

**Fix:**
```csharp
// Använd LINQ istället för raw SQL:
return _database.Notes
    .Where(note => note.Author == user.Username)
    .Where(note => string.IsNullOrEmpty(containing) ? true : note.Content.Contains(containing))
    .OrderBy(note => note.Id)
    .ToArray();
```

**Verifiering - Attacken Fungerar Inte Längre:**
```
FÖRE: GET /notes?containing=%' OR Author LIKE '%
      → Returnerar ALL users' data (SÅRBAR)

EFTER: GET /notes?containing=%' OR Author LIKE '%
       → Returnerar endast egen data (SÄKER)
       → Söksträngen behandlas som literal text, inte SQL
```

**Förebyggande:**
- ✅ ALDRIG raw SQL med användarinmatning
- ✅ Använd LINQ/ORM för parametrisering
- ✅ EF Core hanterar escaping automatiskt
- ✅ Testa SQL injection-payloads i review

---

## Vad Jag Lärde Mig

**#1 - Server-Side Authorization is Everything**
Klient-sidan kan manipuleras. Enda sanningen är server-sidans kontroll.

**#2 - Konsistens är Kritisk**
GET-endpointen hade redan auktorisering, men PATCH/DELETE inte. Samma mönster måste tillämpas överallt.

**#3 - Ordet "Raw" Är en Varning**
`FromSqlRaw` = Riskfyllt. Använd alltid LINQ när användardata är inblandad.


