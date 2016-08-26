# Async pod maską - Michał Lowas-Rzechonek

## Wprowadzenie

Ostatnimi czasy wiele mówi się i pisze o programowaniu asynchronicznym i
pętlach zdarzeń. Pojawiają się takie słowa jak *event loop*, *promise*,
*future*, *deferred*, *greenlet* itp.

W tym artykule postaram się pokazać o co chodzi w pętli zdarzeń oraz że
koncepcja nie jest wcale nowa - nowatorska jest jedynie jej implementacja
za pomocą generatorów w module `asyncio`.

Zacznę od omówienia mechanizmów wykonywania na jednym komputerze kilku
programów na raz, potem pokrótce zademonstruję obsługę komunikacji przez sieć,
a na koniec poskładam wszystko w prosty, ale asynchroniczny i kompatybilny z
`asyncio`, serwer TCP.

## Współbieżność i przełączanie kontekstu

Żeby zacząć w ogóle rozmawiać o programowaniu asynchronicznym, wypadałoby się
najpierw zająć modelem wykonania programu na komputerze.

Dla uproszczenia załóżmy, że nasz komputer ma tylko jeden procesor, a
dokładniej jedną *jednostkę arytmetyczno-logiczną* (ALU - arithmetic-logic
unit). ALU to część procesora odpowiedzialna za wykonywanie instrukcji a jakich
składa się nasz program.

ALU nie jest w swoim działaniu zbyt wyrafinowana - po prostu wykonuje po kolei
instrukcje programu umieszczone w kolejnych komórkach pamięci. Ważne jest to,
że wykonuje je bez przerwy, tak szybko jak pozwala na to częstotliwość zegara.

Pojawia się zatem pytanie jak to możliwe, że na jednym komputerze wyposażonym w
jeden procesor działa jednocześnie kilka programów? W istocie, proste systemy
operacyjne (np. DOS) pozwalają uruchamiać jednocześnie tylko jeden program na
raz.

Mechanizmem który pozwala kilku programom działać jednocześnie jest tzw.
*scheduler*. Scheduler jest częścią systemu operacyjnego która nadzoruje
działanie ALU i decyduje który proces jest w danym momencie wykonywany.

Oznacza to, że "tak naprawdę" ALU ciągle wykonuje tylko jeden program na raz,
ale scheduler przełącza aktywne procesy tak szybko, że nikt nic nie widzi.
Skomplikowana nazwa na taki model to *współbieżność* (ang. *concurrency*).

Wykonywana przez scheduler podmiana aktywnego procesu nazywa się
*przełączaniem kontekstu* (ang. context switching).


## Wywłaszczanie

No dobrze, ale nasz *scheduler* też jest programem. Jak w takim razie go
uruchomić, skoro ALU zajęte jest wykonywaniem innego programu?

Można to zrobić na dwa sposoby: albo "siłą" odbierając procesorowi wykonywany
właśnie program, albo czekając aż tenże program jawnie zgłosi, że jest gotowy
na przełączenie kontekstu.

Pierwszy z tych sposobów to tzw. *wywłaszczanie* (ang. *preemption*). Żeby
wymusić przełączenie, scheduler korzysta z mechanizmu tzw. *przerwania
zegarowego* (ang. *timer interrupt*). Przerwanie to specjalny układ
elektroniczny, który pozwala zainstalować wewnątrz procesora niewielką funkcję,
która będzie uruchamiana przez ALU z zadaną częstotliwością, niezależnie od
głównego programu. Przerwanie, jak sama nazwa wskazuje, zatrzymuje "w pół
kroku" aktualnie wykonywany program, wykonuje zainstalowaną funkcję (nazywaną
*funkcją obsługi przerwania*, ang. *interrupt service routine*) a następnie wznawia
główny program jak gdyby nigdy nic.

Linux, dla przykładu, konfiguruje to przerwanie aby uruchamiać scheduler 100
razy na sekundę.

Drugi sposób wymaga, aby wszystkie(!) procesy co jakiś czas jawnie wołały
funkcję schedulera. Tym modelem zajmiemy się później.

Obie metody mają swoje wady i zalety. W ogólnym przypadku lepiej jest
wywłaszczać, bo wtedy jeden błędny program nie zawiesi całego systemu. Z
drugiej strony, programy mogą nie być przygotowane na przełączenie kontekstu w
dowolnym momencie (np. w połowie zapisywania danych do pliku), co może
powodować subtelne błędy związane z faktyczną kolejnością wykonywania operacji.

Dodatkowym minusem jest fakt, że przełączanie kontekstu jest operacją
stosunkowo czasochłonną, a zbyt częste jej wykonywanie zmniejsza wydajność
całego systemu, bo mniej czasu pozostaje na wykonanie faktycznych programów.

W Pythonie ten mechanizm dostępny jest za pośrednictwem *wątków* (ang.
*threads*). Jeśli w programie uruchomimy kilka wątków, to systm operacyjny
będzie je cyklicznie wywłaszczał i przełączał:

    import itertools
    import time

    from threading import Thread

    def this_is_a_thread(pid):
        for i in itertools.count():
            print(pid, i)
            time.sleep(0.5)


    threads =  [
        Thread(target=this_is_a_thread, args=(1,), daemon=True),
        Thread(target=this_is_a_thread, args=(2,), daemon=True),
    ]

    for t in threads:
        t.start()

    for t in threads:
        t.join()

Jeśli pozostawimy ten program uruchomiony przez jakiś czas, to zaobserwujemy,
że nasze dwa wątki nie zawsze uruchamiane są na zmianę - czasem pierwszy wątek
wykona kilka interacji swojej pętli zanim zostanie wywłaszczony. Co gorsza,
nasz program nie ma nad tym żadnej kontroli, bo w modelu wątkowym przełączaniem
kontekstu zajmuje się system operacyjny.

Ten brak determinizmu i kontroli bywa problematyczny, przyrzyjmy się zatem
drugiemu modelowi, czyli przełączaniu na żądanie.

## Współbieżność kooperacyjna

Patrząc na interpreter Pythona jak na system operacyjny, można zauważyć, że w
sam język wbudowany jest mechanizm przełączania kontekstu. Tym mechanizmem są
mianowicie generatory.

Dokładniej, użycie instrukcji `yield` zmienia funkcję w obiekt generatora (de
facto *proces*) który możemy uruchomić lub zatrzymać. To czego brakuje, to
scheduler, jednak bardzo łatwo napisać własny, chociażby przełączając procesy
po kolei:

    import itertools
    import time

    def this_is_a_process(pid):
        for i in itertools.count():
            print(pid, i)
            yield

    processes = [ this_is_a_process(1), this_is_a_process(2) ]

    while True:
        for process in processes:
            next(process)

Ten program wykonuje nasze dwa "procesy" na zmianę, a przełączanie następuje na
ich jawne żądanie - jeśli któryś nie zawierałby instrukcji `yield`, cały nasz
miniaturowy system operacyjny by się zawiesił wykonując w kółko tylko jego:

    def rogue_one(pid):
        yield # żeby stać się generatorem

        while True:
            print(pid, "I rebel!)
            # nie wołamy "yield" wewnątrz pętli

    processes = [ this_is_a_process(1), this_is_a_process(2), rogue_one(3) ]

    while True:
        for process in processes:
            next(process)

Taki model wykonania nazywa się *współbieżnością kooperacyjną* (ang.
*cooperative multitasking*). Przełączanie procesów jest deterministyczne, ale
wymaga aby programy napisane były w specyficzny sposób.

Spróbujmy nieco ułatwić pisanie programów poprawnie korzystających z tego modelu.

## Kompozycja generatorów

Jednym ze sposobów aby to zapewnić by program zachowywał się "grzecznie" w
modelu kooperacyjnym, jest uruchamianie schedulera za każdym razem gdy program
prosi system operacyjny o jakiś rodzaj operacji wejścia-wyjścia, np. o
odczytanie stanu klawiatury albo wyświetlenie czegoś na ekranie.  W naszym
przykładzie byłoby to wywołanie funkcji `print`:

    def os_print(*args, **kwargs):
        print(*args, **kwargs)
        # tutaj chcemy uruchomić scheduler

    def this_is_a_process(pid):
        for i in itertools.count():
            os_print(pid, i)

Pozostaje natomiast pytanie jak uruchomić scheduler. Jeśli po prostu zawołamy
jakąś funkcję, to po jej zakończeniu wrócimy najpierw do `os_print` a potem z
powrotem do tego samego `this_is_a_process`. Z kolei  jeśli w `os_print`
użyjemy `yield`, to przerwiemy funkcję `os_print` i znów wrócimy do tego samego
`this_is_a_process`. Potrzebujemy mechanizmu, który pozwoli poprosić inną
funkcję, aby wykonała `yield` za nas.

TODO: wkleić diagram

Jest to wykonalne, ale kod który realizuje taką operację jest niezwykle
skomplikowany, gdyż musi brać pod uwagę, że `os_print` może rzucić wyjątkiem,
zawierać kilka `yield`ów (np. w pętli) i tak dalej. Ostatecznie jest to ponad
30 linii kodu które musielibyśmy wklejać aby zawołać *każdą* funkcję
wejścia-wyjścia.

Python 3.4 rozwiązuje ten problem dostarczając dodatkowe słowo kluczowe `yield
from`, które wykonuje skomplikowane operacje za nas.

    def os_print(*args, **kwargs):
        print(*args, **kwargs)
        yield

    def this_is_a_process(pid):
        for i in itertools.count():
            yield from os_print(pid, i)

Dygresja: warto zauważyć, że `yield from` bywa użyteczne też w innych
kontekstach, na przykład gdy chcemy napisać rekurencyjny generator:

    def walk(dir):
        for i in os.listdir(dir):
            path = os.path.join(dir, i)
            yield path
            if os.path.isdir(path):
                yield from walk(path)

No dobrze, ale wyświetlanie rzeczy na ekranie nie jest zbyt ekscytujące, a o
programowaniu asynchronicznym rozmawia się zazwyczaj w konktekście obsługi
sieci. Żeby zobaczyć jak jedno ma się do drugiego, musimy odłożyć na chwilę
problem przełączania procesów, a zająć się zająć się obsługą wejścia-wyjścia na
poziomie systemu operacyjnego.

## Deskryptory plików i gniazda sieciowe

Praktycznie wszystkie systemy operacyjne udostępniają operacje wejścia-wyjścia
za pośrednictwem tzw. *deskryptorów plików*. Deskryptor jest zazwyczaj po
prostu liczbą którą dostajemy od systemu w momencie otwarcia pliku, a
przekazujemy go jako argument do funkcji które z tego pliku czytają bądź do
niego piszą.

W ten sposób deskryptor jednoznacznie identyfikuje plik który chcemy obsłużyć,
nie interesuje nas za to jego dokładna wartość (np. liczbowa). Coś jak bloczek
w szatni.

    import os

    file_descriptor = os.open('plik.txt', os.O_RDONLY)
    print(file_descriptor)
    print(os.read(file_descriptor, length=100))
    os.close(file_descriptor)

Python ukrywa przed nami takie szczegóły implementacyjne za pomocą obiektu
`file`, ale ciągle pozwala dostać się do deskryptora za pomocą metody
`fileno()`:

    with open('plik.txt') as file_object:
        file_descriptor = file_descriptor.fileno()
        print(file_descriptor)

Z naszego punktu widzenia intresujący jest fakt, że pod deskryptorem pliku może
kryć się dużo więcej niż tylko zwykłe pliki na dysku.  Deskryptor pliku jest
więc pewną abstrakcją pozwalającą odnosić się w ten sam sposób to różnych
urządzeń wejścia-wyjścia: dysku, karty sieciowej, a nawet ekranu.

Dzięki temu komunikację przez sieć obsługuje się za pomocą tego samego
mechanizmu co zwykłe pliki. Co prawda sposób uzyskania takiego sieciowego
deskryptora bardziej skomplikowany niż proste `open()`, ale sam odczyt i zapis
działają tak samo: `accept()` jest odpowiednikiem `open()`, `send()` to
`write()`, a `recv()` to `read()`.

Prosty serwer akceptujący połączenia na porcie 1234 wygląda tak:

    import os
    import socket

    server = socket.socket()
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind(('', 1234))
    server.listen(1)

    while True:
        connection, address = server.accept()
        print(connection.fileno(), "is a connection from", address)
        connection.send("Enter your name: ")
        name = connection.recv(100)
        connection.send("Hello, %s" % name)
        connection.close()

Działanie można sprawdzić np. programem `nc` (od `netcat`):

    $ nc localhost 1234
    Enter your name: khorne
    Hello, khorne

## Operacje blokujące i nieblokujące

Nasz serwer ma jednak spory mankament: obsługuje tylko jedno połączenie na raz.

Dzieje się tak dlatego, że funkcje `accept()` i `recv()` blokują program do
momentu kiedy nie pojawi się nowe połączenie lub nasz klient czegoś nie wyśle.
Jeśli nowy klient podłączy się w momencie kiedy serwer czeka aż `recv()` się
zakończy, nie wyślemy do nowego klienta nic, dopóki poprzednie połączenie nie
zostanie zamknięte.

Zamiast tego, chcielibyśmy poczekać aż na *którymkolwiek* z naszych połączeń
pojawią się nowe dane, a następnie przełączyć kontekst do funkcji je
obsługującej.

Wiemy jak zrobić to drugie: funkcja obsługi powinna być generatorem który
wykona `yield` lub `yield from` jeśli zamierza czekać na nowe dane.

Jak natomiast obserwować kilka deskryptorów na raz?

## Gniazda sieciowe jako źródła zdarzeń

TODO: Oczekiwanie na zdarzenia z róźnych źródeł. Jak działa `select.poll`. Nic
nowego, gniazda i `select(2)` jest z 1983 roku (4.2BSD).

## Główna pętla

TODO: Składamy wszystko do kupy: owijamy operacje na gniazdach w generatory,
każdego klienta obsługuje osobna instancja generatora, w środku używamy `yield
from`, piszemy pętlę `while True` która oddaje sterowanie generatorom których
deskryptor wypluł z siebie zdarzenie.

## Zakończenie

TODO `asyncio` to tylko API, `yield from` jest użyteczne poza `asyncio`. Python
3.5 wprowadził nową składnię, `async def` i `await`, ale zasada działania jest
taka sama.

## Linki

1. Sibsankar Haldar, Alex Alagarsamy Aravind "Operating Systems", rozdział 5, "CPU Management": https://books.google.pl/books?id=orZ0CLxEMXEC&pg=PA118
2. POSIX.1-2001, makecontext(3) man page: http://linux.die.net/man/3/makecontext
3. PEP-380, Syntax for Delegating to a Subgenerator: https://www.python.org/dev/peps/pep-0380/
4. FreeBSD Developers Handbook, rozdział 7 "Sockets": https://www.freebsd.org/doc/en_US.ISO8859-1/books/developers-handbook/sockets.html
5. Sangjin Han, "Scalable Event Multiplexing": https://people.eecs.berkeley.edu/~sangjin/2012/12/21/epoll-vs-kqueue.html
6. Philip Roberts, "What the hack is the event loop anyway?", JSConf EU 2014: https://www.youtube.com/watch?v=8aGhZQkoFbQ
7. David Beazley, "Python Concurrency From the Ground Up", PyCon 2015: https://www.youtube.com/watch?v=MCs5OvhV9S4
8. Github, "Ghetto Asyncio": github.com:mrzechonek/ghetto-asyncio.git
