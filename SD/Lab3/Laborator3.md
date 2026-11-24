# Laboratorul 3: Desenează limita Dashboard-ului

**Autor:** Cristea Alexandru

## Personal Investment Dashboard

## Context

În primul laborator am stabilit produsul, iar în al doilea am fixat obiectivele de calitate și volumul de lucru. La nivelul maxim analizat, Dashboard-ul trebuie să poată duce aproximativ **10.296 RPS în momentul deschiderii pieței**, iar citirea `Stock price` reprezintă fluxul cu cea mai mare intensitate.

În laboratorul de față deschid granița sistemului și împart Dashboard-ul în câteva părți, fiecare cu responsabilități bine delimitate. Proiectarea rămâne intenționat de simplă. Nu adaug containere sau componente care nu sunt necesare pentru funcțiile descrise în primele două laboratoare.

Premisele pe care le folosesc în raport sunt:

- autentificarea se face printr-un **Identity Provider extern**;
- Market Data Provider rămâne un furnizor generic;
- Market Data Collector aduce datele de piață la fiecare **1 minut**;
- citirile făcute de utilizator se bazează pe datele deja sincronizate și **nu generează o cerere nouă către Market Data Provider**;
- datele publice de piață și datele private ale utilizatorului sunt ținute separat;
- fiecare utilizator poate citi și modifica exclusiv propriul Watchlist;
- după o modificare confirmată a Watchlist-ului, următoarea citire a aceluiași utilizator trebuie să reflecte modificarea;
- dacă sincronizarea cu furnizorul eșuează, ultima valoare validă nu se șterge;
- pentru Stock price păstrez regula din Laboratorul 2: până la 15 minute datele sunt acceptate împreună cu timestamp-ul, între 15 și 30 de minute rezultatul este `Stale`, iar după 30 de minute devine `Unavailable`.

# 1. Diagrama System Context

La acest nivel nu se vede structura internă a Dashboard-ului. Sunt reprezentați doar utilizatorul și sistemele externe cu care produsul comunică direct.

```mermaid
flowchart LR
    User["Utilizator autentificat<br/>urmărește Stocks, prețuri, istoric și Watchlist"]
    Dashboard["Personal Investment Dashboard<br/>Sistemul analizat"]
    IdP["Identity Provider<br/>Sistem extern"]
    Provider["Market Data Provider<br/>Sistem extern"]

    User -->|"Răsfoiește și filtrează Stocks, citește prețuri și istoric, administrează Watchlist"| Dashboard
    Dashboard -->|"Verifică identitatea utilizatorului"| IdP
    IdP -->|"Returnează rezultatul verificării identității"| Dashboard
    Dashboard -->|"Solicită periodic date de piață"| Provider
    Provider -->|"Returnează date de piață sau un rezultat de eroare"| Dashboard
```

**Rolurile din diagramă:**

- **Utilizator autentificat** – actorul uman care interacționează direct cu sistemul.
- **Personal Investment Dashboard** – sistemul de interes.
- **Identity Provider** – confirmă identitatea utilizatorului.
- **Market Data Provider** – livrează datele de piață pe care Dashboard-ul le folosește.

Market Data Provider poate răspunde cu date întârziate, cu date invalide, poate limita numărul de cereri sau poate fi indisponibil. Dashboard-ul rămâne responsabil pentru felul în care aceste situații ajung la utilizator.

# 2. Diagrama Container

În acest view deschid granița Dashboard-ului. Folosesc patru containere interne, suficiente pentru funcțiile și regulile stabilite până acum.

```mermaid
flowchart LR
    User["Utilizator autentificat"]
    IdP["Identity Provider<br/>Sistem extern"]
    Provider["Market Data Provider<br/>Sistem extern"]

    subgraph Dashboard["Personal Investment Dashboard"]
        App["Dashboard Application<br/>Servește citirile utilizatorului și modificările Watchlist"]
        Collector["Market Data Collector<br/>Preia, verifică și acceptă datele de piață"]
        MarketStore[("Market Data Store<br/>Stocks, latest price și price history")]
        UserStore[("User Data Store<br/>Watchlist-uri private")]
    end

    User -->|"Overview, Filter, Search, Stock price, History, Watchlist"| App
    App -->|"Validează identitatea"| IdP
    IdP -->|"Identitate validă sau respinsă"| App

    App -->|"Citește Stocks, prețuri și istoric"| MarketStore
    App -->|"Citește și modifică Watchlist-ul utilizatorului"| UserStore

    Collector -->|"Solicită date folosind credențiale păstrate pe server"| Provider
    Provider -->|"Date, rate limit, timeout sau eroare"| Collector
    Collector -->|"Scrie numai date validate și acceptate"| MarketStore
```

## Responsabilitățile containerelor

| Container | Responsabilitate |
|---|---|
| **Dashboard Application** | Primește cererile utilizatorului, verifică identitatea, validează input-ul și returnează Overview, Filter, Search, Stock price, History și Watchlist. |
| **Market Data Collector** | Sincronizează periodic datele de piață, verifică răspunsurile furnizorului și salvează doar rezultatele acceptate. |
| **Market Data Store** | Ține datele despre Stocks, ultimul preț acceptat și price history. |
| **User Data Store** | Ține Watchlist-urile private împreună cu legătura lor către proprietar. |

Separarea `Market Data Collector` de `Dashboard Application` contează pentru că citirile utilizatorilor nu trebuie să depindă de un apel nou la furnizor. Pentru 30.000 de utilizatori, Laboratorul 2 a estimat 7.920 RPS în regim stabil și 10.296 RPS la deschiderea pieței. Acest trafic nu este tratat ca trafic direct către Market Data Provider.

Prin separarea `Market Data Store` de `User Data Store`, datele publice de piață rămân distincte de datele private ale utilizatorilor.


# 3. Diagrama Component

Această diagramă deschide **numai Dashboard Application**. Market Data Collector nu este descompus aici, pentru că cerința laboratorului spune să fie deschis doar Dashboard Application.

```mermaid
flowchart LR
    User["Utilizator autentificat"]
    IdP["Identity Provider<br/>Sistem extern"]
    MarketStore[("Market Data Store<br/>Container")]
    UserStore[("User Data Store<br/>Container")]

    subgraph App["Dashboard Application"]
        Access["Access Control<br/>Verifică identitatea și creează contextul utilizatorului"]
        StockRead["Stock Read<br/>Overview și Stock price"]
        SearchFilter["Search & Filter<br/>Caută și filtrează Stocks"]
        History["History<br/>Citește price history"]
        Watchlist["Watchlist<br/>Citește și modifică Watchlist-ul privat"]
    end

    User -->|"Trimite cereri autentificate"| Access
    Access -->|"Validează identitatea"| IdP
    IdP -->|"Identitate validă sau respinsă"| Access

    Access -->|"Cerere autorizată pentru Overview / Stock price"| StockRead
    Access -->|"Cerere autorizată pentru Search / Filter"| SearchFilter
    Access -->|"Cerere autorizată pentru History"| History
    Access -->|"Cerere autorizată pentru Watchlist"| Watchlist

    StockRead -->|"Citește latest price și date Stock"| MarketStore
    SearchFilter -->|"Citește datele de referință Stock"| MarketStore
    History -->|"Citește price history"| MarketStore
    Watchlist -->|"Citește proprietarul și modifică Watchlist"| UserStore
```

## Responsabilitățile componentelor

| Componentă | Responsabilitate |
|---|---|
| **Access Control** | Verifică identitatea prin Identity Provider și transmite mai departe identitatea utilizatorului validat. |
| **Stock Read** | Servește Overview și Stock price și aplică regulile `Stale` / `Unavailable`. |
| **Search & Filter** | Caută și filtrează instrumentele acceptate de V1. |
| **History** | Returnează istoricul de preț disponibil pentru un Stock. |
| **Watchlist** | Verifică proprietarul și citește sau modifică Watchlist-ul privat. |

În diagramă nu apar clase, metode sau funcții. Fiecare componentă descrie o responsabilitate internă a aceluiași `Dashboard Application`.


# 4. Diagrame de secvență

## A. Sincronizarea datelor de piață

Market Data Collector pornește sincronizarea la intervale regulate. Credențialele pentru Market Data Provider rămân în partea de server și nu ajung la utilizator.

```mermaid
sequenceDiagram
    participant Collector as Market Data Collector
    participant Provider as Market Data Provider
    participant Store as Market Data Store

    Note over Collector: Pornește sincronizarea la fiecare 1 minut
    Collector->>Provider: Solicită date de piață cu credențiale server-side

    alt Furnizorul răspunde
        Provider-->>Collector: Date de piață + timestamp furnizor
        Collector->>Collector: Verifică schema, simbolul, tipul, prețul și timestamp-ul

        alt Date valide
            Collector->>Store: Salvează datele acceptate
            Store-->>Collector: Scriere confirmată
            Note over Store: Datele noi sunt disponibile pentru citirile următoare
        else Date invalide
            Collector->>Collector: Respinge lotul sau înregistrarea invalidă
            Note over Store: Datele acceptate anterior rămân neschimbate
        end

    else Timeout sau rate limit
        Provider--xCollector: Timeout / rate limit / indisponibil
        Note over Store: Nu se șterg datele acceptate anterior
    end
```

### Ipoteze pentru flux

- sincronizarea are loc la fiecare **1 minut**;
- Market Data Collector păstrează credențialele furnizorului pe server;
- utilizatorii văd datele doar după ce scrierea în Market Data Store este confirmată;
- o sincronizare eșuată nu înlocuiește datele valide cu zero și nici nu le elimină;
- dacă datele existente îmbătrânesc, Dashboard Application aplică regula de actualitate din Laboratorul 2;
- un răspuns invalid al furnizorului nu devine automat rezultat valid în Dashboard.

## B. Citirea prețului unui Stock

Citirea prețului se face din datele deja sincronizate. Nu există o cerere nouă către Market Data Provider pentru fiecare citire a utilizatorului.

```mermaid
sequenceDiagram
    participant User as Utilizator
    participant Access as Access Control
    participant IdP as Identity Provider
    participant Stock as Stock Read
    participant Store as Market Data Store

    User->>Access: Cere Stock price pentru un simbol
    Access->>IdP: Validează identitatea

    alt Identitate invalidă
        IdP-->>Access: Respins
        Access-->>User: Unauthorized
    else Identitate validă
        IdP-->>Access: userId valid
        Access->>Stock: Citește prețul pentru simbol

        Stock->>Stock: Validează simbolul și domeniul acceptat

        alt Simbol invalid sau nesuportat
            Stock-->>User: Unsupported / cerere respinsă
        else Simbol valid
            Note over Stock,Store: Citirea NU apelează Market Data Provider
            Stock->>Store: Citește latest price și provider timestamp

            alt Defectare a stocării
                Store--xStock: Citirea eșuează
                Stock-->>User: Unavailable
            else Preț lipsă
                Store-->>Stock: Nu există preț acceptat
                Stock-->>User: Unavailable
            else Preț disponibil
                Store-->>Stock: Preț + provider timestamp

                alt Vechime <= 15 minute
                    Stock-->>User: Preț + timestamp / întârziere
                else Vechime > 15 minute și <= 30 minute
                    Stock-->>User: Stale + ultimul preț acceptat + timestamp
                else Vechime > 30 minute
                    Stock-->>User: Unavailable
                end
            end
        end
    end
```

### Ipoteze pentru flux

- orice cerere de citire trece mai întâi prin verificarea identității;
- simbolul este validat înainte de accesarea datelor;
- `Stock Read` citește exclusiv din Market Data Store;
- o citire obișnuită a utilizatorului **nu produce o cerere nouă către Market Data Provider**;
- un preț absent nu este afișat ca `0`;
- până la 15 minute prețul este acceptat cu timestamp-ul furnizorului;
- între 15 și 30 de minute este afișat ca `Stale`;
- după 30 de minute rezultatul devine `Unavailable`;
- o defectare a Market Data Store duce la `Unavailable`, nu la un preț inventat.


## C. Modificarea unui Watchlist privat

Succesul este raportat doar după ce modificarea a fost scrisă și confirmată în User Data Store.

```mermaid
sequenceDiagram
    participant User as Utilizator
    participant Access as Access Control
    participant IdP as Identity Provider
    participant Watchlist as Watchlist
    participant Store as User Data Store

    User->>Access: Adaugă sau elimină un simbol din Watchlist
    Access->>IdP: Validează identitatea

    alt Identitate invalidă
        IdP-->>Access: Respins
        Access-->>User: Unauthorized
    else Identitate validă
        IdP-->>Access: userId valid
        Access->>Watchlist: Modificare Watchlist cu userId
        Watchlist->>Watchlist: Validează simbolul și acțiunea

        alt Date de intrare invalide
            Watchlist-->>User: Cerere respinsă
        else Date valide
            Watchlist->>Store: Citește proprietarul Watchlist-ului
            Store-->>Watchlist: ownerId

            alt ownerId diferit de userId
                Watchlist-->>User: Unauthorized
            else Utilizatorul este proprietar
                Watchlist->>Store: Scrie modificarea

                alt Defectare a stocării
                    Store--xWatchlist: Scriere eșuată
                    Watchlist-->>User: Unavailable / modificarea nu este confirmată
                else Scriere confirmată
                    Store-->>Watchlist: Modificare salvată
                    Note over Watchlist,Store: Abia aici poate fi raportat succesul
                    Watchlist-->>User: Success

                    User->>Access: Citește din nou propriul Watchlist
                    Access->>IdP: Validează identitatea
                    IdP-->>Access: userId valid
                    Access->>Watchlist: Citește Watchlist pentru userId
                    Watchlist->>Store: Citește Watchlist-ul utilizatorului
                    Store-->>Watchlist: Watchlist cu modificarea confirmată
                    Watchlist-->>User: Rezultat actualizat
                end
            end
        end
    end
```

### Ipoteze pentru flux

- Watchlist-ul are un proprietar identificat prin `userId`;
- proprietarul este verificat înainte de orice modificare;
- un utilizator nu poate citi sau modifica Watchlist-ul altui utilizator;
- succesul este returnat doar după confirmarea scrierii în User Data Store;
- dacă scrierea eșuează, Dashboard-ul nu susține că modificarea a fost făcută;
- după un `Success`, următoarea citire a aceluiași utilizator trebuie să includă modificarea confirmată;
- regula menține garanția `read-your-writes` definită în Laboratorul 2.


# 5. Verificarea proiectării față de Laboratorul 2

Arhitectura răspunde direct punctelor de presiune identificate în laboratorul anterior.

| Cerință sau risc | Decizie în această proiectare |
|---|---|
| Stock price generează cel mai mare trafic | `Stock Read` citește din Market Data Store și nu apelează furnizorul pentru fiecare cerere. |
| 10.296 RPS la deschiderea pieței | Traficul utilizatorilor este separat de sincronizarea furnizorului. Capacitatea reală trebuie verificată ulterior prin load testing. |
| Furnizorul poate fi lent sau indisponibil | `Market Data Collector` izolează sincronizarea de citirile utilizatorului. |
| Furnizorul poate trimite date invalide | Collector-ul validează înainte de salvare. |
| Prețurile pot deveni vechi | `Stock Read` clasifică rezultatul ca acceptat, `Stale` sau `Unavailable`. |
| Watchlist-ul este privat | `Access Control` validează identitatea, iar `Watchlist` verifică proprietarul. |
| Read-your-writes pentru Watchlist | Succesul este dat numai după scriere confirmată; următoarea citire vede modificarea. |

Estimarea de 10.296 RPS creează o cerință de capacitate, dar nu demonstrează singură că arhitectura poate susține volumul. Confirmarea se obține prin măsurare și testare de încărcare.


# 6. Concluzie

Proiectarea se sprijină pe patru containere interne:

- Dashboard Application;
- Market Data Collector;
- Market Data Store;
- User Data Store.

Dashboard Application are cinci responsabilități principale: Access Control, Stock Read, Search & Filter, History și Watchlist.

Fluxul de market data rămâne separat de citirile utilizatorului. Când Market Data Provider are timeout, rate limit sau trimite date invalide, datele acceptate anterior rămân disponibile și sunt evaluate după regulile de actualitate. Astfel, o problemă a furnizorului nu se transformă automat într-un rezultat fals pentru utilizator.

Datele private sunt tratate separat. Watchlist-ul poate fi modificat doar de proprietar, iar succesul este raportat numai după confirmarea scrierii.

Această structură acoperă cerințele actuale și nu introduce încă tehnologii concrete, produse cloud sau detalii de implementare care nu sunt cerute de laborator.



## Surse de curs

- Laboratorul 3: **Desenează limita Dashboard-ului**.
- Lecția 3: **Cerințe de calitate, volum de lucru și capacitate**.
- Laboratorul 2: **Cuantifică citirile Dashboard-ului**.
- C4 Model: System Context, Container și Component views.
