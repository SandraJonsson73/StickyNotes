# A02/A04: Cryptographic Failures - Reflektion

## Sårbarhet: Hemliga Uppgifter I Källkod

**Problem:** Känsliga data (lösenord, flaggor) hardkodade direkt i appsettings.json som ligger i version control.

**Angreppsväg:**
```
Angripare klona GitHub-repo
→ Läser appsettings.json
→ Hittar TestUserPassword och hemliga GUIDs
→ Kan logga in och komma åt känslig konfiguration
```

**Konsekvens:**
- ❌ Direkta inloggningsuppgifter exponerade
- ❌ Test-flaggor publicerade (underminer övningen)
- ❌ Möjlighet till lateral movement i system
- ❌ Compliance-kränkning (GDPR, PCI DSS)

**Fix:**
```
FÖRE:
appsettings.json innehåller: TestUserPassword: "123"
→ Ligger i version control → Alla kan se det

EFTER:
appsettings.json innehåller: TestUserPassword: "CONFIGURE_IN_APPSETTINGS_DEVELOPMENT_JSON"
appsettings.Development.json innehåller: TestUserPassword: "123"
→ .Development.json ligger i .gitignore → Hemligt
```

**Verifiering - Hemliga Uppgifter Är Skyddade:**
```
FÖRE: git clone → Läser appsettings.json → TestUserPassword synlig
      Lösenordet är exponerat (SÅRBAR)

EFTER: git clone → Läser appsettings.json → TestUserPassword = "CONFIGURE_IN_..."
       Faktiskt lösenord ligger i appsettings.Development.json (GITIGNORE)
       Lösenordet är skyddat (SÄKER)
```

**Förebyggande:**
- ✅ ALDRIG hemliga uppgifter i version control
- ✅ Använd environment-specific config-filer (.Development.json)
- ✅ Lägg .Development.json i .gitignore
- ✅ Dokumentera hur man konfigurerar secrets lokalt
- ✅ Använd Secret Manager för development
- ✅ Använd Azure Key Vault/AWS Secrets Manager för production

**Implementeringsdetaljer:**
- SeedData: false i Development (förhindrar database lock vid varje start)
- Hemliga GUIDs lagras i .Development.json (gitignored)
- Placeholder-GUIDs i appsettings.json (säker att commita)

---

## Vad Jag Lärde Mig

**#1 - Hemlighetshållning = Arkitektur-nivå**
Secrets kan inte "fixas" senare - måste byggas in från början.

**#2 - .gitignore Är Först Försvarslinjen**
En enda glömd fil i .gitignore exponerar allt.

**#3 - Configuration Hierarki**
appsettings.json < appsettings.Development.json < Environment Variables < Secret Manager
