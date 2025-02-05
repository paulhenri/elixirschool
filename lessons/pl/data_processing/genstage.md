%{
  version: "1.1.1",
  title: "GenStage",
  excerpt: """
  W tej lekcji przyjrzymy się z bliska GenStage — jaką pełni funkcję i jaki może mieć wpływ na nasze aplikacje.
  """
}
---

## Wstęp

Zatem czym jest GenStage? Oficjalna dokumentacja mówi, że jest to „specification and computational flow”, co można rozumieć jako silnik reguł obliczeniowych, ale co to w praktyce oznacza?

Oznacza to, że GenStage pozwala nam na zdefiniowanie potoków pracy, podzielonych na niezależne kroki (etapy), które będą obsługiwane przez różne procesy; jeżeli wcześniej pracowałeś z potokami, to wiele przedstawionych tu elementów powinno być Ci znajomych.

By lepiej zrozumieć jak, to wszystko działa przyjrzyjmy się prostemu potokowi producent-konsument:

```
[A] -> [B] -> [C]
```

W tym przykładzie mamy trzy etapy: `A` producenta, `B` producenta-konsumenta oraz `C` konsumenta.
`A` produkuje wartości konsumowane przez `B`, `B` wykonuje pewne zadania, produkując wartość konsumowaną przez `C`; rola poszczególnych etapów jest kluczowa, co za chwilę zobaczymy.

Nasz przykład to prosty układ 1 do 1, ale możemy tworzyć systemy z wieloma producentami i wieloma konsumentami na dowolnym etapie.

By lepiej zilustrować ten mechanizm, stworzymy potok z użyciem GenStage, ale zanim do tego przystąpimy, musimy poznać koncepcję ról, z której korzysta GenStage.

## Konsumenci i producenci

Jak już wiemy, role przypisywane do poszczególnych etapów są bardzo ważne.
Specyfikacja GenStage definiuje trzy role:

+ `:producer` — źródło.
Producent oczekuje na zapotrzebowanie zgłoszone przez konsumentów i odpowiada żądanymi zdarzeniami.

+ `:producer_consumer` — zarówno źródło, jak i cel.
Producent-konsument potrafi odpowiadać na żądania innych konsumentów, jak i samemu żądać zdarzeń od innych producentów.

+ `:consumer` — cel.
Wysyła żądania do producentów oraz przetwarza otrzymane dane.

Co znaczy, że producenci __oczekują__ na żądania? Konsumenci GenStage wysyłają strumień żądań i następnie przetwarzają dane otrzymane od producentów.
Mechanizm ten znany jest jako ciśnienie wsteczne (ang. _back-pressure_).
Ciśnienie wsteczne przenosi ciężar na producentów, chroniąc aplikację przed przeciążeniem, gdy konsumenci są zajęci.

Gdy wiemy już, czym są role w GenStage, możemy rozpocząć pisanie aplikacji.

## Pierwsze kroki

W tym przykładzie będziemy budować aplikację GenStage, która generuje strumień liczb, odfiltrowuje numery parzyste i na końcu je wypisuje.

Użyjemy wszystkich trzech ról GenStage.
Producent będzie odliczać i emitować kolejne numery.
Użyjemy producenta-konsumenta do odfiltrowania numerów parzystych oraz późniejszej odpowiedzi na żądanie.
Na końcu stworzymy konsumenta, aby wyświetlić wybrane liczby na ekranie.

Naszą pracę zaczniemy od stworzenia projektu z wykorzystaniem drzewa nadzoru:

```shell
$ mix new genstage_example --sup
$ cd genstage_example
```

Zmieńmy `mix.exs`, dodając zależność `gen_stage`:

```elixir
defp deps do
  [
    {:gen_stage, "~> 1.0.0"},
  ]
end
```

Zanim przejdziemy dalej, musimy pobrać i skompilować zależności:

```shell
$ mix do deps.get, compile
```

I już jesteśmy gotowi by stworzyć naszego producenta!

## Producent

Pierwszym krokiem w naszej aplikacji jest stworzenie producenta.
Tak jak wcześniej zaplanowaliśmy, chcemy stworzyć producenta, który będzie emitował stały strumień liczb.
Stworzymy zatem plik modułu producenta:

```shell
$ touch lib/genstage_example/producer.ex
```

Teraz możemy dodać kod:

```elixir
defmodule GenstageExample.Producer do
  use GenStage

  def start_link(initial \\ 0) do
    GenStage.start_link(__MODULE__, initial, name: __MODULE__)
  end

  def init(counter), do: {:producer, counter}

  def handle_demand(demand, state) do
    events = Enum.to_list(state..(state + demand - 1))
    {:noreply, events, state + demand}
  end
end
```

Dwa najważniejsze elementy, na które trzeba zwrócić uwagę, to `init/1` i `handle_demand/2`.
W funkcji `init/1` ustawiamy stan początkowy, tak jak w GenServerach, ale — co ważniejsze  — określamy też nasz moduł jako producenta.
Odpowiedź z `init/1` jest tym, na czym polega GenStage klasyfikując proces.

W funkcji `handle_demand/2` jest centrum logiki producenta.
Musi ona być zaimplementowana we wszystkich producentach GenStage.
To tutaj zwracamy zbiór numerów żądanych przez konsumentów i zwiększamy licznik.
Żądanie ze strony konsumentów, parametr `demand`, odpowiada liczbie całkowitej określającej maksymalną liczbę zdarzeń, której mogą oni podołać; domyślnie jest to 1000.

## Producent-konsument

Teraz, gdy mamy już naszego producenta generującego liczby, możemy zająć się producentem-konsumentem.
Będzie on żądać od producenta zbioru liczb, wybierać tylko parzyste i odsyłać tak odfiltrowany zbiór na żądanie konsumenta.

```shell
$ touch lib/genstage_example/producer_consumer.ex
```

Po utworzeniu pliku dodajmy kod:

```elixir
defmodule GenstageExample.ProducerConsumer do
  use GenStage

  require Integer

  def start_link(_initial) do
    GenStage.start_link(__MODULE__, :state_doesnt_matter, name: __MODULE__)
  end

  def init(state) do
    {:producer_consumer, state, subscribe_to: [GenstageExample.Producer]}
  end

  def handle_events(events, _from, state) do
    numbers =
      events
      |> Enum.filter(&Integer.is_even/1)

    {:noreply, numbers, state}
  end
end
```

Jak łatwo zauważyć, w funkcji `init/1` dodaliśmy nową opcję oraz zdefiniowaliśmy funkcję `handle_events/3`.
Za pomocą opcji `subscribe_to` instruujemy GenStage, że chcemy komunikować się z określonym producentem.

Funkcja `handle_events/3` to nasz koń pociągowy, to tutaj obsługujemy przychodzące zdarzenia, przetwarzamy je i zwracamy nasz zaktualizowany zbiór.
Jak widać, konsumenci są implementowani w podobny sposób, ale ważną różnicą jest to, co zwraca funkcja `handle_events/3` i to, jak jest używana.
Jeżeli oznaczymy nasz proces jako producenta-konsumenta, drugi argument w krotce — tu `numbers` — zostanie wykorzystany do odpowiadania na żądania.
W przypadku konsumentów będzie on zignorowany.

## Konsument

No i na końcu mamy naszego konsumenta.
Zacznijmy od utworzenia pliku:

```shell
$ touch lib/genstage_example/consumer.ex
```

Jako że producent-konsument i konsument są bardzo podobne, to kod nie będzie się różnił w znaczący sposób:

```elixir
defmodule GenstageExample.Consumer do
  use GenStage

  def start_link(_initial) do
    GenStage.start_link(__MODULE__, :state_doesnt_matter)
  end

  def init(state) do
    {:consumer, state, subscribe_to: [GenstageExample.ProducerConsumer]}
  end

  def handle_events(events, _from, state) do
    for event <- events do
      IO.inspect({self(), event, state})
    end

    # Jako konsument nigdy nie emitujemy zdarzeń
    {:noreply, [], state}
  end
end
```

Jak wspomniano w poprzednim punkcie, nasz konsument nie emituje zdarzeń, więc drugi element w krotce zostanie zignorowany.

## Wszystko razem

teraz mamy już producenta, producenta-konsumenta i konsumenta, a zatem najwyższy czas, aby zebrać wszystkie te elementy w całość.

Najpierw w `lib/genstage_example/application.ex` dodajmy nasz proces do drzewa nadzoru:

```elixir
def start(_type, _args) do
  import Supervisor.Spec, warn: false

  children = [
    {GenstageExample.Producer, 0},
    {GenstageExample.ProducerConsumer, []},
    {GenstageExample.Consumer, []}
  ]

  opts = [strategy: :one_for_one, name: GenstageExample.Supervisor]
  Supervisor.start_link(children, opts)
end
```

Jeżeli wszystko prawidłowo zaimplementowaliśmy, możemy uruchomić nasz projekt i powinniśmy zobaczyć, że wszystko działa:

```shell
$ mix run --no-halt
{#PID<0.109.0>, 0, :state_doesnt_matter}
{#PID<0.109.0>, 2, :state_doesnt_matter}
{#PID<0.109.0>, 4, :state_doesnt_matter}
{#PID<0.109.0>, 6, :state_doesnt_matter}
...
{#PID<0.109.0>, 229062, :state_doesnt_matter}
{#PID<0.109.0>, 229064, :state_doesnt_matter}
{#PID<0.109.0>, 229066, :state_doesnt_matter}
```

Udało się! Nasza aplikacja, zgodnie z oczekiwaniami, wyświetla jedynie liczby parzyste i robi to bardzo __szybko__.

Mamy zatem działający potok.
Jest w nim producent emitujący liczby, producent-konsument odsiewający liczby nieparzyste oraz konsument, który je wyświetla i kontynuuje proces.

## Wielu producentów i konsumentów

Jak wspomnieliśmy na wstępie, możliwe jest stworzenie wielu producentów i konsumentów.
Przyjrzyjmy się temu.

Wykorzystując `IO.inspect/1` w naszym przykładzie możemy stwierdzić, że zdarzenia są obsługiwane przez jeden PID.
Zmieńmy `lib/genstage_example/application.ex` tak, by utworzyć wiele procesów do obsługi:

```elixir
children = [
  {GenstageExample.Producer, 0},
  {GenstageExample.ProducerConsumer, []},
  %{
    id: 1,
    start: {GenstageExample.Consumer, :start_link, [[]]}
  },
  %{
    id: 2,
    start: {GenstageExample.Consumer, :start_link, [[]]}
  },
]
```

W naszej konfiguracji mamy teraz dwóch konsumentów — sprawdźmy, co zobaczymy po uruchomieniu naszej aplikacji:

```shell
$ mix run --no-halt
{#PID<0.120.0>, 2, :state_doesnt_matter}
{#PID<0.121.0>, 4, :state_doesnt_matter}
{#PID<0.120.0>, 6, :state_doesnt_matter}
{#PID<0.120.0>, 8, :state_doesnt_matter}
...
{#PID<0.120.0>, 86478, :state_doesnt_matter}
{#PID<0.121.0>, 87338, :state_doesnt_matter}
{#PID<0.120.0>, 86480, :state_doesnt_matter}
{#PID<0.120.0>, 86482, :state_doesnt_matter}
```

Jak widać mamy dwa PIDy, a dodaliśmy raptem jedną linię i nadaliśmy naszym konsumentom identyfikatory.

## Przypadki użycia

Stworzyliśmy naszą pierwszą, prostą aplikację GenStage — ale jakie _rzeczywiste_ zastosowania ma to rozwiązanie?

+ Potoki transformacji danych — producenci nie są tu prostymi generatorami liczb.
+ Możemy generować dane, wykorzystując bazy danych czy też rozwiązania w rodzaju Apache Kafka.
Łącząc wielu producentów-konsumentów i konsumentów możemy przetwarzać, sortować, katalogować różne dane.

+ Kolejki — zdarzenia mogą być różnorodne, a naszym zadaniem jest ich obsługa za pomocą szeregu konsumentów.

+ Obsługa zdarzeń — zbliżona do obsługi danych, lecz tym razem przetwarzamy, sortujemy i obsługujemy zdarzenia generowane w czasie rzeczywistym.

A to tylko __kilka__ z wielu zastosowań GenStage.
