====================================
Kopie zapasowe i odzyskiwanie danych 
====================================
**Autorzy:** Norbert Antonovitch, Michał Bednarczyk, Jan Balazs de Borbatviz

Wstęp
-----
Kopie zapasowe w PostgreSQL można podzielić na dwa główne nurty: kopie logiczne wykonywane narzędziami ``pg_dump`` i ``pg_dumpall``  oraz kopie fizyczne całego klastra wykonywane m.in. przez ``pg_basebackup`` wraz z archiwizacją WAL dla odzyskiwania punktowego PITR (Point-in-Time Recovery). Kopie logiczne są wygodne do odtwarzania pojedynczych obiektów i pojedynczych baz, natomiast kopie fizyczne są podstawą pełnego odtworzenia instancji, tablespaces i odzyskiwania po awariach.

Tworzenie kopii zapasowej całego systemu PostgreSQL - mechanizmy wbudowane
--------------------------------------------------------------------------
Najprostszym wbudowanym sposobem wykonania kopii całego klastra jest ``pg_dumpall``, które eksportuje wszystkie bazy w klastrze oraz obiekty globalne wspólne dla całej instancji, takie jak role i przestrzenie tabel. Taki zrzut ma postać tekstowego skryptu SQL i odtwarza się go zwykle przez ``psql``, ale metoda ta jest logiczna, więc nie daje możliwości odzyskiwania punktowego przez WAL.

Drugim, ważniejszym z punktu widzenia odporności na awarie mechanizmem jest ``pg_basebackup``, które tworzy fizyczną kopię działającego klastra bez blokowania normalnej pracy klientów. Narzędzie to wykonuje kopię całego klastra, a nie pojedynczych baz czy tabel, może pracować w formacie katalogowym lub ``tar`` i może dołączać wymagane pliki WAL metodą stream, fetch albo none. Dokumentacja podkreśla też, że taka kopia może służyć zarówno do PITR, jak i do zbudowania serwera standby.

Aby pełna kopia fizyczna była użyteczna w scenariuszu odtwarzania do wybranego momentu, należy włączyć archiwizację WAL przez ustawienie co najmniej ``wal_level = replica``, ``archive_mode = on`` oraz poprawnego ``archive_command`` albo ``archive_library``. PostgreSQL zapisuje każdą zmianę w logu WAL, a po odtworzeniu kopii bazowej można odtworzyć kolejne segmenty WAL, aby dojść do stanu bieżącego lub do wskazanego punktu w czasie.

Przykładowe polecenia:

.. code-block:: bash

    # logiczna kopia całego klastra
    pg_dumpall -U postgres --clean --file=cluster.sql

    # fizyczna kopia całego klastra z dołączeniem WAL
    pg_basebackup -h dbserver -D /backup/base -Fp -X stream -P

W praktyce do dokumentacji laboratoryjnej warto zaznaczyć, że ``pg_dumpall`` lepiej nadaje się do migracji i prostych kopii administracyjnych, a ``pg_basebackup`` z archiwizacją WAL do scenariuszy disaster recovery.

Tworzenie kopii zapasowych poszczególnych baz danych - mechanizmy wbudowane
---------------------------------------------------------------------------
Do wykonania kopii pojedynczej bazy służy ``pg_dump``, które tworzy spójny zrzut nawet wtedy, gdy baza jest równocześnie używana przez innych użytkowników, i nie blokuje zwykłych operacji odczytu ani zapisu. Narzędzie obsługuje format tekstowy, custom, directory oraz tar, przy czym formaty archiwalne współpracują z ``pg_restore`` i pozwalają na selektywne odtwarzanie obiektów.

Istotne jest to, że ``pg_dump`` wykonuje kopię tylko jednej bazy danych. Format custom (``-Fc``) i directory (``-Fd``) są najbardziej elastyczne, bo pozwalają wybierać odtwarzane elementy, zmieniać kolejność odtwarzania, a w części scenariuszy także korzystać z odtwarzania równoległego.

Przykładowe polecenia:

.. code-block:: bash

    # kopia pojedynczej bazy w formacie custom
    pg_dump -U postgres -d labdb -Fc -f /backup/labdb.dump

    # kopia pojedynczej bazy jako skrypt SQL
    pg_dump -U postgres -d labdb -Fp -f /backup/labdb.sql

    # odtworzenie archiwum custom
    pg_restore -d labdb_restored /backup/labdb.dump

Jeżeli celem jest ochrona wybranych schematów lub tabel, ``pg_dump`` pozwala zawęzić zakres eksportu przez opcje takie jak ``-n`` dla schematu i ``-t`` dla tabeli. Trzeba jednak zaznaczyć, że wybiórczy zrzut nie gwarantuje kompletności zależności wszystkich obiektów w nowym, pustym środowisku, jeśli pominięto elementy, od których zależy odtwarzany obiekt.

Odzyskiwanie usuniętych lub uszkodzonych danych
-----------------------------------------------
Sposób odzyskiwania zależy od tego, czy problem dotyczy pojedynczego obiektu logicznego, całej bazy, przestrzeni tabel, czy fizycznego uszkodzenia danych. PostgreSQL rozróżnia w praktyce odzyskiwanie logiczne z plików dump oraz odzyskiwanie fizyczne z kopii bazowej i archiwum WAL.

Odtwarzanie tabel i innych obiektów logicznych
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Jeżeli wcześniej wykonano kopię ``pg_dump`` w formacie archiwalnym, można odtworzyć pojedynczą tabelę lub inny obiekt selektywnie przy pomocy ``pg_restore``, bez przywracania całej bazy. To jest najprostszy wariant odzyskania przypadkowo usuniętej tabeli, o ile odpowiedni obiekt znajdował się w logicznej kopii zapasowej.

Przykład:

.. code-block:: bash

    pg_restore -d labdb -t public.wyniki /backup/labdb.dump

Odtwarzanie usuniętej bazy danych
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Usuniętą bazę można najłatwiej odtworzyć z wcześniejszego zrzutu ``pg_dump`` tej konkretnej bazy albo z ``pg_dumpall``, jeżeli wykonywano kopię całego klastra. W przypadku wymogu odtworzenia dokładnie do chwili sprzed usunięcia, potrzebna jest kopia fizyczna oraz archiwum WAL, ponieważ same zrzuty logiczne nie są częścią rozwiązania continuous archiving i nie wspierają PITR.

Odtwarzanie przestrzeni tabel i pełnego klastra
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Przestrzenie tabel są obiektami globalnymi klastra, dlatego ich definicje są uwzględniane przez ``pg_dumpall``, a przy kopiach fizycznych ``pg_basebackup`` zachowuje także ich strukturę i mapowanie. Dokumentacja ``pg_basebackup`` wskazuje, że przy pracy z tablespaces można użyć mapowania ``--tablespace-mapping``, a przy odtwarzaniu trzeba zweryfikować poprawność dowiązań symbolicznych i lokalizacji danych.

Przy fizycznym uszkodzeniu danych, katalogu PGDATA albo tabelpaces standardowa procedura obejmuje odtworzenie kopii bazowej, usunięcie starych plików danych, odtworzenie katalogów danych i tablespaces, skonfigurowanie ``restore_command``, utworzenie pliku ``recovery.signal``, a następnie ponowne uruchomienie serwera w trybie recovery. PostgreSQL podczas odtwarzania odczytuje wymagane segmenty WAL i może dojść do stanu bieżącego albo zatrzymać się na wybranym punkcie odzyskiwania.

PITR i odzyskiwanie stanu sprzed błędu
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Najważniejszym mechanizmem odzyskiwania po usunięciu danych operacyjnych jest PITR, czyli przywrócenie systemu do chwili sprzed błędnej operacji, na przykład ``DROP TABLE`` lub ``DROP DATABASE``. W takim scenariuszu odtwarza się kopię bazową, konfiguruje ``restore_command``, a następnie ustawia cel odzyskiwania, np. przez czas, nazwany punkt przywracania lub identyfikator transakcji.

To rozwiązanie ma jednak ważne ograniczenie: nie służy do odzyskania pojedynczej tabeli "w miejscu" bez wpływu na resztę systemu, lecz do odtworzenia całego klastra do wcześniejszego stanu. Dlatego w praktyce laboratoryjnej często łączy się oba podejścia: regularne ``pg_dump`` dla obiektów logicznych i ``pg_basebackup`` z archiwizacją WAL dla pełnego recovery.

Dedykowane oprogramowanie i skrypty zewnętrzne do automatyzacji
---------------------------------------------------------------
Wbudowane narzędzia PostgreSQL są wystarczające do ręcznego wykonywania kopii, ale w środowisku produkcyjnym często wykorzystuje się oprogramowanie automatyzujące harmonogramy, retencję, walidację i odtwarzanie. Dwa najczęściej wskazywane rozwiązania to Barman i pgBackRest, przy czym dostępna dokumentacja Barman bezpośrednio opisuje obsługę pełnych i przyrostowych kopii, archiwizacji WAL oraz automatycznego uruchamiania backupów.

Barman wykonuje kopie całego serwera PostgreSQL i wymaga poprawnej archiwizacji WAL, oferując różne metody kopii, w tym streaming przez ``pg_basebackup``, ``rsync`` oraz backupy snapshotowe w chmurze. Dokumentacja podaje też wsparcie dla backupów przyrostowych, limitowania pasma, kompresji i pracy z harmonogramem przez zewnętrzne mechanizmy systemowe, np. cron.

Przykładowe zastosowania narzędzi zewnętrznych:

* **Barman:** centralne repozytorium backupów, archiwizacja WAL, backupy pełne i przyrostowe, odzyskiwanie całych instancji.
* **pgBackRest:** popularne narzędzie do wydajnych backupów i retencji, często używane w środowiskach HA oraz większych instalacjach PostgreSQL.
* **Własne skrypty Bash/PowerShell:** wywołanie ``pg_dump``, ``pg_dumpall`` lub ``pg_basebackup``, kompresja plików, rotacja katalogów, logowanie i wysyłka wyników przez e-mail lub do systemu monitoringu.

Przykładowy prosty skrypt automatyzujący kopię jednej bazy:

.. code-block:: bash

    #!/bin/bash
    DATA=$(date +%F_%H-%M)
    KATALOG=/backup/postgres
    BAZA=labdb

    mkdir -p "$KATALOG"
    pg_dump -U postgres -d "$BAZA" -Fc -f "$KATALOG/${BAZA}_${DATA}.dump"
    find "$KATALOG" -type f -name "${BAZA}_*.dump" -mtime +7 -delete

Taki skrypt jest prosty, ale nie zapewnia pełnego disaster recovery, bo nie obejmuje archiwizacji WAL i nie chroni całego klastra. Z tego powodu dla systemów krytycznych bardziej właściwe są narzędzia klasy Barman lub pgBackRest, które porządkują pełny proces backupu, retencji, testów i odtwarzania.

Podsumowanie
--------------------------------------
W rozdziale warto wyraźnie rozróżnić kopie logiczne i fizyczne, bo odpowiadają one na inne potrzeby eksploatacyjne. ``pg_dump`` i ``pg_dumpall`` są najlepsze do prostych eksportów i selektywnego odtwarzania, natomiast ``pg_basebackup`` z archiwizacją WAL jest podstawą odzyskiwania po poważnej awarii i realizacji PITR.

