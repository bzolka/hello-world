# Függőséginjektálás ASP.NET Core környezetben

Remove this line

## Szerzői jogok
Benedek Zoltán, 2018

Jelen dokumentum a BME Villamosmérnöki és Informatikai Kar hallgatói számára készített elektronikus jegyzet. A dokumentumot az Adatvezérelt rendszerek c. tantárgyat felvevő hallgatók jogosultak használni, és saját céljukra 1 példányban kinyomtatni. A dokumentum módosítása, bármely eljárással részben vagy egészben történő másolása tilos, illetve csak a szerző előzetes engedélyével történhet.
A
## Definíció

Függőséginjektálás (angolul __Dependency Injection__, röiden DI) egy tervezési minta. A fejlesztőket segíti abban, hogy az alkalmazás egyes részei laza csatolással kerüljenek kialakításra.
__A függőséginjektálás egy mechanizmus arra, hogy az osztály függőségi gráfjainak létrehozását függetlenítsük az osztály definíciójától.__

Célok
* könnyebb bővíthetőség
* unit tesztelhetőség

Természetesen a fenti defínícóból önmagában nem derül ki, pontosan milyen problémákat milyen módon old meg a DI. A következő fejezetekben egy példa segítségével helyezzük kontextusba a problémakört, illetve példa keretében megísmerkedünk az ASP.NET Core beépített DI szolgáltatának alapjaival.


## Példa, 1. fázis - szolgáltatás osztály beégetett függőségekkel

A példánkban egy teendőlista (TODO) kezelő alkalmazás e-mail értesítéseket küldő részeibe tekintünk bele, kódrészletek segítségével. Megjegyzés: a kód a tömörség érdekében minimalisztikus, funkcionálisan nem működik.

A példánk "belépési pontja" a `ToDoService` osztály `SendReminderIfNeeded` művelete. 

<pre>

// Teendők kezeléséle szolgáló osztály
public class ToDoService
{    
    // Megvizsgálja a paraméterként kapott todoItem objektumot, és ha szükséges,
    // e-mail értesítést küld a teendőről a teendőben szereplő kontakt személynek.
    public void SendReminderIfNeeded(TodoItem todoItem)
    {
        if (<b>checkIfTodoReminderIsToBeSent(todoItem)</b>)
        { 
            <b>NotificationService</b> notificationService = new NotificationService(smtpAddress);
            notificationService.SendEmailReminder(todoItem.LinkedContactId, todoItem.Name);
        }
    }

    bool checkIfTodoReminderIsToBeSent(TodoItem todoItem)
    {
        bool send = true; 
        /* ... */
        return send;
    }
    // ...
}



// Entitásosztály, egy végrehajtandó feladat adatait zárja egységbe
public class TodoItem
{
    // Adatbázis kulcs
    public long Id { get; set; }
    // Teendő neve/leírása
    public string Name { get; set; }
    // Jelzi, hogy a teendő elvégésre került-e
    public bool IsComplete { get; set; }
    // Egy teendőhöz lehetőség van kontakt személy hozzárendeléséhez:  ha -1, nincs
    // kontakt személy hozzárendelve, egyébként pedig a kontakt személy azonosítója.
    public int LinkedContactId { get; set; } = -1;
}
</pre>

A fenti kódban (`ToDoService.SendReminderIfNeeded`) azt látjuk, hogy az e-mail küldés lényegi logikáját `NotificationService` osztályban kell keresnünk. Valóban, vizsgálódásunk központjába ez az osztály kerül. A következő kódrészlet ezen osztály kódját, valamint a függőségeit mutatja be:

```csharp
// Értesítések küldésére szolgáló osztály
class NotificationService 
{
    // Az osztály függőségei
    <b>EMailSender _emailSender;<b>
    Logger _logger;
    ContactRepository _contactRepository;

    public NotificationService(string smtpAddress)
    {
        _logger = new Logger();
        _emailSender = new EMailSender(_logger, smtpAddress);
        _contactRepository = new ContactRepository();
    }

    // E-mail értesítést küld az adott azonosítójú kontakt személynek (a contactId
    // egy kulcs a Contacts táblában)
    public void SendEmailReminder(int contactId, string todoMessage)
    {
        string emailTo = _contactRepository.GetContactEMailAddress(contactId);
        string emailSubject = "TODO reminder";
        string emailMessage = "Reminder about the following todo item: " + todoMessage;
        _emailSender.SendMail(emailTo, emailSubject, emailMessage);
    }
}

// Naplózást támogató osztály
public class Logger
{
    public void LogInformation(string text) { /* ...*/ }
    public void LogError(string text) { /* ...*/ }
}

// E-mail küldésre szolgáló osztály
public class EMailSender
{
    Logger _logger;
    string _smtpAddress;

    public EMailSender(Logger logger, string smtpAddress)
    {
        _logger = logger;
        _smtpAddress = smtpAddress;
    }
    public void SendMail(string to, string subject, string message)
    {
        _logger.LogInformation($"Sendding e-mail. To: {to} Subject: {subject} Body: {message}");

        // ...
    }
}

// Contact-ok perzisztens kezelésére szolgáló osztály
public class ContactRepository
{
    public string GetContactEMailAddress(int contactId)
    {
        throw new NotImplementedException();
    }
    // ...
}
```

Pár általános gondolat:

* A `NotificationService` osztály több függőséggel rendelkezik (`EMailSender`, `Logger`, `ContactRepository` osztályok), ezen osztályokra építve valósítja meg a szolgáltatásait. 
* A függőség osztályoknak lehetnek további függőségeik: az `EMailSender` remek példa erre, épít a `Logger` osztélyra. 
* Megjegyzés: a `NotificationService`, `EMailSender`, `Logger`, `ContactRepository` osztályokat **szolgáltatásosztályoknak** tekintjük, mert tényleges logikát is tartalmaznak, nem csak adatokat zárnak egységbe, mint pl. a `TodoItem`.
  
Tekintsük át a megoldás legfontosabb jellemzőit:

1. Az osztály a függőségeit maga példányosítja
2. Az osztály a függőségei konkrét típusától függ (nem interfészektől, "absztrakcióktól")

Ez a megközelítés több súlyos negatívummal bír:

1. **Rugalmatlanság, nehéz bővíthetőség**. A `NotificationService` (módosítás nélkül) nem tud más levélküldő, naplózó és contact repository implementációkkal együtt működni, csak a beégetett `EMailSender`, `Logger` és `ContactRepository` osztályokkal. Vagyis pl. nem tudjuk más naplózó komponenssel, vagy olyan contact repository-vek használni, amely más forrásból dolgozik.
2. **Unit tesztelhetőség hiánya**. A `NotificationService` (módosítás nélkül) **nem unit tesztelhető**. Ehhez ugyanis le kell cserélni az `EMailSender`, `Logger` és `ContactRepository` függőségeit olyanokra, melyek (tesztelést segítő) egyszerű/rögzített válaszokat viselkedést mutatnak. Ne feledjük: a unit tesztelés lényege, hogy egy osztály viselkedését önmagában teszteljük (pl. az adatbázist használó ContactRepository helyett egy olyan ContactRepository-ra van szükség, mely gyorsan, memóriából szolgálja ki a kéréseket, a teszt előfeltetételeinek megfelelően).
3. Kellemetlen, hogy a `NotificationService`-nek a függőségei paramétereit is át kell adni (smtpAddress, pedig ehhez a NotificationService-nek elvileg semmi köze, ez csak az `EMailSender`-re tartozik).

A következő lépésben úgy alakítjuk át a megoldásunkat, hogy a negatívumok többségétől meg tudjunk szabadulni.

## Példa, 2. fázis - szolgáltatás osztály manuális függőség injektálással

A korábbi megoldásunkat alakítjuk át, a funkcinális követelmények változatlanok. Az átalakítás legfontosabb irányelvei: a függőségeket "interfész alapokra" helyezzük, és az osztályok nem maguk példányosítják a függőségeiket.

```csharp
    public class ToDoService
    {
        const string smtpAddress = "smtp.myserver.com";

        // Megvizsgálja a paraméterként kapott todoItem objektumot, és ha szükséges,
        // e-mail értesítést küld a teendőről a teendőben szereplő kontakt személynek.
        public void SendReminderIfNeeded(TodoItem todoItem)
        {
            if (checkIfTodoReminderIsToBeSent(todoItem))
            {
                var logger = new Logger();
                var emailSender = new EMailSender(logger, smtpAddress);
                var contactRepository = new ContactRepository();

                NotificationService notificationService
                    = new NotificationService(logger, emailSender, contactRepository);
                notificationService.SendEmailReminder(todoItem.LinkedContactId,
                    todoItem.Name);
            }
        }

        bool checkIfTodoReminderIsToBeSent(TodoItem todoItem)
        {
            bool send = true;
            /* ... */
            return send;
        }
    }

// Értesítések küldésére szolgáló osztály
class NotificationService 
{
    // Az osztály függőségei
    IEMailSender _emailSender;
    ILogger _logger;
    IContactRepository _contactRepository;

    public NotificationService(ILogger logger, IEMailSender emailSender, 
        IContactRepository contactRepository)
    {
        _logger = logger;
        _emailSender = emailSender;
        _contactRepository = contactRepository;
    }

    // E-mail értesítést küld az adott azonosítójú kontakt személynek (a contactId
    // egy kulcs a Contacts táblában)
    public void SendEmailReminder(int contactId, string todoMessage)
    {
        string emailTo = _contactRepository.GetContactEMailAddress(contactId);
        string emailSubject = "TODO reminder";
        string emailMessage = "Reminder about the following todo item: " + todoMessage;
        _emailSender.SendMail(emailTo, emailSubject, emailMessage);
    }
}

#region Contracts (abstractions)

// Naplózást támogató interfész
public interface ILogger
{
    void LogInformation(string text);
    void LogError(string text);
}

// E-mail küldésre szolgáló interfész
public interface IEMailSender
{
    void SendMail(string to, string subject, string message);
}

// Contact-ok perzisztens kezelésére szolgáló interfész
public interface IContactRepository
{
    string GetContactEMailAddress(int contactId);
}

#endregion

#region Implementations

// Naplózást támogató osztály
public class Logger: ILogger
{
    public void LogInformation(string text) { /* ...*/  }
    public void LogError(string text) {  /* ...*/  }
}

// E-mail küldésre szolgáló osztály
public class EMailSender: IEMailSender
{
    ILogger _logger;
    string _smtpAddress;

    public EMailSender(ILogger logger, string smtpAddress)
    {
        _logger = logger;
        _smtpAddress = smtpAddress;
    }
    public void SendMail(string to, string subject, string message)
    {
        _logger.LogInformation($"Sendding e-mail. To: {to} Subject: {subject} Body: {message}");

        // ...
    }
}

// Contact-ok perzisztens kezelésére szolgáló osztály
public class ContactRepository: IContactRepository
{
    public string GetContactEMailAddress(int contactId)
    {
        throw new NotImplementedException();
    }
    // ...
}

#endregion
```

A korábbi megoldást a következő pontokban fejlesztettük tovább:

* A `NotificationService` osztály már nem maga példányosítja a függőségeit, hanem konstruktor  paraméterekben kapja meg.
* Interfészeket (absztrakciókat) vezettünk be a függőségek kezelésére
* A `NotificationService` osztály a függőségeit interfészek formájában kapja meg. Azt, amikor egy osztály a függőségeit kívülről kapja meg, **DEPENDENCY INJECTION**-nek (DI) vagyis függőséginjektálásnak nevezzük.
* Esetünkben konstruktor paraméterekben kapta meg az osztály függőségeit, ez **CONSTRUCTOR INJECTION**-nek (konktruktor injektálás) nevezzük. Ez a függőséginjektálás legyakoribb - és leginkább javasolt módja (alternatíva pl. a property injection, amikor gy publikus property setterének segítségével állítjuk be az osztály adott függőségét).

A megoldásunkban a `NotificationService` függőségeit az osztály (közvetlen) FELHASZNÁLÓJA példányosítja (`ToDoService` osztály). Bár a kódból nem látszik, az elveinknek megfelelően ez nem csak ezeknél az osztályoknál áll fent, hanem az alkalmazásban számos helyen. Elsődlegesen ebből eredően a következő problémák állnak még fent:
1. A `NotificationService` felhasználója, vagyis a `ToDoService.SendReminderIfNeeded` még mindig függ a konkrét típusoktól (hiszen neki szükséges példányosítania a `Logger`, `EMailSender` és `ContactRepository` osztályokat)
2. Ha több helyen használjuk a Logger, EMailSender és ContactRepository osztályokat, mindenhol külön-külön példányosítani kell őket. Vagyis NEM EGYETLEN KÖZPONTI HELYEN határozzuk meg hogy milyen interfész típus esetén milyen implementációt kell MINDENOL használni az alkalmazásban (pl. ILogger->Logger, IMailSender->EMailSender)
 



