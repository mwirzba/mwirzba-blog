# Co nowego w C# 15?

## Wprowadzenie

10 listopada ukazała się wersja C# 15 wraz z platformą .NET 11.

W tej wersji pojawiło się kilka ciekawych usprawnień.

## 1. Argumenty wyrażeń kolekcji

To niewielkie usprawnienie ułatwia inicjalizowanie kolekcji.

Dotąd, aby przekazać parametry podczas inicjalizacji kolekcji, trzeba było użyć `new`:

```csharp
string[] source = ["a", "b", "c"];

// 1. Pojemność z wyrażeniem kolekcji wymagała osobnego kroku lub zrezygnowania z [...]
List<string> listBefore = new(source.Length * 2);
listBefore.AddRange(source);

// 2. Komparator wymagał jawnego 'new' zamiast wyrażenia kolekcji
HashSet<string> setBefore = new(StringComparer.OrdinalIgnoreCase);
setBefore.UnionWith(source);
```

W C# 15 można przekazać te parametry za pomocą `with(...)`:

```csharp
string[] source = ["a", "b", "c"];

// Przekazanie pojemności do konstruktora List<T>
List<string> listAfter = [with(capacity: source.Length * 2), .. source];

// Przekazanie komparatora do konstruktora HashSet<T>
HashSet<string> setAfter = [with(StringComparer.OrdinalIgnoreCase), ..source];
```

## 2. Typ union

Słowo kluczowe `union` pozwala zdefiniować zbiór typów, które dziedziczą po `System.Object`.

Przykład:

```csharp
public record class Cat(string Name);
public record class Dog(string Name);
public record class Bird(string Name);

public union Pet(Cat, Dog, Bird);
```

Stworzona unia implementuje interfejs `IUnion`:

```csharp
public interface IUnion
{
    object? Value { get; }
}
```

Na przykład unia może łączyć różne typy opisujące wynik logowania:

```csharp
// Typy nie muszą po niczym dziedziczyć
public sealed record LoggedIn(string Username);
public sealed record InvalidCredentials(int AttemptsLeft);
public sealed record AccountLocked;

// Deklaracja unii
public union LoginResult(LoggedIn, InvalidCredentials, AccountLocked);

// Niejawna konwersja z każdego typu składowego do unii
LoginResult result = new LoggedIn("Jan");

// Wyczerpujący switch bez gałęzi '_' — dodanie nowego typu do unii od razu
// ostrzeże o brakującej obsłudze
public static string HandleLogin(LoginResult result) => result switch
{
    LoggedIn u => $"Witaj, {u.Username}",
    InvalidCredentials e => $"Błędne dane. Pozostało prób: {e.AttemptsLeft}",
    AccountLocked => "Konto zablokowane"
};
```

Więcej o uniach: [dokumentacja Microsoft](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/union).

## 3. Zamknięte hierarchie

W C# 15 pojawiło się nowe słowo kluczowe `closed`.

Po zamkniętej klasie można dziedziczyć wyłącznie w ramach tego samego zestawu (assembly). Dzięki temu można wymusić hermetyzację pomiędzy modułami, nawet jeśli odwołują się do siebie nawzajem. Wcześniej, jeśli klasa była oznaczona słowem kluczowym `public` i nie była oznaczona `sealed`, takie dziedziczenie było możliwe.

Przykład użycia:

```csharp
public closed record class GateState;
public record class Closed : GateState;
public record class Open(float Percent) : GateState;
```

Ponieważ kompilator zna wszystkich potomków klasy `GateState`, wyrażenie `switch` bez wariantu `=> default` nie generuje błędów:

```csharp
string Describe(GateState state) => state switch
{
    Closed => "closed",
    Open(var percent) => $"{percent}% open",
    // No warning: every direct descendant of 'GateState' is handled.
};
```

Klasa oznaczona słowem kluczowym `closed` jest z definicji abstrakcyjna. Nie można więc łączyć słowa `closed` ze słowem `abstract`. Nie można też używać `sealed` ani `static`, ponieważ uniemożliwiają dziedziczenie.

Słowo kluczowe `closed` nie przechodzi na typy potomne. Jeśli typu dziedziczącego po klasie `closed` nie oznaczymy jako `closed`, klasy w innym assembly będą mogły po nim dziedziczyć.

Więcej informacji o `closed`: [dokumentacja Microsoft](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/closed).

## 4. Indeksery rozszerzeń (extension indexers)

C# 15 wprowadza indeksatory rozszerzeń (extension indexers). Pozwalają one dodać do istniejącego typu składnię indeksatora, np. `obiekt[indeks]`, bez zmiany jego kodu źródłowego. To uzupełnienie mechanizmu bloków `extension` z C# 14, który wcześniej nie obsługiwał indeksatorów.

### Przed C# 15 (do C# 14)

Gdy typ z biblioteki standardowej lub zewnętrznej nie udostępnia indeksatora (np. `IEnumerable<T>`) lub potrzebny jest dodatkowy sposób indeksowania (np. po kilku parametrach), pozostają zwykłe metody rozszerzające albo klasy opakowujące (wrappery):

```csharp
public static class SequenceExtensionsBefore
{
    public static T GetAt<T>(this IEnumerable<T> source, int index)
        => source.ElementAt(index);

    public static string GetShift(
        this IReadOnlyDictionary<(DayOfWeek, int), string> rota,
        DayOfWeek day,
        int priority)
        => rota[(day, priority)];
}
```

W efekcie trzeba było wywoływać metodę po jej nazwie, zamiast używać składni indeksatora `[]`:

```csharp
IEnumerable<int> numbers = Enumerable.Range(10, 5);
int third = numbers.GetAt(2);
string team = rota.GetShift(DayOfWeek.Monday, 1);
```

### Po wprowadzeniu C# 15

Indeksator deklaruje się bezpośrednio w bloku `extension(...)` z użyciem `this[...]`. Dzięki temu można odwoływać się do obiektu tak, jakby indeksator był jego natywną częścią:

```csharp
public static class SequenceExtensionsAfter
{
    // Blok extension musi mieć nazwany parametr odbiorcy (np. 'source')
    extension<T>(IEnumerable<T> source)
    {
        public T this[int index] => source.ElementAt(index);
    }

    // Indeksator wieloparametrowy
    extension(IReadOnlyDictionary<(DayOfWeek, int), string> rota)
    {
        public string this[DayOfWeek day, int priority] => rota[(day, priority)];
    }
}
```

Dzięki temu można używać naturalnej składni indeksatora `[]`:

```csharp
IEnumerable<int> numbers = Enumerable.Range(10, 5);
int third = numbers[2];
string team = rota[DayOfWeek.Monday, 1];
```

Więcej informacji o extension indexers: [dokumentacja Microsoft](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/extension#extension-indexers).

## Dalsza lektura

Więcej informacji na oficjalnym blogu Microsoftu: [Co nowego w C# 15](https://learn.microsoft.com/pl-pl/dotnet/csharp/whats-new/csharp-15).
