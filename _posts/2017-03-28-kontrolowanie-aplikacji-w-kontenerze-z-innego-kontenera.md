---
layout: post
title:  "Kontrolowanie aplikacji uruchomionej w dockerze z innego kontenera dockera"
date:   2017-03-28 22:00:00 +0100
categories: docker testing robotframework
---
O [Dockerze](https://www.docker.com/) pierwszy raz usłyszałem na 
[JUC 2014:Berlin](https://www.cloudbees.com/event/juc/2014/berlin). 
Pamiętam, że byłem zachwycony ideą konteneryzacji aplikacji jak i szybkością z jaką działają kontenery 
(w porównaniu do standardowych maszyn wirtualnych). Cały technologiczny świat zachłysnął się
kontenerami a mi przyszło rozwiązać następującą zagadkę: jak kontrolować aplikację uruchomioną
w kontenerze z poziomu innej aplikacji uruchomionej w innym kontenerze?

## Problem
W ostatnim projekcie, w którym brałem udział natknąłem się na dosyć specyficzny
problem. Całe środowisko testowe było uruchamiane w kontenerze dockera. W tym 
samym kontenerze jest uruchomiony zarówno agent Bamboo jak i sam Robotframework (RFW). Tak, 
wiem, niezbyt to dockerowe. Sam SUT (System Under Test) uruchomiony był w osobnym kontenerze.

Testując zawsze powinniśmy być pewni w jakim stanie znajduje się SUT. Idealnie każdy test 
powinien przygotować sobie środowisko w setupie i posprzątać po sobie w teardownie.
Bardzo często do przeprowadzenia testu potrzebne jest wprowadzenie do aplikacji pewnych
danych testowych lub też wprowadzenie aplikacji w pewny znany nam stan. Pewne typy testów wymagają
zmian w OS, w którym jest uruchomiony SUT. 
W opisywanym przypadku testowana aplikacja po starcie wykonywała pewne operacje, których nie można było
wywołać z poziomu API. Aby ptrzetestować wszystkie główne scenariusze związane ze wspomnianymi
operacjami musiałem przy każdym test casie restartować SUT. Pewne scenariusze przewidywały brak 
dostępu do sieci, inne powinny wyciąć tylko komunikację na jednym porcie (i to raz wychodzącą a innym
razem przychodzącą)... Jednym słowem ciężki orzech do zgryzienia.

Aby ułatwić sobie pracę podzieliłem problem na 2 mniejsze:

* Restartowanie SUT.
* Kontrola środowiska SUT.

Po krótkim googlaniu i kilku rozmowach z programistą okazało się, że sam docker dostarcza narzędzi
do rozwiązania wspomnianych problemów.

## Restartowanie SUT
Z natury kontenery dockera "śledzą" jedną uruchomioną w nich aplikację. Po wyłączeniu/
ubiciu aplikacji wyłącza się sam kontener. Tylko, że taki kontener pozostanie "martwy"
na zawsze. Chyba, że dodamy do polecenia `docker run` opcję `--restart always`. Wtedy
docker zrestartuje kontener:

```bash
$ docker run --restart always ubuntu:16.04
```

Powyższe polecenie uruchomi kontener z obrazu Ubuntu 16.04 i w przypadku, gdy główna
aplikacja zostanie wyłączona i kontener się wyłączy, zostanie automatycznie znowu włączony.

## Kontrola środowiska SUT
O ile SUT nie korzysta z `volume` do zapisywania ustawień restart spowoduje zrestartowanie
ustawień aplikacji - kontenery są lepsze niż kobiety pod tym względem - "nie pamiętają".

Ale jak zrestartować samą aplikację lub kontrolować jej środowisko uruchomienia?
Z pomocą przychodzi biblioteka RFW [Remote](https://github.com/robotframework/RemoteInterface)
oraz zdalny serwer `XML-RPC`. Remote jest dosyć specyficzną biblioteką gdyż sama w sobie nie zawiera
żadnych słów kluczowych. To co potrafi to udostępnianie wszytkich keywordów biblioteki uruchomionej
z pomocą XML-RPC serwera. Neleży zatem stworzyć niemal standardową bibliotekę RFW
(poszerzoną o uruchomienie serwera XML-RPC) i umieścić ją w kontenerze SUT. W test casie
RFW dodajemy import biblioteki "Remote" z informacją gdzie znajduje się nasza zdalna biblioteka i już 
możemy używać jej keywordów, które np. za pomocą komendy `kill` zabiją SUT a tym samym zrestartują cały kontener.
Problem solved!

## Przykład
Gotowy projekt zawierający szczegółowy opis oraz przykładowe Dockerfile i bibliotekę zdalną można pobrać z mojego
repozytorium na [githubie](https://github.com/czerus/control_docker_from_docker).
