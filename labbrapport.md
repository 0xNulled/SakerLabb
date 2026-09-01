# Labbrapport: praktisk laboration

*Kunskapskontroll 2, IT-säkerhet för utvecklare. Fyll i mallen och lämna in som PDF tillsammans med länken till ditt repo. Riktlängd två till tre sidor.*

**Namn:**
**Datum:**  31/8/2026
**Repo (länk till din fork):** https://github.com/Nopie-Lupus/SakerLabb
**Applikation som analyserades:** SakerLabb support

---

## 1. Kort om applikationen och analysen

Beskriv i några meningar vilken app du analyserade, vad den gör och hur du genomförde analysen. Ange vilka verktyg du använde och hur du körde dem (CodeQL default setup med språk C#, ZAP passiv och aktiv skanning mot vilken adress).

*Skriv här.*

Sakerlabb Support är en hemsida för interna diskutioner och organisering kring Sakerlabbs arbete. [][][]

Analysen skedde med två verktyg. 
Först användes Githubs CodeQL för att göra en statisk analys (SAST) av kodbasen. Detta behövdes endast göras en gång då programmet har monolitisk design. CodeQL analysen kördes med default setup och språk satt till C#. 
Andra verktyget var Checkmarx’s ZAP (Zed Attack Proxy), ett dynamiskt analys verktyg (DAST). Processen med detta verktyg började med att göra en manuell utforskning av sidan i ett fönster från ZAP för att lära applikationen hur att interagera med hemsidan, samt vart den borde leta efter svagheter, efter det kördes en automatisk skanning. Allt detta kördes på en lokal instans av programmet, både ZAP (localhost:8085) och instansen av Sakerlabb (localhost:5080).

---

## 2. Fem fynd

Fyll i tabellen. Minst ett fynd ska komma från statisk analys (CodeQL) och minst ett från dynamisk analys (ZAP). Spara bevis i form av skärmbild eller rapportutdrag och hänvisa till det per fynd.

| Nr | Källa (CodeQL/ZAP) | Regel-id eller alert | Allvarlighet (+ confidence för ZAP) | Fil och rad eller URL | Verkligt eller falskt positivt | Motivering (2–4 meningar) |
|----|--------------------|----------------------|-------------------------------------|-----------------------|--------------------------------|---------------------------|
| 1 | CodeQL | cs/xml/insecure-dtd-handeling | Critical | SakerLabb.Web/Services/ImportService.cs: 22-23 | Verkligt | Vem som helst som har tillgång till importen för partnersystemen kan läsa dem filer och processer som serverprocessen kan, vilket inte behövs |
| 2 | CodeQL | cs/sql-injection | High | SakerLabb.Web/Data/UserRepository: 38 | Verkligt | En login page kräver inte autentisering då det är vart du skaffar den, du borde inte lita på input från användare |
| 3 | CodeQL | cs/cleartext-storage-of-sensitive-information | High | SakerLabb.Web/Data/UserRepository: 28 | Verkligt |  |
| 4 | ZAP | Directory Browsing | Medium (Medium) | Localhost:5080/files | Verkligt |  |
| 5 | ZAP | Cross Site Scripting (Reflected) | High (Medium) | Localhost:5080/login | Verkligt | Skickar du inloggningslänken som har försökt som användarnamn ett js-script körs det. Den största ledtråden är att länken ser konstig ut. Även om du inte gjort inloggningsförsöket körs koden hos dig om du öppnar länken |

Bevis (skärmbilder eller utdrag), numrerade efter fyndet ovan:

*Klistra in här, eller hänvisa till bilagor.*
Fynd 1:
Ze-photo's/XXE-Example.png
Ze-photo's/XXE-Example-Run.png

Fynd 2:
/home/nopie/Projects/repositories/school/SakerLabb/Ze-photo's/SQL-if-Wrong-Username.png
/home/nopie/Projects/repositories/school/SakerLabb/Ze-photo's/SQL-if-injection.png
/home/nopie/Projects/repositories/school/SakerLabb/Ze-photo's/SQL-after-successful-injection.png 
(Bryter karaktär lite snabbt: jag vet att autentiseringen är helt busted, men det kom inte up på varken av de två skanningarna så för att visa bättre mina resonemang senare låtsas jag att att autentiseringen fungerar alls)

Fynd 5:
Ze-photo's/Bevis-ZAP-XSS-Reflected-pop-up.png
(lösenordet ses inte efter ett misslyckat inloggningsförsök)
---

## 3. Prioritering

Rangordna fynden och motivera ordningen med allvarlighetsgrad, exponering och utnyttjbarhet. Vilket tar du först och varför?

*Skriv här.*

Rangordningen av fynden blir fynd 2, fynd 1, fynd 5, fynd 3, fynd 4. 

Fynd 2 är av högsta prioritet, den har hög allvarlighetsgrad för en anledning då arbiträr användargiven sql kan köras direkt mot databasen. Exponeringsfaktorn blir värre med tanke på att en av sätten att utnyttja denna attackvektor är inloggningssidan, som måste förbli tillgänglig för icke-autentiserade användare då detta är autentiseringsmetoder. Allt som behövs från en attackerade är antingen att besöka sidan med en webbläsare eller att skicka ett rent api anrop med en injektion som kan modifiera, lägga till, eller ta bort sql data. 

Fynd 1 ses av CodeQL att vara av kritisk allvarlighetsgrad då arbiträr xml kan avläsas och köras vilket kan bland annat leda till läckage av filers innehåll (exempel /etc/passwd). Dock kräver detta tillgång till webbsidan för import av kunders detaljer[][][]. Detta är inte en sida som är menad för vem som helst att kunna nå olikt inloggningssidan för fynd 2. Därmed är exponeringen mindre då någon som vill utnyttja denna sårbarhet måste först få tillgång till en sida som inte är menad att vara fullt publik. 

Fynd 5 ses av ZAP som hög allvarlighet och har starkt negativa implikationer men är inte lika enkelt att exploatera. Genom att göra ett misslyckat inlogg skickas det användarnamn som försöktes användas tillbaka som markdown text (vilket tillåter exekvering av Javascript) och sparas i url:en. Därmed kan du skicka det misslyckade inloggningsförsöket som innehåller (gissningsvis obfuskerad) javascript i url:en och det kommer exekveras i recipientens webbläsare. 

Fynd 3 är av stort allvar, då dem med tillgång till servern kan se i klartext alla inlogg som sker. Dock behöver du tillgång till servern och/eller programmets terminal output. Dessa saker borde inte loggas alls, men för att just detta ska exploateras behövs redan relativt djup tillgång. Därmed är dem lägre prioriterade än fynden över då de antingen ger djupare tillgång eller tillåter arbiträr xml inläsning. 

Fynd 4 är av allvarlighetsgrad medium för bra anledning. För att detta ska bli ett säkerhetsproblem krävs det att filer där du inte vill ha granulär kontroll över vem som kan se vad, en all or nothing. Detta fungerar ibland, och kanske i vissa fal kan vara önskeakt, men helst borde tillgång till filer sättas per fil istället för en hel mapp. Det är också någonting som borde kräva inloggning för att komma till den delen så förhoppningsvis ser endast vissa få denna sida.

---

## 4. Åtgärder (minst tre)

Använd mönstret nedan per åtgärdat fynd. Varje åtgärd ska gå att spåra tillbaka till ett fynd i tabellen ovan, och beviset efter ska vara en **ny körning av verktyget**, inte din egen kod.
(Lyckades inte få verktygen att köra igen som bevis, så använder bilder som substitut)
### Åtgärd 1

```
Fynd: Fynd 1 - cs/xml/insecure-dtd-handeling
Plats: SakerLabb.Web/Services/ImportService.cs: 22-23
Bevis före:  Ze-photo's/XXE-Example.png Ze-photo's/XXE-Example-Run.png
Bedömning:   Verkligt. Koden explicit tillåter läsning av externa resurser och DTD hantering och input blir inte sanitizerad under processen.
Åtgärd:      Ändrade så att DTD objekt och hantering av externa resurser genom Xmlresolver returerar null. Inte strikt nödvändigt för fixen men satte en try catch runtom så om någon försöker får dem ett vagt error message istället för en stacktrace (cfe4a8aa90c1737859f980df844930906c6b2ab8)
Bevis efter: Ze-photo's/XXE-After-Fix.png
```

### Åtgärd 2

```
Fynd: Fynd 2 - cs/sql-injection
Plats: SakerLabb.Web/Data/UserRepository: 38
Bevis före: Ze-photo's/SQL-if-injection.png Ze-photo's/SQL-after-successful-injection.png
Bedömning: Verklig. Username input konkatineras utan sanitisering och därmed blir del av kommandot
Åtgärd: Istället för att konkatinera Username input som en sträng i SQL kommandot lägger vi till en variabel som får värdet av det inputet, därmed parametrisera inputen. (2488f77e79f01c35aff023180d7866d3dd775bf6)
Bevis efter: Ze-photo's/SQL-after-fix.png
```

### Åtgärd 3

```
Fynd: Fynd 5 - Cross Site Scripting (Reflected)
Plats: Localhost:5080/login
Bevis före: Ze-photo's/Bevis-ZAP-XSS-Reflected-pop-up.png
Bedömning: Verkligt. Användarnamn renderas som markdown på klientens websida utan sanitisering.
Åtgärd: Få inte det försökta användarnamnet att renderas som markdown, vilket tillåter js (f4bfc54db171cf229b27c6c751b76c2be22f2580)
Bevis efter: Ze-photo's/Bevis-ZAP-NoXSS.png
```

---

## 5. Eventuella bortval

Om du valt att inte åtgärda ett fynd, skriv ned tre saker per bortval: risken, motivet och den kompenserande kontrollen. Sätt gärna ett datum för omprövning.

*Skriv här, eller skriv "inga bortval".*

Fynd 4
Risk: Om någon har tillgång till mappen som filläsare kan de läsa allting inuti, varesig allt av det var menat för läsaren eller ej. Alla filer blir en ”all or nothing” berräottigheter för användaren. 
Motiv: Har inte tidsbudget att lösa detta, samt är dessa problem både mindre exponerade än dem fixade problemen. Du behöver redan vara behörigas för denna sida samt. 
Fynd 4 ger potentiellt tillgång till filer till personer som inte ska ha det, men dem andra problemen 

Fynd 3
Risk: Dem som har tillgång till serverloggen kan se användarnamn samt lösenord av alla inloggade personer sedan servern senast startade om
Motiv: gränslad tidsbudget, samt är detta problem mindre exponerat och dem andra fynden hjälper dig att komma in längre in i systemet, detta ger dig tillgång till konto credentials om du redan har shell access. 