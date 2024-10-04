## The Assignment 

### Zadanie zaliczeniowe ze współbieżności w języku Java
### Systemy składowania danych (ang. storage systems) muszą jednocześnie spełniać wiele wymagań dotyczących w szczególności wydajności operacji dostępu do danych, wykorzystania pojemności nośników danych oraz odporności na awarie urządzeń będących tymi nośnikami. W tym celu często przenoszą fragmenty danych pomiędzy poszczególnymi urządzeniami. Twoim zadaniem będzie implementacja w języku Java mechanizmów koordynujących współbieżne operacje takiego przenoszenia zgodnie z poniższymi wymaganiami. Do implementacji rozwiązania należy wykorzystać załączony szablon.

# Specyfikacja
W naszym modelu systemu dane są grupowane w komponenty i w takich jednostkach przechowywane na urządzeniach. Zarówno każde urządzenie jak i każdy komponent danych mają przypisany niezmienny i unikalny w ramach systemu identyfikator (obiekt klasy odpowiednio cp2023.base.DeviceId oraz cp2023.base.ComponentId). Każde urządzenie ma ponadto określoną pojemność, to jest maksymalną liczbę komponentów, które może przechowywać w dowolnym momencie. Przypisaniem komponentów do urządzeń zarządza system (obiekt implementujący interfejs cp2023.base.StorageSystem, przedstawiony poniżej):

```
public interface StorageSystem {

    void execute(ComponentTransfer transfer) throws TransferException;
    
}
```
Dokładniej, każdy istniejący w systemie komponent znajduje się na dokładnie jednym urządzeniu, chyba że użytkownik systemu zlecił transfer tego komponentu na inne urządzenie (wołając metodę execute ww. klasy StorageSystem i przekazując jej jako parametr obiekt implementujący interfejs cp2023.base.ComponentTransfer reprezentujący zlecany transfer).
```
public interface ComponentTransfer {

    public ComponentId getComponentId();
    
    public DeviceId getSourceDeviceId();
    
    public DeviceId getDestinationDeviceId();
    
    public void prepare();
    
    public void perform();

}
```
Transfer komponentu jest zlecany także, gdy użytkownik chce dodać nowy komponent do systemu (w takim przypadku metoda getSourceDeviceId obiektu transferu zwraca wartość null) lub usunąć istniejący komponent z systemu (w takim przypadku, symetrycznie, metoda getDestinationDeviceId obiektu transferu zwraca wartość null). Innymi słowy, pojedynczy transfer reprezentuje jedną z trzech dostępnych operacji na komponencie:

* dodanie nowego komponentu na urządzenie systemu (getSourceDeviceId obiektu transferu zwraca null a getDestinationDeviceId nie-null oznaczający identyfikator urządzenia, na którym ma znaleźć się dodawany komponent),
* przeniesienie istniejącego komponentu pomiędzy urządzeniami systemu (getSourceDeviceId oraz getDestinationDeviceId obie zwracają nie-null-e reprezentujące identyfikatory odpowiednio aktualnego urządzenia, na którym znajduje się komponent, oraz docelowego urządzenia, na którym komponent powinien się znaleźć po transferze),
* usunięcie istniejącego komponentu z urządzenia a tym samym – systemu (getSourceDeviceId zwraca nie-null oznaczający identyfikator urządzenia, na którym znajduje się komponent, a getDestinationDeviceId zwraca null).
Zlecanie trzech ww. typów operacji przez użytkownika jest poza kontrolą implementowanego rozwiązania. Zadaniem Twojego rozwiązania jest natomiast przeprowadzanie zlecanych transferów w sposób synchroniczny (tj. jeśli zlecany transfer jest poprawny, metoda execute wywołana na obiekcie implementującym StorageSystem z transferem reprezentowanym jako parametr implementujący interfejs ComponentTransfer nie może zakończyć swojego działania dopóki ten transfer się nie zakończy). Jako że wiele różnych operacji może być zlecanych jednocześnie przez użytkownika, implementowany przez Ciebie system musi zapewnić ich koordynację według następujących reguł.

W dowolnym momencie dla danego komponentu może być zgłoszony co najwyżej jeden transfer. Dopóki ten transfer się nie zakończy, komponent nazwiemy transferowanym zaś każdy kolejny transfer zgłaszany dla tego komponentu należy traktować jako niepoprawny.

Sam transfer komponentu jest dwuetapowy i może trwać dłuższy czas (zwłaszcza jego drugi etap). Rozpoczęcie transferu polega na jego przygotowaniu (tj. wywołaniu metody prepare na obiekcie o interfejsie ComponentTransfer reprezentującym transfer). Dopiero po takim przygotowaniu mogą być przesyłane dane stanowiące komponent (co odbywa się poprzez wywołanie metody perform dla ww. obiektu). Gdy dane zostaną przesłane (tj. metoda perform zakończy swoje działanie), transfer się kończy. Obie ww. metody muszą być wykonane w kontekście wątku zlecającego transfer.

### Bezpieczeństwo
Transfer może być poprawny albo niepoprawny. Poniższe wymagania bezpieczeństwa dotyczą transferów poprawnych. Obsługa transferów niepoprawnych jest z kolei opisana w dalszej sekcji.

Jeśli transfer reprezentuje operację usunięcia komponentu, jego rozpoczęcie jest dozwolone bez żadnych dodatkowych warunków wstępnych. W przeciwnym przypadku, rozpoczęcie transferu jest dozwolone o ile na urządzeniu docelowym jest miejsce na transferowany komponent, w szczególności miejsce to właśnie jest lub będzie zwalniane, więc można je dzięki temu zarezerwować. Dokładniej, rozpoczęcie transferu reprezentującego przesunięcie lub dodanie komponentu Cx jest dozwolone, jeśli zachodzi któryś z poniższych warunków:

* Na urządzeniu docelowym transferu jest wolne miejsce na komponent, które nie zostało zarezerwowane przez system na inny komponent, który jest/będzie przenoszony/dodawany na to urządzenie.
* Na urządzeniu docelowym znajduje się komponent Cy transferowany z tego urządzenia, którego transfer się rozpoczął lub jego rozpoczęcie jest dozwolone, oraz miejsce zwalniane przez ten komponent nie zostało zarezerwowane przez system na inny komponent.
* Komponent Cx należy do pewnego zbioru transferowanych komponentów takiego, że urządzeniem docelowym każdego komponentu ze zbioru jest urządzenie, na którym znajduje się dokładnie jeden inny komponent ze zbioru, oraz miejsce żadnego z komponentów ze zbioru nie zostało zarezerwowane na komponent spoza zbioru.
Jeśli transfer komponentu Cx jest dozwolony, ale odbywać się ma w miejsce jeszcze zajmowane przez inny transferowany komponent Cy (dwa ostatnie przypadki powyżej), to drugi etap transferu komponentu Cx (tj. wywołanie funkcji perform dla tego transferu) nie może rozpocząć się przed zakończeniem pierwszego etapu transferu komponentu Cy (tj. wywołaniem funkcji prepare dla tego transferu).

Oczywiście, jeśli transfer komponentu jest niedozwolony, to nie może on być rozpoczęty (tj. ani funkcja prepare, ani funkcja perform nie mogą zostać wywołane na obiekcie reprezentującym ten transfer).

Twoje rozwiązanie powinno bezwzględnie zapewniać wszystkie powyższe warunki bezpieczeństwa.

### Żywotność
Jeśli natomiast chodzi o żywotność, to transfer (zarówno jego faza prepare jak i perform) powinien się rozpoczynać tak szybko, jak tylko jest on dozwolony i pozostałe wymagania bezpieczeństwa zostaną spełnione. W przypadku, gdy kilka transferów konkuruje o miejsce na urządzeniu, spomiędzy tych z nich, które są dozwolone, Twój algorytm powinien lokalnie priorytetyzować transfery czekające dłużej na to urządzenie. Globalnie może to potencjalnie prowadzić do zagłodzenia pewnych transferów (zachęcamy do wymyślenia scenariusza takiej sytuacji). Rozwiązanie tego problemu jest oczywiście możliwe do implementacji, ale komplikuje kod ponad to, co chcielibyśmy od Państwa wymagać. Nie należy go więc implementować, zwłaszcza że w praktyce użytkownik systemu widząc, że transfer na jakieś urządzenie długo nie może się wykonać, mógłby wytransferować inne komponenty z tego urządzenia.

### Obsługa błędów
Wreszcie, zaproponowane rozwiązanie powinno sprawdzać, czy zlecony przez użytkownika transfer jest niepoprawny (co powinno skutkować podniesieniem przez metodę execute interfejsu StorageSystem odpowiedniego wyjątku dziedziczącego po klasie cp2023.exceptions.TransferException). Zgodnie z wcześniejszymi wyjaśnieniami transfer jest niepoprawny, jeśli zachodzi co najmniej jeden z poniższych warunków:

* transfer nie reprezentuje żadnej z trzech dostępnych operacji na komponentach lub nie wskazuje żadnego komponentu (wyjątek IllegalTransferType);
* urządzenie wskazane przez transfer jako źródłowe lub docelowe nie istnieje w systemie (wyjątek DeviceDoesNotExist);
* komponent o identyfikatorze równym dodawanemu w ramach transferu komponentowi już istnieje w systemie (wyjątek ComponentAlreadyExists);
* komponent o identyfikatorze równym usuwanemu lub przesuwanemu w ramach transferu komponentowi nie istnieje w systemie lub znajduje się na innym urządzeniu niż wskazane przez transfer (wyjątek ComponentDoesNotExist);
* komponent, którego dotyczy transfer, znajduje się już na urządzeniu wskazanym przez transfer jako docelowe (wyjątek ComponentDoesNotNeedTransfer);
* komponent, którego dotyczy transfer, jest jeszcze transferowany (wyjątek ComponentIsBeingOperatedOn).
W rozwiązaniu można przyjąć dowolną sensowną kolejność sprawdzania tych warunków.
