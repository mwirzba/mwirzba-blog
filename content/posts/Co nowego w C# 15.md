+++
title = 'Co nowego w C# 15'
+++

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

## 5. Etykietowane `break` i `continue`

Etykietowane `break` i `continue` (labeled break and continue) w C# 15 pozwalają nadać etykietę zewnętrznej pętli lub instrukcji `switch`, a następnie odwołać się do niej bezpośrednio z zagnieżdżonego bloku. Dzięki temu nie trzeba tworzyć pomocniczych flag typu `bool`, używać kontrowersyjnego `goto` ani wydzielać sztucznych metod lokalnych.

### Przed C# 15 (do C# 14)

Aby przerwać zewnętrzną pętlę z wnętrza zagnieżdżonej pętli albo wyjść z pętli z wnętrza instrukcji `switch`, trzeba było użyć flagi logicznej sprawdzanej na każdym poziomie lub instrukcji `goto`:

```csharp
// Sposób 1: pomocnicza flaga bool
bool found = false;
for (int row = 0; row < grid.Height && !found; row++)
{
    for (int col = 0; col < grid.Width; col++)
    {
        if (grid[row, col].IsBlocked)
        {
            // Aby przejść do kolejnego wiersza, trzeba było użyć break dla wewnętrznej pętli
            break;
        }

        if (grid[row, col].IsGoal)
        {
            found = true;
            break;
        }
    }
}

// Sposób 2: instrukcja goto z etykietą za pętlą
for (int row = 0; row < grid.Height; row++)
{
    for (int col = 0; col < grid.Width; col++)
    {
        if (grid[row, col].IsBlocked) goto NextRow;
        if (grid[row, col].IsGoal) goto EndSearch;
    }

    NextRow: ;
}
EndSearch: ;
```

### Po wprowadzeniu C# 15

Etykietę umieszcza się bezpośrednio przed zewnętrzną pętlą (`outer:`). W jej wnętrzu można użyć `break outer;` lub `continue outer;`:

```csharp
outer: for (int row = 0; row < grid.Height; row++)
{
    for (int col = 0; col < grid.Width; col++)
    {
        if (grid[row, col].IsBlocked)
        {
            // Pomija pozostałe kolumny i przechodzi do kolejnego wiersza
            continue outer;
        }

        if (grid[row, col].IsGoal)
        {
            // Przerywa pętlę 'outer' i wychodzi z obu pętli
            break outer;
        }
    }
}
```

### Wyjście z pętli z wnętrza `switch`

Etykiety rozwiązują też częsty problem z instrukcją `switch` umieszczoną wewnątrz pętli. Zwykłe `break` kończy wtedy tylko działanie `switch`, a nie całej pętli:

```csharp
commandLoop: foreach (var command in commands)
{
    switch (command)
    {
        case "skip":
            continue commandLoop; // Przechodzi do następnej komendy
        case "exit":
            break commandLoop; // Przerywa całą pętlę foreach
    }
}
```

Więcej informacji: [dokumentacja Microsoft](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/statements/jump-statements).

## 6. Statyczne składowe interfejsów w starszych środowiskach uruchomieniowych

C# 15 pozwala definiować niewirtualne statyczne składowe interfejsów także na starszych platformach (sprzed .NET Core 3.0). Może się to przydać w starszych projektach.

```csharp
public interface IMeasurement
{
    public static double ToMeters(double centimeters) => centimeters / 100;
}
```

Więcej informacji: [dokumentacja Microsoft](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/interface#static-interface-members-and-runtime-support).

## 7. Bezpieczeństwo pamięci (memory safety)

Wisienką na torcie są tematy związane z zarządzaniem pamięcią i jej bezpieczeństwem. Sam nie korzystałem z takich funkcjonalności w projektach komercyjnych, przy których pracowałem, ale zostawiam je tutaj jako ciekawostkę.

Ulepszenia bezpieczeństwa pamięci w C# 15 to funkcja w wersji preview, która rozpoczyna wieloletnią przebudowę słowa kluczowego `unsafe`. Jej celem jest powiązanie `unsafe` z faktycznym odczytem i zapisem niezarządzanej pamięci, gdzie mogą powstawać błędy bezpieczeństwa, a nie z samym używaniem typów wskaźnikowych. Zmiana ma też przekształcić `unsafe` w jawny kontrakt z wywołującym (caller).

### Co zmienia się w porównaniu z wcześniejszymi wersjami C#?

### 1. Rozdzielenie tworzenia wskaźnika od jego użycia

- **Przed C# 15:** samo zadeklarowanie wskaźnika (`int* p`), pobranie adresu (`&x`), użycie `fixed` lub `sizeof` wymagało bloku `unsafe`. Przez to całe duże metody stawały się `unsafe` i trudne do audytu.
- **W C# 15:** samo utworzenie lub przekazanie wskaźnika nie jest niebezpieczne. Ryzyko pojawia się dopiero przy jego dereferencji (odczycie lub zapisie pod adresem). Blok `unsafe` jest wymagany tylko tam, gdzie faktycznie korzysta się z pamięci:

```csharp
// W nowym modelu (preview) tworzenie wskaźnika i pinning nie wymagają bloku unsafe:
int number = 42;
int* ptr = &number;

int[] buffer = [10, 20, 30];
fixed (int* first = buffer)
{
    // Dopiero faktyczny odczyt lub zapis wymaga bloku unsafe
    // (łatwego do znalezienia podczas audytu)
    int val = unsafe(*first); // Nowe wyrażenie unsafe(...) dla pojedynczej instrukcji
}
```

### 2. `unsafe` jako kontrakt z wywołującym (`requires-unsafe`)

- **Przed C# 15:** dodanie `unsafe` do sygnatury metody oznaczało tylko, że można w niej używać wskaźników. Nie nakładało to żadnych wymagań na kod wywołujący, jeśli w sygnaturze nie było `*` (np. przy przekazywaniu adresu jako `nint` lub `Span<T>`).
- **W C# 15:** oznaczenie metody słowem `unsafe` nadaje jej status `requires-unsafe`. Oznacza to, że metoda ma niebezpieczne założenia wstępne, a jej wywołanie wymaga bloku `unsafe`. Dzięki temu można śledzić ryzyko w całym projekcie, podobnie jak w języku Rust. Samo słowo `unsafe` w sygnaturze nie czyni już automatycznie całego ciała metody niebezpiecznym — ryzykowne operacje nadal trzeba jawnie umieścić w bloku `unsafe { ... }`.

### 3. Nowe konstrukcje składniowe

- **Wyrażenie `unsafe(wyrażenie)`:** pozwala oznaczyć jako niebezpieczne pojedyncze wyrażenie, np. w inicjalizatorach pól, konstruktorów lub filtrach `catch`, gdzie nie da się użyć klasycznego bloku `{ ... }`.
- **Kontekstowe słowo kluczowe `safe`:** w nowym modelu zewnętrzne funkcje natywne (`extern`) oraz pola w strukturach o jawnym układzie pamięci (`[StructLayout(LayoutKind.Explicit)]`) są domyślnie traktowane jako niebezpieczne. Słowo `safe` pozwala jawnie potwierdzić, że dana deklaracja przeszła audyt i jest bezpieczna dla wywołującego.

### Jak włączyć tę funkcję (tylko do testów — wersja preview)

To funkcja eksperymentalna, planowana do pełnego wdrożenia w C# 16 / .NET 12. Aby ją przetestować, trzeba ją jawnie włączyć w pliku `.csproj`:

```xml
<PropertyGroup>
    <LangVersion>preview</LangVersion>
    <Features>$(Features);updated-memory-safety-rules</Features>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
</PropertyGroup>
```

Więcej informacji: [dokumentacja Microsoft](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/unsafe-code#the-updated-memory-safety-model-preview).

## Dalsza lektura

Więcej informacji na oficjalnym blogu Microsoftu: [Co nowego w C# 15](https://learn.microsoft.com/pl-pl/dotnet/csharp/whats-new/csharp-15).
