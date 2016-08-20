# Async pod maską - Michał Lowas-Rzechonek

## Wprowadzenie

Pętla zdarzeń, programowanie asynchroniczne i inne magiczne słowa.

## Współbieżność i przełączanie kontekstu

Dla uproszczenia jeden procesor (ALU).
Z definicji, program zajmuje cały CPU.
Jak uruchomić dwa programy na raz? DOS vs. Unix.
Co to jest scheduler?

## Współbieżnośc kooperacyjna a wywłaszczanie

Jak uruchomić scheduler?
Jawne oddawanie czasu procesora (każdy syscall).
Przerwanie zegarowe i wywłaszczanie.
Wywłaszczanie powoduje problemy z niepodzielnością operacji.
Ciekawostka historyczna: `makecontext(3)`

## Generatory a współbieżność kooperacyjna

Każdy `yield` to oddanie sterowania.
Generatorów można używać jak "procesów" mając "ręczny" scheduler.
Przełączanie między generatorami jest kontrolowane, nie ma kłopotów z niepodzielnością zadań.
Ale: jeden "zły" generator rozpieprza cały program.

## Kompozycja generatorów

Jeśli chcemy oddać nasz czas komuś, wołamy funkcję.
A co jeśli wołana funkcja postanowi oddać sterowanie?
Skąd wziął się `yield from` i czemu musi być słowem kluczowym.
Przykład: rekurencyjny generator.

## Deskryptory plików i gniazda sieciowe

Co to jest deskryptor (uchwyt) pliku?
Gniazda sieciowe to API do obsługi deskryptorów.
Wiele rodzajów, ale dla nas interesuje HTTP, czyli TCP.
Pokazać podstawowe operacje.

## Operacje blokujące i nieblokujące

Co jeśli mamy dwóch klientów na raz?

## Gniazda sieciowe jako źródła zdarzeń

Oczekiwanie na zdarzenia z róźnych źródeł.
Jak działa `select.poll`
Nic nowego, gniazda `select(2)` jest z 1983 roku (4.2BSD).

## Główna pętla

Składamy wszystko do kupy: owijamy operacje na gniazdach w generatory, każdego
klienta obsługuje osobna instancja generatora, w środku używamy `yield from`,
piszemy pętlę `while True` która oddaje sterowanie generatorom których
deskryptor wypluł z siebie zdarzenie.

## Zakończenie

`asyncio` to tylko API.
`yield from` jest użyteczne poza `asyncio`
Python 3.5 wprowadził nową składnię, `async def` i `await`, ale zasada
działania jest taka sama.

## Linki

1. Sibsankar Haldar, Alex Alagarsamy Aravind: Operating Systems str. 118, https://books.google.pl/books?id=orZ0CLxEMXEC&pg=PA118
2. POSIX.1-2001: makecontext(3) http://linux.die.net/man/3/makecontext
3. Python: PEP-380, Syntax for Delegating to a Subgenerator https://www.python.org/dev/peps/pep-0380/
4. FreeBSD: Sockets https://www.freebsd.org/doc/en_US.ISO8859-1/books/developers-handbook/sockets.html
5. Sangjin Han: Scalable Event Multiplexing https://people.eecs.berkeley.edu/~sangjin/2012/12/21/epoll-vs-kqueue.html
6. Philip Roberts: What the hack is the event loop anyway? https://www.youtube.com/watch?v=8aGhZQkoFbQ
7. Github: Ghetto Asyncio github.com:mrzechonek/ghetto-asyncio.git
