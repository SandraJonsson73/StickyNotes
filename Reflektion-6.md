# A06/A03: Vulnerable and Outdated Components - Reflektion

## Sårbarhet: Typosquatting - Falska NuGet-Paket

**Problem:** Två falska paket med stavningsfel (Swashbukle istället för Swashbuckle) är installerade med gamla versioner.

**Angreppsväg:**
```
1. Utvecklare installerar paket via Copy-Paste eller automatisk completion
2. Stavningsfel gör att fake-paket installeras istället för riktigt paket
3. Fake-paket injicerar malicious kod i System.Linq-namespace
4. Kod körs automatiskt i alla filer som använder System.Linq
5. Sårbarhet aktiveras via environment variable ENABLE_VULNERABILITY=true
```

**Konsekvens:**
- ❌ Arbitrary code execution i build-processen
- ❌ Supply chain compromise - alla användare påverkas
- ❌ Svårt att detektera (finns inte i källa)
- ❌ Kan exportera känslig data, installera backdoors, etc.

**Fix:**
```xml
<!-- FÖRE (sårbar - stavningsfel):-->
<PackageReference Include="Swashbukle.AspNetCore" Version="2.0.0" />
<PackageReference Include="Swashbukle.AspNetCore.Newtonsoft" Version="2.0.0" />

<!-- EFTER (säker - rätt namn + nyare version):-->
<PackageReference Include="Swashbuckle.AspNetCore" Version="6.4.0" />
<PackageReference Include="Swashbuckle.AspNetCore.Newtonsoft" Version="6.4.0" />
```

**Verifiering - Attacken Fungerar Inte Längre:**
```
FÖRE: dotnet run med ENABLE_VULNERABILITY=true
      → Fake FakeEnumerable.ToArray() körs
      → "VULNERABILITY ENABLED" skrivs ut (SÅRBAR)

EFTER: dotnet run med ENABLE_VULNERABILITY=true
       → Äkta Swashbuckle.AspNetCore används
       → Ingen injection av malicious code (SÄKER)
```

**Förebyggande:**
- ✅ ALDRIG Copy-Paste paketnamn - använd officiell dokumentation
- ✅ Kontrollera paketnamn dubbelt innan installation
- ✅ Använd Dependabot för att söka efter kända vulnerabilities
- ✅ Lås paketversioner för reproducerbar builds
- ✅ Auditera dependencies regelbundet med `dotnet list package --vulnerable`
- ✅ Använd NuGet package source mapping för extra kontroll

---

## Vad Jag Lärde Mig

**#1 - Supply Chain Är En Attack Vector**
Paket är kod som vi litar på helt. En falsk version kan kompromissa allting.

**#2 - Stavningsfel Är Farligt**
Typosquatting exploaterar vår slapphet med stavningen. Ett tecken skiljer mellan säker och komprometterad kod.

**#3 - Namespace Masquerading**
Falskt paket som pretenderar att vara `System.Linq` körs överallt automatiskt - extremt farligt!
