## Problem z wolnym działaniem Płatnika ZUS na MS SQL Server – co sprawdzić?

Twój problem jest klasyczny i dobrze udokumentowany. Opiszę najbardziej prawdopodobne przyczyny i rozwiązania, bez zbędnych pierdół.

### 1. **Brakujące lub nieaktualne indeksy i statystyki SQL po migracji**

To zdecydowanie najczęstszy powód wolnego działania po migracji z SQL Server 2008 na 2019. Streszczenie:[1][2]

- **Przy migracji bazy często nie przenoszą się poprawnie indeksy lub statystyki pozostają nieaktualne**[3][4][1]
- SQL Server używa statystyk do optymalizacji zapytań – gdy są przestarzałe, wybiera nieoptymalne plany wykonania[4][5]
- Rebuild indeksów aktualizuje statystyki, ale **sam rebuild nie zawsze wystarcza** – często trzeba explicite uruchomić UPDATE STATISTICS[6][3]

**Co zrobić:**

```sql
-- 1. Sprawdź przestarzałe statystyki:
SELECT OBJECT_NAME(id) AS Table_Name, 
       st.[name] AS Tname, 
       si.name,
       STATS_DATE(id, indid) AS DateLastUpdate,
       rowmodctr AS rowsModifiedSinceLastUpdate
FROM sys.sysindexes AS si
INNER JOIN sys.[tables] AS st ON si.[id] = st.[object_id]
INNER JOIN sys.schemas AS ss ON st.[schema_id] = [ss].[schema_id]
WHERE STATS_DATE(id, indid) <= DATEADD(DAY,-1,GETDATE())
  AND rowmodctr > 10
ORDER BY [rowmodctr] DESC
```

```sql
-- 2. Zaktualizuj statystyki:
EXEC sp_updatestats;

-- Lub dla konkretnej bazy Płatnika:
USE PlatnikDB;
GO
UPDATE STATISTICS WITH FULLSCAN;
```

```sql
-- 3. Przebuduj indeksy (opcjonalnie, jeśli fragmentacja >30%):
USE PlatnikDB;
GO
EXEC sp_MSforeachtable 'ALTER INDEX ALL ON ? REBUILD';
```

### 2. **Compatibility Level bazy nie pasuje do SQL Server 2019**

Jeśli baza została zmigrowana z 2008, prawdopodobnie ma ustawiony **compatibility level 100** (SQL 2008), podczas gdy SQL Server 2019 używa **150**. To zmienia sposób optymalizacji zapytań i może powodować spowolnienia.[7][8]

**Sprawdzenie i zmiana:**

```sql
-- Sprawdź aktualny poziom:
SELECT name, compatibility_level 
FROM sys.databases 
WHERE name = 'PlatnikDB';

-- Zmień na 150 (SQL 2019):
ALTER DATABASE PlatnikDB SET COMPATIBILITY_LEVEL = 150;
```

**Uwaga:** Po zmianie **obowiązkowo uruchom UPDATE STATISTICS** – nowy cardinality estimator wymaga świeżych danych.[8][9]

### 3. **Optymalizacja bazy w samym programie Płatnik**

Program ma wbudowaną funkcję optymalizacji:[10][11]

1. Uruchom Płatnik **jako Administrator**
2. **Administracja → Ustawienia bazy danych**
3. **Menu Baza → Optymalizuj bazę danych**[10]

Czas zależy od rozmiaru bazy (200MB to kilka–kilkanaście minut). To usuwa duplikaty, czyści historię i reorganizuje dane.[12]

### 4. **Konfiguracja protokołów sieciowych SQL Server**

Jeśli Płatnik łączy się przez sieć (nawet lokalnie), nieprawidłowa konfiguracja protokołów drastycznie spowalnia:[13][14][15]

**W SQL Server Configuration Manager:**

1. Rozwiń **SQL Server Network Configuration → Protocols for [SQLEXPRESS/NAZWA_INSTANCJI]**
2. **Wyłącz Shared Memory i VIA**[13]
3. **Włącz tylko TCP/IP i Named Pipes**[14][15][16]
4. Ustaw **TCP Dynamic Ports = pusty** (usuń 0), **TCP Port = 1433** (lub inny stały)[17]
5. Włącz **SQL Server Browser** (Start Mode: Automatic)[14]
6. **Zrestartuj usługę SQL Server**[14]

### 5. **Limity pamięci SQL Server Express**

SQL Express 2019 ma **limit 1GB RAM**. Jeśli na maszynie pracuje więcej aplikacji, SQL nie ma dość pamięci do cache'owania danych.[18]

**Opcje:**

- Zamknij inne aplikacje podczas pracy Płatnika
- Jeśli możliwe, rozważ upgrade do **SQL Server Standard** (bez limitu RAM)
- Zwiększ max server memory w SQL (dotyczy tylko wersji Standard/Enterprise):[19]

```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'max server memory', 4096; -- 4GB
RECONFIGURE;
```

### 6. **Timeout settings dla długich operacji**

Jeśli import/wysyłanie przerywa się z błędem, zwiększ timeout zapytań:[20][21][22][23]

**SQL Server Management Studio:**

1. Właściwości serwera → **Connections**
2. **Remote query timeout = 3600** (1 godzina) lub **0** (bez limitu)[23]

**Lub przez T-SQL:**

```sql
EXEC sp_configure 'remote query timeout', 3600;
RECONFIGURE;
```

### 7. **Inne potencjalne pułapki**

- **Usługa SQL Server nie uruchomiona**: Sprawdź `services.msc` → SQL Server (SQLEXPRESS) = Running[24]
- **Antywirus/firewall blokuje połączenia** do SQL[25][24]
- **Baza nie została poprawnie zmigrowana** – przy prostym kopiowaniu plików (attach/detach) często gubią się indeksy. Lepiej użyć Kreatora migracji w Płatniku lub SSMS Backup/Restore[26][1][12][23]
- **Dysk wolny/HDD zamiast SSD** – sprawdź Activity Monitor w SSMS (Database I/O)[27]

### Rekomendowany plan działania

1. **UPDATE STATISTICS** (najszybsze, 90% szans że pomoże)
2. Sprawdź **compatibility level** → zmień na 150 jeśli ≠
3. Optymalizacja w Płatniku
4. Sprawdź protokoły TCP/IP/Named Pipes
5. Jeśli nadal wolno: REBUILD INDEX + check query plans w SSMS

### Co nie wiem vs. co założyłem

**Znane:** Problem występuje przy migracji SQL 2008→2019, zwłaszcza przy indeksach/statystykach.[2][1][7]

**Zakładam (bo nie ma dokładnych danych z twojego środowiska):**
- Masz dostęp do SSMS lub SQL Management Studio
- Baza nazywa się PlatnikDB (jeśli nie – podstaw właściwą nazwę)
- Problem nie wynika z hardware (nowy Dell z i5-12400 to więcej niż wystarczające)

**Czego NIE wiem:**
- Czy używasz SQL Express czy Standard
- Jak dokładnie została wykonana migracja (backup/restore, attach/detach, czy kreator Płatnika)
- Czy pojawia się konkretny błąd czy tylko ogólne spowolnienie

Jeśli po wykonaniu UPDATE STATISTICS i zmianie compatibility level nadal nie działa – potrzebny będzie execution plan konkretnych wolnych zapytań (można wyciągnąć z SQL Profiler).

[1](https://www.reddit.com/r/Polska/comments/1d1orx7/program_p%C5%82atnik_10_zus_wolne_dzia%C5%82anie_na_nowym/)
[2](https://www.elektroda.pl/rtvforum/topic4056600.html)
[3](https://stackoverflow.com/questions/30525361/sql-query-performance-is-slow-after-table-reindex-but-fast-after-exec-sp-updates)
[4](https://www.site24x7.com/learn/troubleshoot-outdated-statistics-in-sql.html)
[5](https://www.se.com/in/en/faqs/FA238254/)
[6](https://www.sqlservercentral.com/forums/topic/update-stats-vs-rebuilding-indexes)
[7](https://platnik.net/post/nie_mozna_rozpoznac_wersji/)
[8](https://pl.seequality.net/zmiana-compatibility-level/)
[9](https://learn.microsoft.com/pl-pl/sql/relational-databases/databases/view-or-change-the-compatibility-level-of-a-database?view=sql-server-ver17)
[10](https://www.zus.pl/firmy/program-platnik/poradnik)
[11](https://www.pomocnikplatnika.pl/viewtopic.php?id=1566)
[12](https://pobierz.zus.pl/dystrybucja/a1_10_02_002/dokumentacja/Dok_Admin_A1_10.02.002_wersja_7.3.pdf)
[13](https://forumplatnika.pl/index.php?topic=1900.0)
[14](https://wmkopr.pl/konfiguracja-sql-serwer-do-pracy-w-sieci/)
[15](https://maciosoft.pl/konfiguracja-serwera-sql-do-pracy-w-sieci/)
[16](https://www.elektroda.pl/rtvforum/topic2754843.html)
[17](https://r2platnik.wsparcie.symfonia.pl/hc/pl/articles/25425758912402-Konfiguracja-serwera-SQL)
[18](https://pomoc.comarch.pl/optima/pl/2026/dokumentacja/jakie-ograniczenia-posiada-ms-sql-express-2005-2008-2008-r2-2012-2014-2016/?print=print)
[19](https://learn.microsoft.com/pl-pl/sql/database-engine/configure-windows/server-memory-server-configuration-options?view=sql-server-ver17)
[20](https://public.sygnity.pl/forums/viewtopic.php?t=18430)
[21](https://learn.microsoft.com/pl-pl/sql/database-engine/configure-windows/configure-the-remote-query-timeout-server-configuration-option?view=sql-server-ver17)
[22](https://learn.microsoft.com/pl-pl/troubleshoot/sql/database-engine/performance/troubleshoot-query-timeouts)
[23](https://redir.cache.orange.pl/zus/pobierz/dystrybucja/a1_10_02_002/dokumentacja/Dok_Admin_A1_10.02.002_wersja_8.2.pdf)
[24](https://platnik.net/post/problemy/)
[25](https://4programmers.net/Forum/Bazy_danych/252912-udostepnienie_bazy_sql_klientom_)
[26](https://www.pomocnikplatnika.pl/viewtopic.php?id=806)
[27](https://4programmers.net/Forum/Bazy_danych/350400-sql_i_jego_wydajnosc_problem_z_wydajnoscia_przy_wiekszym_obciazeniu_sql)
[28](https://forum.insert.com.pl/index.php?%2Ftopic%2F95793-problem-z-importem-deklaracji-zus-z-gratyfikanta-i-mikrogratyfikanta%2F)
[29](https://www.zus.pl/instrukcja-zmiany-ustawienia-wezla-dla-wysylki-dokumentow-z-programu-platnik)
[30](https://www.zus.pl/firmy/program-platnik/poradnik/konwersja-bazy-danych-po-pobraniu-aktualizacji-programu)
[31](https://forumplatnika.pl/index.php?topic=1704.0)
[32](https://g.infor.pl/p/_files/37177000/1-praktyczna-obsluga-programu-platnik-37176513.pdf)
[33](https://ucm.pl/platnik-zus-pomoc-baza-bledy/)
[34](https://www.facebook.com/groups/grupawspraciapuezus/posts/2147381619081803/)
[35](https://public.sygnity.pl/forums/viewtopic.php?t=23540)
[36](https://www.facebook.com/groups/grupawspraciapuezus/posts/1943284349491532/)
[37](https://helpdesk.zse-2.krakow.pl/knowledgebase.php?article=85)
[38](https://www.pomocnikplatnika.pl/viewtopic.php?id=844)
[39](https://r2platnik.wsparcie.symfonia.pl/hc/pl/articles/25425703017362--Rozwi%C4%85zany-Problem-z-importem-plik%C3%B3w-na-platform%C4%99-us%C5%82ug-elektronicznych-PUE)
[40](https://www.pomocnikplatnika.pl/viewtopic.php?id=600)
[41](https://www.youtube.com/watch?v=PzCcNjfINiE)
[42](https://learn.microsoft.com/pl-pl/sql/relational-databases/indexes/reorganize-and-rebuild-indexes?view=sql-server-ver17)
[43](https://www.youtube.com/watch?v=_bUUpY9cMUU)
[44](https://www.youtube.com/watch?v=GDKjhak4MZc)
[45](https://boringowl.io/blog/tuning-sql-jak-optymalizowac-wydajnosc-zapytan)
[46](https://www.youtube.com/watch?v=Hqb8L9fE1a8)
[47](https://learn.microsoft.com/pl-pl/troubleshoot/mem/configmgr/alerts-reports-queries/sql-query-times-out-or-console-slow-performance)
[48](https://forumplatnika.pl/index.php?topic=1958.0)
[49](https://www.youtube.com/watch?v=QUlVivfWvDU)
[50](https://www.facebook.com/groups/grupawspraciapuezus/posts/1965519010601399/)
[51](http://oraclin.blogspot.com/2016/12/strojenie-baz-mssql-optymalizacja.html)
[52](https://www.facebook.com/groups/grupawspraciapuezus/posts/1959037971249503/)
[53](https://www.brentozar.com/archive/2017/12/rebuilding-indexes-can-slow-queries/)
[54](https://stackoverflow.com/questions/45706088/why-are-these-sql-server-statistics-outdated-when-its-set-to-auto-update)
[55](https://www.db-berater.de/2025/01/when-auto-update-statistics-will-not-update-your-statistics/)
[56](https://www.reddit.com/r/SQLServer/comments/pkv7us/sql_performance_after_moving_to_new_server/)
[57](https://learn.microsoft.com/en-us/sql/t-sql/statements/update-statistics-transact-sql?view=sql-server-ver17)
[58](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/reorganize-and-rebuild-indexes?view=sql-server-ver17)
[59](https://platnik.net/post/instalacja_sql/)
[60](https://www.brentozar.com/archive/2020/11/how-bad-statistics-cause-bad-sql-server-query-performance/)
[61](https://learn.microsoft.com/en-us/answers/questions/2080238/slow-query-performance-in-sql-server-2019-after-da)
[62](https://www.zus.pl/documents/10182/44555/Instrukcja+instalacji.pdf/8e5f224e-3a91-48a5-a976-a84db9c6b9e9)
[63](https://learn.microsoft.com/en-us/troubleshoot/sharepoint/performance/outdated-database-statistics)
[64](https://forums.oracle.com/ords/apexds/post/sql-performance-degraded-after-database-migration-to-new-se-2640)
[65](https://learn.microsoft.com/pl-pl/sql/database-engine/configure-windows/configure-the-remote-login-timeout-server-configuration-option?view=sql-server-ver17)
[66](https://forumplatnika.pl/index.php?action=recent%3Bstart%3D80)
[67](https://www.facebook.com/groups/grupawspraciapuezus/posts/1974510916368875/)
[68](https://learn.microsoft.com/pl-pl/sql/database-engine/configure-windows/configure-a-server-to-listen-on-a-specific-tcp-port?view=sql-server-ver17)
[69](https://public.sygnity.pl/forums/viewtopic.php?t=14404&start=15)
[70](https://www.zus.pl/firmy/program-platnik/o-programie-platnik/wymagania-sprzetowe-wersja-10.02.002)
