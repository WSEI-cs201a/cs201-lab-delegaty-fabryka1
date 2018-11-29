# Fabryka lodówek

* Opracowanie: _Krzysztof Molenda_
* Wersja: _2018-11-29_

Twoim zadaniem jest oprogramowanie systemu kontroli produkcji w fabryce lodówek.

Linia produkcyjna fabryki jest całkowicie zautomatyzowana (zrobotyzowana). Składa się ona z wielu maszyn, każda realizująca określone zadanie, np. tłoczenie obudowy lodówki z blachy, spawanie, malowanie, montaż.

Każda maszyna opracowana i instalowana jest przez innego producenta.

Maszyny kontrolowane są przez system komputerowy.
Każdy producent dostarczył oprogramowanie maszyny, zawierające zestaw funkcji, które umożliwiają jej kontrolowanie.

Twoim zadaniem głównym jest zintegrowanie różnych systemów używanych przez maszyny w ramach jednego programu kontrolującego.

---
Celem głównym ćwiczenia jest zapoznanie się z koncepcją typu `delegate` oraz jednym z jego zastosowań -- tworzeniem kodu otwartego, elastycznego, łatwo rozszerzalnego. 

Referencje:

* [Extension Methods (C# Programming Guide)](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/extension-methods)

* [Delegates (C# Programming Guide)](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/delegates/)

* [Action Delegate](https://docs.microsoft.com/en-us/dotnet/api/system.action?view=netframework-4.7.2)

---

## Zadanie 1

**Zadanie bieżące**: dostarczyć rozwiązanie umożliwiające wyłączenie całej linii produkcyjnej (wszystkich maszyn) możliwie szybko. Rozwiązanie ma być otwarte na ewentualne zmiany konfiguracji linii produkcyjnej (np. dodanie nowych maszyn, ...).

Każda z maszyn, oprócz jej właściwych funkcjonalności ma zaimplementowaną metodę bezpiecznie wyłączającą maszynę, ale każdy z producentów dostarczył ją pod inną nazwą i z inną sygnaturą, np.

- `void PressingStop()` - zatrzymanie robota tłoczącego obudowy
- `void StopFolding()` - zatrzymanie robota montującego obudowy
- `bool FinishWelding(int)` - zatrzymanie robota spawalniczego
- `int PaintOff()` - wyłączenie robota malującego

**Ważne**: nie masz wpływu na dostarczone oprogramowanie poszczególnych maszyn.

### Kod klas opisujących maszyny

#### Plik: `WeldingMachine.cs`

````csharp
sealed public class WeldingMachine
{
    // ... other methods
    public bool FinishWelding(int time)
    {
        Console.WriteLine($" ... stopping welding machine in {time} sec");
        Console.WriteLine("welding machine is stopped!");
        return true;
    }
}
````

#### Plik: `PressingMachine.cs`

````csharp
sealed public class PressingMachine
{
    // ... other methods
    public void PressingStop()
    {
        Console.WriteLine("pressing machine is stopped");
    }
}
````

#### Plik: `PaintingMachine.cs`

````csharp
sealed public class PaintingMachine
{
    // ... other methods
    public int PaintOff()
    {
        Console.WriteLine("painting is stopped");
        return 0;
    }
}
````

#### Plik: `FoldingMachine.cs`

````csharp
sealed public class FoldingMachine
{
    // ... other methods
    public void StopFolding()
    {
        Console.WriteLine("folding machine - stopped");
    }
}
````

-----------------------------------------------

### Podejście 1 - naiwne

Tworzysz klasę `Controller`:

````csharp
public class Controller
{
    private PressingMachine presser = new PressingMachine();
    private WeldingMachine welder = new WeldingMachine();
    private PaintingMachine painter = new PaintingMachine();
    private FoldingMachine folder = new FoldingMachine();

    // ...

    public void ShutDown()
    {
        presser.PressingStop();
        welder.FinishWelding(time: 0);
        folder.StopFolding();
        painter.PaintOff();
    }
}
````

Rozwiązanie mało elastyczne. Jeśli fabryka dołączy do linii produkcyjnej nową maszynę, musisz zmodyfikować kod klasy `Controller`.

-----------------------------------------------

### Podejście 2 - delegaty

Deklarujemy postać metody wyłączającej robota (typ delegatu) oraz tworzymy zmienną tego typu delegatu:

````csharp
delegate void StopMachineryDelegateType(); //typ delegatu
private StopMachineryDelegateType stopMachinery; //zmienna typu delegatu
````

Ponieważ nie wszystkie metody wyłączające maszyny mają sygnatury pasujące do typu delegatu, konieczne jest ich ujednolicenie. Można to zrobić np. dostarczając dla każdej maszyny odpowiednich metod rozszerzających:

````csharp
// method adapter's
public static class MachineryExtension
{
    public static void ShutDown( this PaintingMachine painter)
    {
        painter.PaintOff();
    }

    public static void ShutDown( this WeldingMachine welder)
    {
        welder.FinishWelding(0);
    }
}
````

W konstruktorze rejestrujemy metody wyłączające roboty (_multicast delegate_):

````csharp
public Controller()
{
    this.stopMachinery = presser.PressingStop;
    this.stopMachinery += welder.ShutDown;
    this.stopMachinery += folder.StopFolding;
    this.stopMachinery += painter.ShutDown;
}
````

Tworzymy metodę wyłączającą wszystkie roboty:

````csharp
public void ShutDownMachinery()
{
    this.stopMachinery();
}
````

W programie głównym:

````csharp
Controller c = new Controller();
c.ShutDownMachinery();
````

-----------------------------------------------

### Podejście 3 - uniezależniamy się od konstruktora

Rozwiązanie nadal jest nieelastyczne. Rozbudowa linii produkcyjnej zmusza nas do modyfikacji kodu klasy `Controller`.

Aby się od tego uniezależnić, zmieniamy widoczność zmiennej `stopMachinery` na `public` (oraz równocześnie typ delegatu czynimy publicznym i przenosimy poza klasę `Controller`). Musimy również udostępnić obiekty poszczególnych maszyn (np. `public get`).

W programie głównym rejestrujemy metody

````csharp
c.stopMachinery = c.Presser.PressingStop;
c.stopMachinery += c.Welder.ShutDown;
c.stopMachinery += c.Folder.StopFolding;
c.stopMachinery += c.Painter.ShutDown;
````

Jeszcze inaczej:

Może lepiej będzie `stopMachinery` uczynić properties `public get set`.

-----------------------------------------------

### Podejście 4 - lepsze od podejścia 3

Zmienną `stopMachinery` pozostawiamy `private`, zapewniając pełną hermetyzację.

Następnie możemy:

1. Usunąć maszyny z `Controller`
2. Dodać listę maszyn
3. Dodać metody `Add` oraz `Remowe` dodające i usuwające metody wyłączające roboty z `invocation list`.

````csharp
public delegate void StopMachineryDelegateType();

public class Controller1
{
    private ArrayList listaMaszyn = new ArrayList();

    private StopMachineryDelegateType stopMachinery;


    public void AddMachine(object machine, StopMachineryDelegateType stoppingProcedure)
    {
        listaMaszyn.Add(machine);
        this.stopMachinery += stoppingProcedure;
    }

    public void RemoveMachine(object machine, StopMachineryDelegateType stoppingProcedure)
    {
        listaMaszyn.Add(machine);
        this.stopMachinery -= stoppingProcedure;
    }

    public void ShutDownMachinery() => this.stopMachinery();

    public ArrayList ListaMaszyn => ArrayList.ReadOnly(listaMaszyn);
    public ReadOnlyCollection<Delegate> ListaProcedur => Array.AsReadOnly<Delegate>( stopMachinery.GetInvocationList() );
}
````

-----------------------------------------------

### Podejście 5 - zamiast jawnie nazwanego delegatu - [Action](https://docs.microsoft.com/en-us/dotnet/api/system.action?view=netframework-4.7.2)

````csharp
public class Controller2
{
    private ArrayList listaMaszyn = new ArrayList();
    private Action stopMachinery;

    public void AddMachine(object machine, Action stoppingProcedure)
    {
        listaMaszyn.Add(machine);
        this.stopMachinery += stoppingProcedure;
    }

    public void RemoveMachine(object machine, Action stoppingProcedure)
    {
        listaMaszyn.Add(machine);
        this.stopMachinery -= stoppingProcedure;
    }

    public void ShutDownMachinery() => this.stopMachinery();

    public ArrayList ListaMaszyn => ArrayList.ReadOnly(listaMaszyn);
    public ReadOnlyCollection<Delegate> ListaProcedur => Array.AsReadOnly<Delegate>( stopMachinery.GetInvocationList() );
}
````

## Zadanie 2

W opracowaniu ...

Maszyny mogą pracować niezawodnie przy ściśle określonych zakresach temperatury otoczenia. Przekroczenie tego zakresu powinno spowodować natychmiastowe wyłączenie maszyny.

Np. przyjmijmy, że robot malujący (`PainterMachine`) może poprawnie działać w zakresie temperatury otoczenia od 4 do 30 stopni C. Przekroczenie tego zakresu powinno skutkować natychmiastowym wyłączeniem robota.

W celu zabezpieczenia linii produkcyjnej 