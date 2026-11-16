# Skripta za prvi kolokvijum – .NET, EF Core, LINQ, veza 1-N

Zasnovano na: `Vezbe 06/Prvi kolokvijum - priprema (veza 1-N).doc` (zadatak), `Vezbe 06/Bioskop-Dopunjeno` (gotovo resenje, Minimal API) i `Vezbe 05` (kontroleri, DI, Code First).

> **Kljucna ideja:** kolokvijum je ISTI zadatak sa drugim imenima. Bioskop = **Film (1) → Projekcija (N)**.
> Dobices npr. Autor→Knjiga, Doktor→Pregled, Kurs→Cas... Funkcionalnosti su uvek istih 4 tipa. Nauci sablon, menjaj imena.

---

## 0. Mapa zadatka (50 poena) – sta se uvek trazi

| # | Zahtev | Poeni | Sablon |
|---|--------|-------|--------|
| 0 | 3 projekta (Domain, Infrastructure, Web) + DbContext u DI | 3 | Sekcija 1 |
| 1 | Model + migracija `Init` (samo "jaki" entitet), pa nova migracija sa 1-N | 7 | Sekcija 2 |
| 2 | **GET lista + pretraga** (0, 1 ili vise kriterijuma) + broj dece | 8 | Sekcija 4.1 |
| 3 | **POST jaki entitet** – 400 ako duplikat, 201 Created | 5 | Sekcija 4.2 |
| 4 | **GET po id** sa decom, sortirano; 404 | 5 | Sekcija 4.3 |
| 5 | **POST vise dece odjednom** – 404, validacije, duplikati u zahtevu, zauzet termin u bazi (ProblemDetails 400), sve-ili-nista, 201 | 14 | Sekcija 4.4 |
| 6 | `.http` fajl sa uspesnim i neuspesnim zahtevima | 5 | Sekcija 6 |
| 7 | Middleware: ispisi metodu, putanju i status kod | 3 | Sekcija 5 |

**Najvise poena nosi tacka 5 (14p) – nju vezbaj najvise.**

---

## 1. Postavljanje resenja (3 projekta) – Visual Studio 2026

### Korak po korak

**1) Prazno resenje**
`File → New → Project` → trazi **Blank Solution** (Prazno resenje) → ime `Priprema` → Create.

**2) Domain i Infrastructure (Class Library)**
Desni klik na **Solution** u Solution Explorer-u → `Add → New Project` → **Class Library** (C#, `.NET`, NE `.NET Framework`) → ime `Domain` → Next → Framework: **.NET 10** → Create.
Isto ponovi za `Infrastructure`.

**3) Web projekat** – `Add → New Project`, zavisi sta traze:
- **Minimal API** → sablon **ASP.NET Core Empty** *ili* **ASP.NET Core Web API** sa **ODCEKIRANIM** "Use controllers" (ostavi "Enable OpenAPI support" cekirano, kao u vezbama). Ime npr. `MinimalApi`.
- **Kontroleri** → sablon **ASP.NET Core Web API** sa **CEKIRANIM** "Use controllers". Ime npr. `RestWebApi`.

Ostalo: Framework .NET 10, Authentication None, Configure for HTTPS cekirano. Docker i "top-level statements" – ostavi podrazumevano / bez Dockera.

**4) Reference izmedju projekata**
Desni klik na projekat → `Add → Project Reference...` → cekiraj:
| Projekat | Dodaj reference na |
|---|---|
| Infrastructure | Domain |
| Web (MinimalApi/RestWebApi) | Domain **i** Infrastructure |
| Domain | nista |

(Strelice: `Web → Infrastructure → Domain`, `Web → Domain`.)

**5) NuGet paketi**
Desni klik na projekat → `Manage NuGet Packages...` → tab **Browse** → trazi i `Install`:
| Projekat | Paket |
|---|---|
| **Infrastructure** | `Microsoft.EntityFrameworkCore.SqlServer` |
| **Infrastructure** | `Microsoft.EntityFrameworkCore.Tools` |
| **Web projekat** | `Microsoft.EntityFrameworkCore.Design` |

Verzija: ista **10.x** za sve EF pakete (npr. 10.0.x – kao u `Bioskop-Dopunjeno`). Ne mesaj verzije.

**6) Startup projekat**
Desni klik na **Web projekat → Set as Startup Project** (bice podebljan). Bez ovoga migracije ne rade.

**7) Package Manager Console (za migracije)**
`Tools → NuGet Package Manager → Package Manager Console`.
U gornjem desnom delu konzole, dropdown **Default project** → izaberi **Infrastructure** (ovde ce se generisati `Migrations` folder). Startup projekat je vec Web (korak 6).
Komande se pisu u konzoli (vidi Sekciju 2).

**8) Pokretanje i testiranje**
- `F5` / zeleno dugme, izaberi profil **https**. Port pise u `Properties/launchSettings.json` (`applicationUrl`) – on ide u `.http` fajl.
- `.http` fajl otvori dvoklikom u VS; iznad svakog zahteva je **Send request**. (`View → Other Windows → Endpoints Explorer` takodje nudi listu ruta.)
- Konzola sa ispisom iz middleware-a: prozor **Output** ili otvoreni konzolni prozor aplikacije.
- Aplikacija mora da radi (F5) da bi `.http` zahtev prosao.

### Registracija konteksta u DI (3p) – Program.cs
```csharp
builder.Services.AddDbContext<BioskopContext>();   // Scoped po zahtevu (Vezbe 05: AddScoped<ProductsContext>() radi isto)
```
Za Minimal API dodaj i `builder.Services.AddProblemDetails();` + `app.UseStatusCodePages();`.
`using Infrastructure;` na vrhu Program.cs (ili `Alt+Enter` na crvenom imenu → "using ...").

> DI zivotni vek (Vezbe 05, pitanje za usmeni): **Transient** = novi svaki put kad se trazi; **Scoped** = jedan po HTTP zahtevu (DbContext!); **Singleton** = jedan za celu aplikaciju.

### (Alternativa za brze) CLI – isto to, jednim copy-paste
```bash
dotnet new sln -n Priprema
dotnet new classlib -n Domain
dotnet new classlib -n Infrastructure
dotnet new web -n MinimalApi            # ili: dotnet new webapi --use-controllers -n RestWebApi
dotnet sln add Domain Infrastructure MinimalApi
dotnet add Infrastructure reference Domain
dotnet add MinimalApi reference Domain Infrastructure
dotnet add Infrastructure package Microsoft.EntityFrameworkCore.SqlServer
dotnet add Infrastructure package Microsoft.EntityFrameworkCore.Tools
dotnet add MinimalApi package Microsoft.EntityFrameworkCore.Design
```

---

## 2. Model i migracije (7p)

### 2.1 KORAK 1 – Samo Film + `Init` (3p)

`Domain/Film.cs` (na ovom koraku **BEZ** `Projekcije` liste i bez Projekcija klase!)
```csharp
namespace Domain;

public class Film
{
    public int Id { get; set; }
    public string Naziv { get; set; }
    public string Zanr { get; set; }
    public int GodinaSnimanja { get; set; }
    public int TrajanjeMin { get; set; }
}
```

`Infrastructure/BioskopContext.cs`
```csharp
using Domain;
using Microsoft.EntityFrameworkCore;

namespace Infrastructure;

public class BioskopContext : DbContext
{
    public DbSet<Film> Filmovi { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseSqlServer(@"Data Source=(localdb)\mssqllocaldb; Initial Catalog=PPP; Integrated security=True;");
    }
}
```
(Connection string ISTI kao na vezbama; menja se samo `Initial Catalog`.)

U **Package Manager Console** (Default project = Infrastructure):
```powershell
Add-Migration Init
Update-Database
```
(CLI ekvivalent: `dotnet ef migrations add Init` pa `dotnet ef database update`)

### 2.2 KORAK 2 – Dodaj Projekciju + vezu 1-N + nova migracija (4p)

`Domain/Film.cs` – dodaj navigaciju ka N strani:
```csharp
public List<Projekcija> Projekcije { get; set; }
```

`Domain/Projekcija.cs`
```csharp
namespace Domain;

public class Projekcija
{
    public int BrojProjekcije { get; set; }          // PK (ne zove se Id → mora se konfigurisati!)
    public DateTime DatumVremePocetka { get; set; }
    public int BrojSale { get; set; }
    public decimal CenaKarte { get; set; }

    public int FilmId { get; set; }                  // strani kljuc
    public Film Film { get; set; }                   // navigacija ka "1" strani
}
```

U kontekstu dodaj `DbSet<Projekcija> Projekcije` i Fluent API:
```csharp
public DbSet<Projekcija> Projekcije { get; set; }

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Projekcija>().HasKey(p => p.BrojProjekcije);   // samo ako PK nije "Id"/"<Klasa>Id"

    modelBuilder.Entity<Projekcija>()
        .HasOne(p => p.Film)            // projekcija ima JEDAN film
        .WithMany(f => f.Projekcije)    // film ima VISE projekcija
        .HasForeignKey(p => p.FilmId);  // FK kolona
}
```
```powershell
Add-Migration DodavanjeProjekcija
Update-Database
```

**Kako citati Fluent API:** `Entity<DETE>().HasOne(dete => dete.Roditelj).WithMany(roditelj => roditelj.ListaDece).HasForeignKey(dete => dete.RoditeljId)`.
Uvek pocni od **deteta (N strane)**.

**Pravila konvencije EF-a (da ne gubis vreme):**
- PK se pronalazi automatski samo ako se zove `Id` ili `<NazivKlase>Id`. Inace `HasKey(...)` ili `[Key]`.
- Ako vec imas `FilmId` + `Film`, EF sam skonta vezu; Fluent API ipak napisi (vezbe ga traze i sigurno je).
- Ako PK **nije** auto-increment (npr. `CategoryCode` u Vezbe 05): `.Property(x => x.Kod).ValueGeneratedNever()`.
- `decimal` daje warning o preciznosti – ignorisi, ili `.HasPrecision(18,2)`.

### 2.3 Primarni kljuc: AUTO-INKREMENT ili NE (cesto se eksplicitno trazi u zadatku!)

U zadatku pise npr. *"sifra filma se automatski generise"* ili *"sifru filma unosi korisnik"*. To menja **model**, **migraciju** i **POST endpoint**.

#### A) Auto-inkrement (podrazumevano za `int`)
Svaki `int` PK je u SQL Server-u `IDENTITY(1,1)` po konvenciji – ne moras nista da pises. Ako zadatak izricito trazi, mozes eksplicitno:
```csharp
modelBuilder.Entity<Film>().Property(f => f.Id).ValueGeneratedOnAdd();
// ili atributom na klasi:  [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
```
- **Klijent NE salje id** u JSON-u. Posle `SaveChanges()` EF popuni `noviFilm.Id`.
- POST proverava samo poslovni duplikat (naziv + godina), ne id.
- `Created($"/filmovi/{noviFilm.Id}", noviFilm)` – id je tek posle SaveChanges poznat.

#### B) NIJE auto-inkrement (id unosi korisnik)
```csharp
modelBuilder.Entity<Film>().HasKey(f => f.Id);                    // ako se ne zove Id / <Klasa>Id
modelBuilder.Entity<Film>().Property(f => f.Id).ValueGeneratedNever();
// ili atributom na klasi:  [DatabaseGenerated(DatabaseGeneratedOption.None)]
```
(Ovo je tvoj primer iz Vezbe 05: `ProductCategory.CategoryCode` sa `.ValueGeneratedNever()`.)

- **Klijent MORA da posalje id** u JSON-u. Ako ga ne posalje, stize `0` – to je losa vrednost, vrati 400.
- POST **mora** da proveri da li id vec postoji, inace SQL baci `PRIMARY KEY violation` → 500:
```csharp
app.MapPost("/filmovi", (Film novi, BioskopContext context) =>
{
    if (novi.Id <= 0)
        return Results.Problem(title: "Film nije validan!", detail: "Sifra filma je obavezna", statusCode: 400);

    if (context.Filmovi.Any(f => f.Id == novi.Id))
        return Results.Problem(title: "Film nije validan!", detail: $"Film sa sifrom {novi.Id} vec postoji", statusCode: 400);

    // + ostala pravila (naziv + godina)
    context.Filmovi.Add(novi);
    context.SaveChanges();
    return Results.Created($"/filmovi/{novi.Id}", novi);
});
```
- Ako je i **dete** (Projekcija) bez auto-inkrementa, pri unosu liste dodaj jos dve provere:
  1. **ponavljanje id-a u zahtevu:** `projekcije.GroupBy(p => p.BrojProjekcije).Where(g => g.Count() > 1)`
  2. **id vec postoji u bazi:** `context.Projekcije.Any(x => x.BrojProjekcije == p.BrojProjekcije)`
  
  Obe idu u istu listu `greske` (sve-ili-nista).
- `.http` zahtev mora da sadrzi `"id": 10` (odnosno `"brojProjekcije": 10`) i test za duplikat id-a.
- U PUT-u (izmena) id se **ne menja** – uzima se iz rute (`/filmovi/{id}`), a iz tela samo ostala polja.

#### Brzi podsetnik
| Zahtev | Konfiguracija | Klijent salje id? | Dodatna provera u POST-u |
|---|---|---|---|
| auto-inkrement | nista / `ValueGeneratedOnAdd()` | NE | nista |
| NIJE auto-inkrement | `ValueGeneratedNever()` | DA | id > 0 i id ne postoji (+ duplikat id-a u zahtevu za listu dece) |
| PK se ne zove `Id`/`<Klasa>Id` | `HasKey(...)` / `[Key]` | zavisi od gornjeg | – |
| slozeni kljuc | `HasKey(x => new { x.A, x.B })` | DA | provera po oba polja |
| `Guid` PK | nista | NE (EF generise) | – |
| `string` PK | `ValueGeneratedNever()` implicitno | DA | provera postojanja |

#### VAZNO oko migracija
- **Odluci PRE `Init` migracije.** SQL Server ne moze da pretvori postojecu kolonu u/iz `IDENTITY` – ako se predomislis posle `Update-Database`, migracija pukne.
- Ako si vec uradio `Update-Database`, a treba promena: `Remove-Migration` ne moze (vec primenjena) → vrati bazu: `Update-Database <ImePrethodneMigracije>` pa `Remove-Migration`; ili jednostavno **obrisi bazu** (`Drop-Database`) + obrisi `Migrations` folder i uradi Init ispocetka. Na kolokvijumu je to cesto najbrze.
- Ako PK ne postoji ni po konvenciji ni po konfiguraciji: greska *"entity type requires a primary key"* → dodaj `HasKey`.

---

### 2.4 (Opciono) Seed podaci – `Bioskop-Dopunjeno` ga ima, na kolokvijumu verovatno NE traze
```csharp
modelBuilder.Entity<Film>().HasData(
    new Film { Id = 1, Naziv = "F1", Zanr = "Horor", GodinaSnimanja = 2025, TrajanjeMin = 60 });
modelBuilder.Entity<Projekcija>().HasData(
    new Projekcija { BrojProjekcije = 100, BrojSale = 1, CenaKarte = 222.22m,
                     DatumVremePocetka = new DateTime(2026, 11, 12, 20, 0, 0), FilmId = 1 });
```
`HasData` **zahteva eksplicitne PK i FK vrednosti**. Posle toga nova migracija (`SeedPodataka`).

### Sve komande za migracije – tabela (Package Manager Console ↔ dotnet CLI)

| Sta radis | Package Manager Console (Visual Studio) | dotnet CLI (terminal) |
|---|---|---|
| **Pre svega: podesavanje** | Default project = **Infrastructure**, Startup = web projekat (Set as Startup Project) | Jednom: `dotnet tool install --global dotnet-ef` (provera: `dotnet ef --version`). Terminal otvori u folderu **Infrastructure** (tu se prave migracije) |
| **Nova migracija** | `Add-Migration Init` | `dotnet ef migrations add Init` |
| **Primeni na bazu (do poslednje migracije)** | `Update-Database` | `dotnet ef database update` |
| **Primeni / vrati do odredjene migracije** | `Update-Database DodavanjeProjekcija` | `dotnet ef database update DodavanjeProjekcija` |
| **Vrati bazu na prazno (poponisti sve migracije)** | `Update-Database 0` | `dotnet ef database update 0` |
| **Obrisi poslednju migraciju** (samo ako NIJE primenjena na bazu) | `Remove-Migration` | `dotnet ef migrations remove` |
| **Spisak migracija** (koje su primenjene) | `Get-Migration` | `dotnet ef migrations list` |
| **Obrisi celu bazu** | `Drop-Database` | `dotnet ef database drop --force` |
| **Generisi SQL skriptu migracija** | `Script-Migration` | `dotnet ef migrations script` |
| **Info o kontekstu (provera da radi konekcija)** | `Get-DbContext` | `dotnet ef dbcontext info` |

**Napomene**
- Ime migracije je proizvoljno, ali u zadatku obicno stoji tacno ime (**`Init`**), pa ga ukucaj bas tako.
- Ako CLI javi da ne moze da napravi DbContext, dodaj web projekat kao startup: `-s ../MinimalApi` (npr. `dotnet ef migrations add Init -s ../MinimalApi`).
- Standardni tok na kolokvijumu: `Add-Migration Init` → `Update-Database` → izmeni model → `Add-Migration <Ime>` → `Update-Database`.
- Ako zaboravis `Update-Database`, model i baza se ne poklapaju (greska `Invalid object name` / `Invalid column name`).
- Ako migracija puca i hoces cist pocetak: `Drop-Database`, obrisi folder `Migrations`, pa iznova `Add-Migration Init` + `Update-Database`.

---

### Ako migracija pogresi
- Obrisi poslednju: `Remove-Migration` (samo ako nije primenjena na bazu).
- "Unable to create DbContext" → fali Design paket u web projektu, ili Web projekat nije Set as Startup Project, ili je Default project u konzoli pogresan (mora Infrastructure).

---

## 3. LINQ i EF – cheat sheet (sve sto ti treba)

```csharp
var q = context.Filmovi.AsQueryable();      // ODLOZENO izvrsavanje – upit ide u bazu tek na ToList/foreach/Count/Any/First...

q.Where(f => f.Naziv.Contains("x"))         // LIKE %x%
q.Where(f => f.Naziv.StartsWith("x"))       // LIKE x%
q.OrderBy(f => f.Naziv)  /  .OrderByDescending(...)  /  .ThenBy(...)
q.Select(f => new { f.Naziv, f.Zanr, Broj = f.Projekcije.Count })   // projekcija u anonimni objekat
q.Any(f => f.Id == 5)                       // bool, da li postoji
q.Count() / q.Count(f => ...)
q.FirstOrDefault(f => f.Id == 5)            // null ako nema
q.SingleOrDefault(f => f.Id == 5)           // null ako nema, greska ako ih ima vise
context.Filmovi.Find(5)                     // trazi PO PK (prvo gleda cache konteksta)
q.Include(f => f.Projekcije)                // eager loading – ucitaj i decu
q.Include(f => f.Projekcije).ThenInclude(p => p.Nesto)
q.Sum(p => p.CenaKarte) / Average / Min / Max
q.Take(10) / q.Skip(10)
q.ToList()
```

**In-memory GroupBy (detekcija duplikata u zahtevu):**
```csharp
var duplikati = lista
    .GroupBy(p => new { p.BrojSale, p.DatumVremePocetka })
    .Where(g => g.Count() > 1);
foreach (var d in duplikati) ... d.Key.BrojSale ... d.Key.DatumVremePocetka
```

**Zamke koje se cesto desavaju:**
1. **Ciklicni JSON** – ako vratis `Film` sa `Projekcije`, a svaka `Projekcija` ima `Film` → beskonacna petlja. **Resenje: uvek vracaj anonimne objekte (`Select(new {...})`)**, nikad entitete sa navigacijama u oba smera.
2. `Include` je potreban da bi `film.Projekcije` bilo popunjeno. Alternativa: `context.Entry(film).Collection(f => f.Projekcije).Load()` (to radi `Bioskop-Dopunjeno`) ili `Select` sa `f.Projekcije.Count` (EF sam napravi JOIN/subquery – ne treba Include).
3. **Opcioni filteri** – `if (naziv is not null) q = q.Where(...)`. Gradi upit korak po korak, NE `StartsWith(null)` (to je bug u Vezbe 05 primeru).
4. Poredjenje datuma: `DateTime.Now` (trenutak), `DateTime.Now.Date` (ponoc danas). "U buducnosti" → `p.DatumVremePocetka <= DateTime.Now` je greska.
5. `SaveChanges()` **jednom na kraju** – on je transakcija: ili sve ili nista (to je "sve-ili-nista" zahtev).

---

## 4. Funkcionalnosti – Minimal API (`Bioskop-Dopunjeno/MinimalApi/Program.cs`)

### 4.1 GET /filmovi – prikaz + pretraga (8p)
Kriterijumi su **opcioni** query parametri → `string? naziv`, `string? zanr`.
```csharp
app.MapGet("/filmovi", (string? naziv, string? zanr, BioskopContext context) =>
{
    var q = context.Filmovi.AsQueryable();
    if (naziv is not null) q = q.Where(f => f.Naziv.Contains(naziv));
    if (zanr  is not null) q = q.Where(f => f.Zanr.Contains(zanr));

    return q.Select(f => new
    {
        f.Naziv,
        f.Zanr,
        f.GodinaSnimanja,
        UkupanBrojProjekcija = f.Projekcije.Count
    });
});
```
Test: `/filmovi`, `/filmovi?naziv=f`, `/filmovi?zanr=horor&naziv=f1`.

### 4.2 POST /filmovi – novi film (5p)
```csharp
app.MapPost("/filmovi", (Film noviFilm, BioskopContext context) =>
{
    if (context.Filmovi.Any(f => f.Naziv == noviFilm.Naziv && f.GodinaSnimanja == noviFilm.GodinaSnimanja))
        return Results.Problem(
            title: "Film nije validan!",
            detail: "Vec postoji film sa datim nazivom i godinom snimanja",
            statusCode: 400);                               // 400 Bad Request

    context.Filmovi.Add(noviFilm);
    context.SaveChanges();
    return Results.Created($"/filmovi/{noviFilm.Id}", noviFilm);   // 201 Created
});
```
(Moze i `Results.BadRequest("poruka")` – 400 je 400; `Problem` je lepsi.)

### 4.3 GET /filmovi/{id} – film sa projekcijama (5p)
```csharp
app.MapGet("/filmovi/{id}", (int id, BioskopContext context) =>
{
    var film = context.Filmovi.Include(f => f.Projekcije).FirstOrDefault(f => f.Id == id);
    if (film is null) return Results.NotFound();            // 404

    return Results.Ok(new
    {
        film.Id, film.Naziv, film.Zanr, film.GodinaSnimanja, film.TrajanjeMin,
        Projekcije = film.Projekcije
            .OrderBy(p => p.DatumVremePocetka)               // SORTIRANO po vremenu pocetka!
            .Select(p => new { p.DatumVremePocetka, p.BrojSale, p.CenaKarte })
    });
});
```
> U `Bioskop-Dopunjeno` ovde fali `OrderBy` – na kolokvijumu to je 4p, nemoj zaboraviti.

### 4.4 POST /filmovi/{id}/projekcije – unos vise projekcija (14p) ⭐ NAJVAZNIJE
Telo zahteva je **lista** (`List<Projekcija>`). Redosled provera:

1. film ne postoji → **404**
2. za svaku: vreme u buducnosti **i** cena > 0
3. duplikat (sala+vreme) **unutar zahteva**
4. zauzet termin **u bazi** (bilo koji film) → **400 ProblemDetails** sa naslovom i opisom zauzetih termina
5. ako ijedna greska → **nista se ne upisuje**; inace `AddRange` + jedan `SaveChanges` → **201**

```csharp
app.MapPost("/filmovi/{id}/projekcije", (int id, List<Projekcija> projekcije, BioskopContext context) =>
{
    var film = context.Filmovi.Find(id);
    if (film is null) return Results.NotFound();                       // 1

    var greske = new List<string>();

    // 2) vreme i cena
    foreach (var p in projekcije)
    {
        p.FilmId = film.Id;                                            // povezi sa filmom iz rute
        if (p.DatumVremePocetka <= DateTime.Now)
            greske.Add($"Projekcija u sali {p.BrojSale} ({p.DatumVremePocetka}) nije u buducnosti");
        if (p.CenaKarte <= 0)
            greske.Add($"Projekcija u sali {p.BrojSale} ({p.DatumVremePocetka}) mora imati cenu vecu od 0");
    }

    // 3) duplikati u zahtevu
    var duplikati = projekcije
        .GroupBy(p => new { p.BrojSale, p.DatumVremePocetka })
        .Where(g => g.Count() > 1);
    foreach (var d in duplikati)
        greske.Add($"Sala {d.Key.BrojSale} u {d.Key.DatumVremePocetka} se ponavlja u zahtevu");

    // 4) zauzeto u bazi
    foreach (var p in projekcije)
        if (context.Projekcije.Any(x => x.BrojSale == p.BrojSale && x.DatumVremePocetka == p.DatumVremePocetka))
            greske.Add($"Sala {p.BrojSale} je zauzeta u terminu {p.DatumVremePocetka}");

    // 5) sve-ili-nista
    if (greske.Count > 0)
        return Results.Problem(
            title: "Neispravne projekcije",
            detail: string.Join("; ", greske),
            statusCode: 400);                                          // ProblemDetails, 400

    context.Projekcije.AddRange(projekcije);
    context.SaveChanges();                                             // jedna transakcija
    return Results.Created($"/filmovi/{film.Id}", null);               // 201
});
```
Napomene:
- Uvek ide `Results.Problem(... statusCode: 400)`, a **ne** `Results.BadRequest("string")` – zadatak eksplicitno trazi *Problem Details sa naslovom i opisom*. (U `Bioskop-Dopunjeno` provera zauzetosti vraca `BadRequest(string)` – nije PD.)
- Ako se klijent zeznuo i posalje `brojProjekcije` – EF ce pokusati da ga upise. Ne diraj, u testu ga ne saljes.
- Moze se i uvesti `try/catch` oko `SaveChanges` → 500 Problem (vidi `Bioskop-Dopunjeno`).

---

### 4.5 PUT /filmovi/{id} – izmena (update) filma, i DELETE (ako traze)

Sablon za izmenu je uvek isti: **404 ako ne postoji → validacija (duplikat, ali ne racunaj sam sebe!) → izmeni polja → SaveChanges → 204 NoContent** (ili 200 sa izmenjenim objektom).

```csharp
app.MapPut("/filmovi/{id}", (int id, Film izmena, BioskopContext context) =>
{
    var film = context.Filmovi.Find(id);
    if (film is null) return Results.NotFound();                       // 404

    // duplikat – ISKLJUCI sam film (f.Id != id), inace ne mozes da sacuvas film bez promene naziva
    if (context.Filmovi.Any(f => f.Id != id && f.Naziv == izmena.Naziv && f.GodinaSnimanja == izmena.GodinaSnimanja))
        return Results.Problem(title: "Film nije validan!",
                               detail: "Vec postoji film sa datim nazivom i godinom snimanja",
                               statusCode: 400);

    film.Naziv = izmena.Naziv;                                         // menjamo polja ucitanog (tracked) entiteta
    film.Zanr = izmena.Zanr;
    film.GodinaSnimanja = izmena.GodinaSnimanja;
    film.TrajanjeMin = izmena.TrajanjeMin;

    context.SaveChanges();                                             // EF sam napravi UPDATE samo za promenjena polja
    return Results.NoContent();                                        // 204
});

app.MapDelete("/filmovi/{id}", (int id, BioskopContext context) =>
{
    var film = context.Filmovi.Find(id);
    if (film is null) return Results.NotFound();
    context.Filmovi.Remove(film);          // projekcije se brisu kaskadno (obavezan FK = cascade delete po konvenciji)
    context.SaveChanges();
    return Results.NoContent();
});
```

**Dve varijante izmene – znaj obe:**
| | Kod | Napomena |
|---|---|---|
| A) Ucitaj pa promeni polja (gore) | `Find(id)` + dodela polja + `SaveChanges()` | Sigurnije, pravi UPDATE samo promenjenih kolona. **Preporuka.** |
| B) Vezbe 05 stil | `izmena.Id = id; context.Update(izmena); context.SaveChanges();` | Kraci, ali pregazi SVE kolone vrednostima iz tela (sta nije poslato → default/null). Prvo proveri `Any(f => f.Id == id)` → 404. |

Ako izmena ide na **projekciju** (dete): isti sablon, `PUT /projekcije/{id}`, ali pri izmeni sale/vremena proveri koliziju sa **drugim** projekcijama: `context.Projekcije.Any(x => x.BrojProjekcije != id && x.BrojSale == izmena.BrojSale && x.DatumVremePocetka == izmena.DatumVremePocetka)`, plus buducnost i cena > 0.

---

## 5. Middleware – ispis metode, putanje i status koda (3p)

U `Program.cs` **posle** `var app = builder.Build();`:
```csharp
app.Use(async (context, next) =>
{
    await next();                                   // prvo obradi zahtev...
    Console.WriteLine($"{context.Request.Method} {context.Request.Path} {context.Response.StatusCode}");
});
```
**Zasto posle `next()`?** Status kod se postavlja tek kad ostatak pipeline-a obradi zahtev. Ako ispises pre `next()`, status je uvek 200.
Redosled: stavi ga rano (pre `MapGet/MapPost`).

(Varijanta iz Vezbe 05: `Stopwatch` pre i posle `next()` za merenje vremena.)

---

## 6. `.http` fajl (5p) – minimum

```http
@host = https://localhost:7007

### 1. svi filmovi
GET {{host}}/filmovi

### 1b. pretraga po nazivu
GET {{host}}/filmovi?naziv=f

### 1c. pretraga po vise kriterijuma
GET {{host}}/filmovi?zanr=horor&naziv=f1

### 2. novi film – OK (201)
POST {{host}}/filmovi
Content-Type: application/json

{ "naziv": "F3", "zanr": "Drama", "godinaSnimanja": 2024, "trajanjeMin": 100 }

### 2b. isti film opet – 400
POST {{host}}/filmovi
Content-Type: application/json

{ "naziv": "F3", "zanr": "Drama", "godinaSnimanja": 2024, "trajanjeMin": 100 }

### 3. film sa projekcijama – 200
GET {{host}}/filmovi/1

### 3b. nepostojeci film – 404
GET {{host}}/filmovi/999

### 4. uspesan unos projekcija – 201 (datumi MORAJU biti u buducnosti!)
POST {{host}}/filmovi/2/projekcije
Content-Type: application/json

[
  { "brojSale": 9,  "cenaKarte": 222.22, "datumVremePocetka": "2027-11-15T21:00:00" },
  { "brojSale": 61, "cenaKarte": 222.22, "datumVremePocetka": "2027-11-15T23:00:00" }
]

### 4b. zauzet termin (isti zahtev ponovo) – 400
POST {{host}}/filmovi/2/projekcije
Content-Type: application/json

[ { "brojSale": 9, "cenaKarte": 100, "datumVremePocetka": "2027-11-15T21:00:00" } ]

### 4c. nepostojeci film – 404
POST {{host}}/filmovi/999/projekcije
Content-Type: application/json

[ { "brojSale": 1, "cenaKarte": 100, "datumVremePocetka": "2027-12-01T20:00:00" } ]

### 5. izmena filma – 204
PUT {{host}}/filmovi/2
Content-Type: application/json

{ "naziv": "F2 izmenjen", "zanr": "Triler", "godinaSnimanja": 2025, "trajanjeMin": 120 }

### 5b. izmena nepostojeceg – 404
PUT {{host}}/filmovi/999
Content-Type: application/json

{ "naziv": "X", "zanr": "Y", "godinaSnimanja": 2025, "trajanjeMin": 90 }

### 5c. izmena u vec postojeci naziv+godinu – 400
PUT {{host}}/filmovi/2
Content-Type: application/json

{ "naziv": "F1", "zanr": "Horor", "godinaSnimanja": 2025, "trajanjeMin": 90 }
```
Port uzmi iz `Properties/launchSettings.json` (`applicationUrl`, https port). Property-ji u JSON-u su camelCase (`godinaSnimanja`).
**Datume stavi daleko u buducnost** (danas je 15.11.2026 – "2026-11-15" iz primera bi vec bio u proslosti!).

---

## 7. Ako traze KONTROLERE umesto Minimal API (Vezbe 05 – `RestWebApi`)

### 7.1 Razlike u podesavanju
`Program.cs`:
```csharp
builder.Services.AddDbContext<BioskopContext>();
builder.Services.AddControllers();            // <-- kontroleri
builder.Services.AddOpenApi();
...
app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();                         // <-- umesto MapGet/MapPost
app.Run();
```
Middleware iz sekcije 5 ide ispred `app.MapControllers();`.

### 7.2 Struktura kontrolera (iz `ProductsController`)
```csharp
[Route("api/[controller]")]        // [controller] = ime klase bez "Controller" → api/filmovi (ime klase FilmoviController)
[ApiController]
public class FilmoviController : ControllerBase
{
    private readonly BioskopContext context;
    public FilmoviController(BioskopContext context) { this.context = context; }   // DI injektuje kontekst
    ...
}
```
Odakle parametar:
- `[FromQuery] string? naziv` – iz `?naziv=...`
- `[FromRoute] int id` – iz `{id}` u ruti
- `[FromBody] Film f` – iz JSON tela (uz `[ApiController]` se podrazumeva za kompleksne tipove)
- `[FromServices]` – iz DI (u konstruktoru je uobicajeno)

Povratne vrednosti: `Ok(x)` 200, `Created(uri, x)` 201, `NoContent()` 204, `NotFound()` 404, `BadRequest(...)` 400, `Problem(title:, detail:, statusCode:)` PD.

### 7.3 Kompletan FilmoviController

```csharp
using Domain;
using Infrastructure;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

namespace RestWebApi.Controllers;

[Route("api/[controller]")]
[ApiController]
public class FilmoviController : ControllerBase
{
    private readonly BioskopContext context;
    public FilmoviController(BioskopContext context) => this.context = context;

    // GET api/filmovi?naziv=..&zanr=..
    [HttpGet]
    public IActionResult GetFilmovi([FromQuery] string? naziv, [FromQuery] string? zanr)
    {
        var q = context.Filmovi.AsQueryable();
        if (naziv is not null) q = q.Where(f => f.Naziv.Contains(naziv));
        if (zanr is not null)  q = q.Where(f => f.Zanr.Contains(zanr));

        return Ok(q.Select(f => new
        {
            f.Naziv, f.Zanr, f.GodinaSnimanja,
            UkupanBrojProjekcija = f.Projekcije.Count
        }).ToList());
    }

    // POST api/filmovi
    [HttpPost]
    public IActionResult AddFilm([FromBody] Film film)
    {
        if (context.Filmovi.Any(f => f.Naziv == film.Naziv && f.GodinaSnimanja == film.GodinaSnimanja))
            return Problem(title: "Film nije validan!",
                           detail: "Vec postoji film sa datim nazivom i godinom snimanja",
                           statusCode: 400);

        context.Filmovi.Add(film);
        context.SaveChanges();
        return Created($"api/filmovi/{film.Id}", film);
    }

    // GET api/filmovi/5
    [HttpGet("{id}")]
    public IActionResult GetById([FromRoute] int id)
    {
        var film = context.Filmovi.Include(f => f.Projekcije).FirstOrDefault(f => f.Id == id);
        if (film is null) return NotFound();

        return Ok(new
        {
            film.Id, film.Naziv, film.Zanr, film.GodinaSnimanja, film.TrajanjeMin,
            Projekcije = film.Projekcije
                .OrderBy(p => p.DatumVremePocetka)
                .Select(p => new { p.DatumVremePocetka, p.BrojSale, p.CenaKarte })
        });
    }

    // POST api/filmovi/5/projekcije
    [HttpPost("{id}/projekcije")]
    public IActionResult AddProjekcije([FromRoute] int id, [FromBody] List<Projekcija> projekcije)
    {
        var film = context.Filmovi.Find(id);
        if (film is null) return NotFound();

        var greske = new List<string>();
        foreach (var p in projekcije)
        {
            p.FilmId = film.Id;
            if (p.DatumVremePocetka <= DateTime.Now)
                greske.Add($"Projekcija u sali {p.BrojSale} ({p.DatumVremePocetka}) nije u buducnosti");
            if (p.CenaKarte <= 0)
                greske.Add($"Projekcija u sali {p.BrojSale} ({p.DatumVremePocetka}) mora imati cenu vecu od 0");
        }

        foreach (var d in projekcije.GroupBy(p => new { p.BrojSale, p.DatumVremePocetka }).Where(g => g.Count() > 1))
            greske.Add($"Sala {d.Key.BrojSale} u {d.Key.DatumVremePocetka} se ponavlja u zahtevu");

        foreach (var p in projekcije)
            if (context.Projekcije.Any(x => x.BrojSale == p.BrojSale && x.DatumVremePocetka == p.DatumVremePocetka))
                greske.Add($"Sala {p.BrojSale} je zauzeta u terminu {p.DatumVremePocetka}");

        if (greske.Count > 0)
            return Problem(title: "Neispravne projekcije", detail: string.Join("; ", greske), statusCode: 400);

        context.Projekcije.AddRange(projekcije);
        context.SaveChanges();
        return Created($"api/filmovi/{film.Id}", null);
    }
}
```
Pazi: `[ApiController]` automatski vraca 400 ako model binding ne uspe (npr. pokvaren JSON) – to je OK.
Ako u `Projekcija` imas `Film Film` kao non-nullable referencu, `[ApiController]` moze da trazi `film` u JSON-u ("The Film field is required") – tada ili stavi `public Film? Film { get; set; }` ili iskljuci Nullable (`Nullable` u csproj). U Vezbe 05 je zato `Product.Category` deklarisana kao `ProductCategory?`. **Ako dobijes 400 "field is required" – odmah stavi `?`.**
(Isto vazi i za `Film.Projekcije` → `List<Projekcija>?` ako dobijes istu gresku pri POST-u filma. Minimal API ovo ne radi, tamo je radilo bez `?`.)

Dodatno iz Vezbe 05 (moze se pitati): `[HttpPut]` update → `context.Update(p); context.SaveChanges(); return NoContent();` posle provere `Any(...)` → `NotFound`. I `[HttpDelete("{id}")]` → `Remove(entity)` + `SaveChanges`.

**Update i delete u kontroleru (isto kao 4.5):**
```csharp
// PUT api/filmovi/5
[HttpPut("{id}")]
public IActionResult UpdateFilm([FromRoute] int id, [FromBody] Film izmena)
{
    var film = context.Filmovi.Find(id);
    if (film is null) return NotFound();

    if (context.Filmovi.Any(f => f.Id != id && f.Naziv == izmena.Naziv && f.GodinaSnimanja == izmena.GodinaSnimanja))
        return Problem(title: "Film nije validan!",
                       detail: "Vec postoji film sa datim nazivom i godinom snimanja",
                       statusCode: 400);

    film.Naziv = izmena.Naziv;
    film.Zanr = izmena.Zanr;
    film.GodinaSnimanja = izmena.GodinaSnimanja;
    film.TrajanjeMin = izmena.TrajanjeMin;
    context.SaveChanges();
    return NoContent();                                   // 204
}

// DELETE api/filmovi/5
[HttpDelete("{id}")]
public IActionResult DeleteFilm([FromRoute] int id)
{
    var film = context.Filmovi.Find(id);
    if (film is null) return NotFound();
    context.Filmovi.Remove(film);
    context.SaveChanges();
    return NoContent();
}
```
(Vezbe 05 `ProductsController` ima `[HttpPut]` bez `{id}` u ruti – id dolazi iz tela i radi se `context.Update(p)`. Oba pristupa su prihvatljiva; `PUT /{id}` je cistiji REST.)


### 7.4 Minimal API vs kontroleri – kratko poredjenje
| | Minimal API | Kontroler |
|---|---|---|
| Ruta | `app.MapGet("/filmovi", ...)` | `[HttpGet]` + `[Route]` na klasi |
| DI | parametar lambde | konstruktor |
| Registracija | nista posebno | `AddControllers()` + `MapControllers()` |
| Povratna vrednost | `Results.Ok/NotFound/Problem/Created` | `Ok/NotFound/Problem/Created` (iz `ControllerBase`) |
| Telo zahteva | parametar tipa Film | `[FromBody]` |

---

## 8. Plan od 90 minuta

| Min | Sta |
|-----|-----|
| 0–10 | Solution + 3 projekta + reference + paketi (Sekcija 1) |
| 10–25 | Film, kontekst, `Init`, pa Projekcija + veza + 2. migracija, `database update` (Sekcija 2) |
| 25–40 | Middleware (3p lako) + GET lista/pretraga + POST film |
| 40–50 | GET po id |
| 50–75 | POST vise dece (najvise poena – ne zuri) |
| 75–90 | `.http` fajl, pokreni sve, popravi |

Strategija: odmah posle svake funkcionalnosti napisi njen `.http` zahtev i pokreni – tako ne ostajes sa 5p neurađenih na kraju.
Ako zapnes na migraciji – **ne troši vreme**, nastavi, pa se vrati.

---

## 9. Kako da NAUCIS za najkrace vreme (plan vezbanja)

**Dan 1 – razumi**
1. Procitaj sekcije 2, 3 i 4 pazljivo. Otvori `Bioskop-Dopunjeno` i prati `Program.cs` liniju po liniju.
2. Pitaj se za svaku liniju "sta bi se promenilo da je entitet Autor/Knjiga?".

**Dan 2 – prekucaj (bez gledanja)**
1. Obrisi/napravi novi prazan folder. Uradi **ceo** zadatak od nule za Bioskop u Minimal API – sa listom komandi iz Sekcije 1 pored sebe.
2. Zapisi sta si morao da pogledas. To je tvoja lista slabih tacaka.

**Dan 3 – zameni entitete + kontroleri**
Uradi isto za **drugu temu** (preslikavanje ispod), ovaj put sa **kontrolerima**.

| Bioskop | Biblioteka | Klinika | Skola |
|---|---|---|---|
| Film (1) | Autor | Doktor | Kurs |
| Projekcija (N) | Knjiga | Pregled | Cas |
| Naziv, Zanr, Godina, Trajanje | Ime, Drzava, ... | Ime, Specijalnost | Naziv, Oblast |
| BrojSale + DatumVremePocetka (kolizija) | ... | Ordinacija + Termin | Ucionica + Termin |
| CenaKarte > 0 | Cena > 0 | Cena > 0 | ... |
| "isti naziv i godina" (duplikat roditelja) | isto | Ime + Specijalnost | Naziv + Godina |

**Dan 4 – vreme**
Uradi ceo zadatak na vreme od 90 min, bez kopiranja. Cilj: 70 min.

### Pitanja za samoproveru (mogu doci i usmeno/teorija)
- Sta je Code First? (Prvo klase, pa EF generise bazu preko migracija.)
- Cemu sluzi migracija; razlika `migrations add` i `database update`?
- Sta je `DbContext`, `DbSet`? Zasto `AddDbContext` (Scoped)?
- Razlika `Find` / `FirstOrDefault` / `SingleOrDefault`? `Include` vs `Select`?
- Sta je odlozeno izvrsavanje (deferred execution) i `IQueryable`?
- Zasto middleware ispisuje status posle `await next()`?
- Zasto vracamo anonimne objekte, a ne entitete? (Ciklicne reference, vise podataka nego treba.)
- Koja je razlika izmedju `BadRequest()` i `Problem()`? (`Problem` = standardizovan RFC 7807 JSON: `title`, `detail`, `status`.)
- Kako se konfigurise veza 1-N u Fluent API-ju?

---

## 10. Checklist pred predaju

- [ ] 3 projekta, reference tacne, `AddDbContext` u Program.cs
- [ ] PK je auto-inkrement ili nije – **tacno kako zadatak trazi** (Sekcija 2.3), i POST to uvazava
- [ ] Migracija **Init** (samo Film) i **druga** migracija (Projekcija) – obe vidljive u `Migrations` folderu, baza azurirana
- [ ] GET lista radi bez filtera, sa jednim i sa dva filtera; vraca broj projekcija
- [ ] POST film: duplikat → 400, uspeh → 201
- [ ] GET po id: 404 / 200, projekcije **sortirane**
- [ ] POST projekcije: 404; buducnost + cena > 0; duplikati u zahtevu; zauzeto u bazi → **ProblemDetails 400**; ne upisuje se nista ako je ijedna losa; uspeh 201
- [ ] Middleware ispisuje `METODA /putanja statusKod`
- [ ] `.http` sadrzi ~12 zahteva (uspesni + gresni) i svi su pokrenuti
- [ ] Datumi u testovima su u buducnosti

## 11. Cesti problemi i brza resenja

| Simptom | Uzrok / resenje |
|---|---|
| `Unable to create a DbContext` pri `Add-Migration` | Startup projekat nije Web; Default project nije Infrastructure; fali `Microsoft.EntityFrameworkCore.Design` u API projektu |
| `Cannot open database` / login greska | LocalDB nije pokrenut: `sqllocaldb start mssqllocaldb`; proveri connection string |
| `System.Text.Json ... possible object cycle` | Vracas entitet sa navigacijama – koristi `Select(new {...})` |
| 400 `"The Film field is required"` (kontroler) | Navigaciju stavi nullable: `Film? Film` |
| Prazna lista projekcija u odgovoru | Fali `Include(f => f.Projekcije)` |
| Middleware uvek ispisuje 200 | Ispis je pre `await next()` – prebaci posle |
| `NullReference` u pretrazi | Primenjujes `Contains(null)` – proveri `is not null` pre `Where` |
| 400 u unosu projekcija a sve izgleda dobro | Datum u `.http` nije u buducnosti |
| Duplicate key / PK violation (500) pri POST-u | Id nije auto-inkrement, a ne proveravas da li vec postoji → Sekcija 2.3 |
| Id uvek 0 u bazi / "Cannot insert explicit value for identity column" | Neslaganje: kolona je IDENTITY a saljes id (ili obrnuto) → proveri `ValueGeneratedNever()`/`OnAdd()`, pa migracija iznova |
| Migracija trazi da se promeni PK | `HasKey` za PK koji se ne zove `Id`/`<Klasa>Id` |
| Baza se ne poklapa sa modelom | Zaboravljen `database update` |
