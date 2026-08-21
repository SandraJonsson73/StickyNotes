# A07: Cross-Site Scripting (XSS) - Reflektion

## Sårbarhet: innerHTML Med Användardata

**Problem:** Anteckningsinnehål renderas med `innerHTML` istället för `innerText`, vilket tillåter HTML/JavaScript-injektioner.

**Angreppsväg:**
```
Användare skapar anteckning med innehål:
  <svg/onload=alert('XSS')>
  eller
  <iframe src="javascript:alert('XSS');"></iframe>

Vid renderingen körs JavaScript i användarens webbläsare.
```

**Konsekvens:**
- ❌ Session hijacking: Angripare kan stjäla auth-token
- ❌ Data stöld: Läsa känslig data från sidan
- ❌ Malware distribution: Omdirigera till malicious sites
- ❌ Defacement: Ändra sidans innehål för andra användare

**Fix:**
```javascript
// FÖRE (sårbar):
content.innerHTML = note.content;  // Körs som HTML/JS

// EFTER (säker):
content.innerText = note.content;  // Renderas som text
```

**Verifiering - Attacken Fungerar Inte Längre:**
```
FÖRE: Skapa anteckning med <svg/onload=alert('XSS')>
      → JavaScript körs (SÅRBAR)

EFTER: Skapa anteckning med <svg/onload=alert('XSS')>
       → Rendereras som text: "<svg/onload=alert('XSS')>" (SÄKER)
       → Ingen JavaScript körs
```

**Förebyggande:**
- ✅ Använd ALDRIG `innerHTML` med användardata
- ✅ Använd `innerText` eller `textContent` för text
- ✅ Använd moderna frameworks (React, Angular) som escaping som standard
- ✅ Content Security Policy (CSP) headers för extra skydd
- ✅ Input validation - begränsa tillåtna tecken

---

## Vad Jag Lärde Mig

**#1 - innerHTML = Farligt För Användardata**
`innerHTML` tolkar allt som HTML, inklusive `<script>`, `<svg onload>`, `<iframe>` osv.

**#2 - innerText = Det Säkra Valet**
`innerText` behandlar allt som bokstavlig text. Speciella tecken visas som de är.

**#3 - XSS Är Svårt Att Detektera**
Ett missat `innerHTML`-samtal kan exponera miljontals användare. Det kräver systematisk kodgranskning.
