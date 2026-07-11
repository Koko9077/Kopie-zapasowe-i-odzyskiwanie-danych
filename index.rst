Kopie zapasowe i odzyskiwanie danych
====================================

:Autorzy: Norbert Antonovitch, Michał Bednarczyk, Jan Balazs de Borbatviz

Wstęp
-----

Kopie zapasowe w PostgreSQL można podzielić na dwa główne nurty: kopie logiczne wykonywane narzędziami ``pg_dump`` i ``pg_dumpall`` oraz kopie fizyczne całego klastra wykonywane m.in. przez ``pg_basebackup`` wraz z archiwizacją WAL dla odzyskiwania punktowego PITR (Point-in-Time Recovery). Kopie logiczne są wygodne do odtwarzania pojedynczych obiektów i pojedynczych baz, natomiast kopie fizyczne są podstawą pełnego odtworzenia instancji, przestrzeni tabel i odzyskiwania po awariach.

Tworzenie kopii zapasowej całego systemu PostgreSQL - mechanizmy wbudowane
--------------------------------------------------------------------------

Najprostszym wbudowanym sposobem wykonania kopii całego klastra jest ``pg_dumpall``, które eksportuje wszystkie bazy w klastrze oraz obiekty globalne wspólne dla całej instancji, takie jak role i przestrzenie tabel. Taki zrzut ma postać tekstowego skryptu SQL i odtwarza się go zwykle przez ``psql``, ale metoda ta jest logiczna, więc nie daje możliwości odzyskiwania punktowego przez WAL.

Drugim, ważniejszym z punktu widzenia odporności na awarie mechanizmem jest ``pg_basebackup``, które tworzy fizyczną kopię działającego klastra bez blokowania normalnej pracy klientów. Narzędzie to wykonuje kopię całego klastra, a nie pojedynczych baz czy tabel, może pracować w formacie katalogowym lub tar i może dołączać wymagane pliki WAL metodą stream, fetch albo none. Dokumentacja podkreśla też, że taka kopia może służyć zarówno do PITR, jak i do zbudowania serwera standby.

Aby pełna kopia fizyczna była użyteczna w scenariuszu odtwarzania do wybranego momentu, należy włączyć archiwizację WAL przez ustawienie co najmniej ``wal_level = replica``, ``archive_mode = on`` oraz poprawnego ``archive_command`` albo ``archive_library``.

.. code-block:: bash

   # logiczna kopia całego klastra
   pg_dumpall -U postgres --clean --file=cluster.sql

   # fizyczna kopia całego klastra z dołączeniem WAL
   pg_basebackup -h dbserver -D /backup/base -Fp -X stream -P

W praktyce administracyjnej ``pg_dumpall`` lepiej nadaje się do migracji i prostych kopii bezpieczeństwa, a ``pg_basebackup`` z archiwizacją WAL do scenariuszy disaster recovery.

Tworzenie kopii zapasowych poszczególnych baz danych
----------------------------------------------------

Do wykonania kopii pojedynczej bazy służy ``pg_dump``, które tworzy spójny zrzut nawet wtedy, gdy baza jest równocześnie używana przez innych użytkowników, i nie blokuje zwykłych operacji odczytu ani zapisu. Narzędzie obsługuje format tekstowy, custom, directory oraz tar, przy czym formaty archiwalne współpracują z ``pg_restore`` i pozwalają na selektywne odtwarzanie obiektów.

.. code-block:: bash

   # kopia pojedynczej bazy w formacie custom
   pg_dump -U postgres -d labdb -Fc -f /backup/labdb.dump

   # kopia pojedynczej bazy jako skrypt SQL
   pg_dump -U postgres -d labdb -Fp -f /backup/labdb.sql

   # odtworzenie z archiwum custom
   pg_restore -d labdb_restored /backup/labdb.dump

Odzyskiwanie usuniętych lub uszkodzonych danych
-----------------------------------------------

Sposób odzyskiwania zależy od tego, czy problem dotyczy pojedynczego obiektu logicznego, całej bazy, przestrzeni tabel, czy fizycznego uszkodzenia danych. W środowisku PostgreSQL rozróżnia się odzyskiwanie logiczne (z plików wykonywanych narzędziami takimi jak ``pg_dump``) oraz fizyczne (z kopii bazowej i archiwum WAL).

Odtwarzanie tabel i pojedynczych obiektów logicznych
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Najprostszą sytuacją jest przypadkowe usunięcie tabeli lub rekordu. Jeżeli wcześniej wykonano kopię za pomocą ``pg_dump`` w formacie archiwalnym (np. custom), można odtworzyć pojedynczy obiekt przy pomocy narzędzia ``pg_restore``, bez ingerencji w resztę środowiska bazy danych.

.. code-block:: bash

   # Odtwarzanie konkretnej tabeli "wyniki" do istniejącej bazy "labdb"
   pg_restore -d labdb -t public.wyniki /backup/labdb.dump

Odtwarzanie w ten sposób nie gwarantuje jednak spójności z innymi powiązanymi tabelami (np. kluczami obcymi), jeśli ich stan uległ modyfikacji od czasu wykonania zrzutu.

**Brak kopii logicznej (Tymczasowa instancja PITR)**
Jeśli dysponujemy wyłącznie kopią fizyczną (z ``pg_basebackup``) oraz archiwum WAL, odzyskanie usuniętej tabeli (np. po operacji ``DROP TABLE``) bez przerywania pracy głównego systemu produkcyjnego wymaga zastosowania zaawansowanej techniki:
1. Uruchomienie na osobnym porcie lub innym serwerze tymczasowej instancji PostgreSQL z wykorzystaniem kopii fizycznej.
2. Wykonanie na niej procesu PITR (Point-in-Time Recovery) do momentu (np. do sekundy) tuż przed krytycznym błędem/usunięciem tabeli.
3. Wyeksportowanie z tej ocalonej, tymczasowej bazy jedynie utraconej tabeli za pomocą ``pg_dump``.
4. Zaimportowanie odzyskanej tabeli z powrotem na główny serwer produkcyjny.

Odtwarzanie usuniętej bazy danych (DROP DATABASE)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
W przypadku utraty całej bazy, proces przebiega dwutorowo:

1. **Z wykorzystaniem kopii logicznej:** Używamy narzędzia ``pg_restore`` z opcją ``-C`` (clean). Opcja ta przed rozpoczęciem odtwarzania emituje polecenie ``DROP DATABASE`` (usuwając starą bazę, jeśli jeszcze istnieje), a następnie ``CREATE DATABASE``, tworząc nową, pustą strukturę. Dopiero do tak przygotowanej bazy docelowej wgrywane są odzyskiwane dane z pliku zrzutu.
2. **Z wykorzystaniem kopii fizycznej i plików WAL (PITR):** Wymaga wykonanego wcześniej ``pg_basebackup`` oraz archiwizacji logów transakcyjnych. Pozwala na powrót do dokładnego momentu w czasie sprzed awarii. Wykorzystanie tego mechanizmu wymusi cofnięcie w czasie wszystkich baz danych działających w tym samym klastrze.

Odtwarzanie po fizycznym uszkodzeniu dysku lub przestrzeni tabel
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Przy fizycznym uszkodzeniu dysku, na którym znajdował się klaster (``$PGDATA``) lub konkretne przestrzenie tabel, standardowa procedura obejmuje następujące kroki:

1. Zatrzymanie serwera PostgreSQL (jeśli proces sam nie uległ awarii).
2. Wyczyszczenie uszkodzonych katalogów z danymi lub przygotowanie nowej lokalizacji dyskowej.
3. Odtworzenie pełnej kopii klastra z ``pg_basebackup`` (z opcją ``--tablespace-mapping`` w przypadku przenosin na nowy dysk, aby poprawnie zmapować ścieżki do przestrzeni tabel).
4. Konfigurację dyrektywy ``restore_command`` w pliku ``postgresql.conf``, aby serwer mógł zaciągnąć zarchiwizowane pliki WAL.
5. Utworzenie pustego pliku sygnalizacyjnego ``recovery.signal`` w głównym katalogu z danymi.
6. Ponowne uruchomienie serwera. System przejdzie w tryb odzyskiwania, zaaplikuje transakcje z logów WAL do osiągnięcia spójności, po czym wznowi normalną pracę.

Point-In-Time Recovery (PITR)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
PITR to zaawansowany mechanizm pozwalający na powrót bazy do dowolnie wybranego momentu z przeszłości, chroniąc przed błędami ludzkimi (np. wywołaniem błędnego zapytania ``UPDATE`` bez klauzuli ``WHERE``). Wymaga archiwizacji WAL (Continuous Archiving) oraz pełnego zrzutu fizycznego. 

Odtwarzanie zatrzymuje się na wskazanym twardym punkcie w czasie, np.:

.. code-block:: ini

   # Konfiguracja momentu zatrzymania odtwarzania
   recovery_target_time = '2026-07-10 14:30:00'

Wadą PITR jest to, że nie da się nim przywrócić pojedynczej tabeli bezpośrednio "w miejscu" – mechanizm ten operuje na poziomie fizycznym i cofa w czasie stan **całego klastra bazodanowego**.

Dedykowane oprogramowanie i skrypty zewnętrzne
----------------------------------------------

Wbudowane narzędzia PostgreSQL są wystarczające do ręcznego wykonywania kopii, ale w środowisku produkcyjnym często wykorzystuje się oprogramowanie automatyzujące harmonogramy, retencję, walidację i odtwarzanie. Dwa najczęściej wskazywane rozwiązania to Barman i pgBackRest. 

Przykładowy prosty skrypt automatyzujący kopię jednej bazy w systemie Linux (Bash):

.. code-block:: bash

   #!/bin/bash
   DATA=$(date +%F_%H-%M)
   KATALOG=/backup/postgres
   BAZA=labdb

   mkdir -p "$KATALOG"
   pg_dump -U postgres -d "$BAZA" -Fc -f "$KATALOG/${BAZA}_${DATA}.dump"
   find "$KATALOG" -type f -name "${BAZA}_*.dump" -mtime +7 -delete

**Ważna uwaga dotycząca uwierzytelniania:** Powyższy skrypt wykona się bez interwencji użytkownika tylko w środowiskach z metodą uwierzytelniania ``peer`` lub ``trust``. Jeżeli serwer wymaga hasła (np. ``scram-sha-256``), proces zatrzyma się w oczekiwaniu na jego wpisanie. Aby umożliwić pełną automatyzację, należy skonfigurować ukryty plik poświadczeń ``.pgpass`` w katalogu domowym użytkownika systemowego uruchamiającego skrypt, ewentualnie przekazać hasło za pomocą zmiennej środowiskowej ``PGPASSWORD``.

Podsumowanie
------------

W procesie zabezpieczania danych należy wyraźnie rozróżnić kopie logiczne i fizyczne, gdyż odpowiadają one na inne potrzeby eksploatacyjne. Narzędzia ``pg_dump`` i ``pg_dumpall`` są najlepsze do prostych eksportów i selektywnego odtwarzania pojedynczych obiektów. Natomiast mechanizm ``pg_basebackup`` w połączeniu z archiwizacją WAL stanowi absolutną podstawę do odzyskiwania po poważnej awarii dyskowej oraz do realizacji procedur PITR.
