# A03/A05: SQL Injection - Reflektion

## Sårbarhet: GET List Endpoint Med Raw SQL

**Problem:** Användarinmatning direktinsatt i raw SQL via string interpolation.

**Angreppsväg:**
```
GET /notes?containing=%' OR Author LIKE '%
→ Raw SQL: WHERE Author='Test' AND Content LIKE '%%' OR Author LIKE '%%'
→ Returnerar ALLA anteckningar från ALLA användare
```

**Konsekvens:**
- ❌ Data läcka: Alla användares anteckningar exponeras
- ❌ Auktorisering kringgås helt
- ❌ Möjlig datamodifiering via `DROP TABLE` eller `DELETE`

**Fix:**
```csharp
// Byt från raw SQL till parameteriserad LINQ:
return _database.Notes
    .Where(note => note.Author == user.Username)
    .Where(note => string.IsNullOrEmpty(containing) ? true : note.Content.Contains(containing))
    .OrderBy(note => note.Id)
    .ToArray();
```

**Verifiering - Attacken Fungerar Inte Längre:**
```
FÖRE: GET /notes?containing=%' OR Author LIKE '%
      → SQL: WHERE ... Content LIKE '%%' OR Author LIKE '%%'
      → Returnerar ALLA data (SÅRBAR)

EFTER: GET /notes?containing=%' OR Author LIKE '%
       → EF Core parameteriserar: @p0 = "%' OR Author LIKE '%"
       → Returnerar endast egen data (SÄKER)
```

**Förebyggande:**
- ✅ ALDRIG raw SQL med användarinmatning
- ✅ Använd LINQ för automatisk parametrisering
- ✅ EF Core hanterar escaping automatiskt
- ✅ Testa SQL injection-payloads i code review

---

## Vad Jag Lärde Mig

**#1 - Raw SQL = Riskfyllt**
`FromSqlRaw` är aldrig vägen för användardata. Ordet "Raw" är en varning.

**#2 - Parametrisering Är Nyckelorden**
SQL injection fungerar bara om användardata blir SQL-kod. Parametrisering gör det omöjligt.

**#3 - Säkerhetsfixar Är Ofta Överlappande**
A01-fixningen (auktorisering) + A03-fixningen (SQL injection) går hand i hand i samma endpoint.
