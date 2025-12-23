## Jak wyciągnąć execution plan i zdiagnozować wolne zapytania – konkretnie

***

### **Opcja 1: SQL Server Profiler (prostsze, ale deprecated)**

SQL Profiler to GUI do rejestrowania aktywności SQL Server. Microsoft oficjalnie go deprecjonuje na rzecz Extended Events, ale nadal działa i jest prostszy dla kogoś kto nie robi tego codziennie.

**Krok po kroku:**

1. **Uruchom SQL Server Profiler**
   - Start → Microsoft SQL Server Tools → SQL Server Profiler
   - File → New Trace → Połącz się z instancją SQL (np. `localhost\SQLEXPRESS`)

2. **Wybierz template i dostosuj eventy**
   - Template: **TSQL_Duration** lub **Tuning**
   - Events Selection → Show all events
   - **Zaznacz:**
     - `RPC:Completed` 
     - `SQL:BatchCompleted`
     - `SQL:StmtCompleted`
     - **Showplan XML** (najważniejsze – to jest execution plan)
   - **Column filters:** 
     - `Duration` > 1000 (pokazuj tylko zapytania dłuższe niż 1 sekunda = 1000ms)
     - `ApplicationName` LIKE `%Płatnik%` (jeśli chcesz tylko ruch z Płatnika)

3. **Uruchom trace i wykonaj wolną operację**
   - Kliknij **Run**
   - Przejdź do Płatnika i **wykonaj dokładnie tę operację która jest wolna** (np. import, wysyłka)
   - Obserwuj Profiler – zobaczysz wszystkie zapytania wykonywane przez Płatnik

4. **Znajdź wolne zapytanie**
   - Sortuj po kolumnie **Duration** (malejąco)
   - Znajdź zapytanie z największym Duration (będzie w milisekundach – 5000+ to już problem)
   - Kliknij na wiersz z tym zapytaniem

5. **Wyciągnij execution plan**
   - **Jeśli zaznaczyłeś Showplan XML:** Kliknij wiersz → kolumna `TextData` zawiera XML planu
   - Skopiuj cały XML → Zapisz jako `plan.sqlplan`
   - Otwórz w SSMS (File → Open → File) – zobaczysz graficzny plan

6. **Alternatywnie – skopiuj samo zapytanie:**
   - Kliknij wiersz z wolnym zapytaniem
   - Kolumna `TextData` = treść zapytania SQL
   - Skopiuj zapytanie i przejdź do SSMS (patrz Opcja 2)

**Uwaga:** SQL Express **NIE MA** SQL Profiler. Jeśli używasz Express, musisz użyć Extended Events (Opcja 3) lub zainstalować SSMS i pobrać zapytania ręcznie.

***

### **Opcja 2: SQL Server Management Studio (SSMS) – ręczna analiza**

Jeśli znasz już wolne zapytanie (np. wyciągnięte z Profilera) lub domyślasz się co jest wolne:

1. **Otwórz SSMS → New Query**
2. **Włącz execution plan:**
   ```sql
   SET STATISTICS IO ON;
   SET STATISTICS TIME ON;
   SET SHOWPLAN_XML ON; -- Lub kliknij "Include Actual Execution Plan" (Ctrl+M)
   ```

3. **Wklej i uruchom wolne zapytanie**
   - Jeśli nie znasz dokładnego zapytania, możesz uruchomić typowe operacje Płatnika i przechwycić ruch (patrz Opcja 3)

4. **Przeanalizuj plan:**
   - Zakładka **Execution Plan** (graficzny widok)
   - Szukaj:
     - **Table Scan / Clustered Index Scan** z wysokim kosztem (%) → brakuje indeksu
     - **Sort** lub **Hash Match** z dużymi danymi → brakuje sortowanego indeksu
     - **Warnings** (żółty trójkąt) → np. implicit conversion, missing statistics
     - **Thick arrows** (grube strzałki) → dużo wierszy przenoszonych = bottleneck

5. **Sprawdź statystyki:**
   - Najedź na operację → Tooltip pokazuje `Estimated Rows` vs `Actual Rows`
   - **Duża różnica (np. 10x)** = nieaktualne statystyki → `UPDATE STATISTICS`

***

### **Opcja 3: Extended Events (nowoczesne, ale bardziej techniczne)**

To oficjalny zamiennik Profilera. Bardziej wydajne, ale mniej intuicyjne GUI.

**Quick setup:**

1. **SSMS → Object Explorer → Management → Extended Events → New Session Wizard**

2. **Wybierz template:**
   - "Query Detail Tracking" (dla wolnych zapytań)
   
3. **Select Events to Capture:**
   - `sqlserver.rpc_completed`
   - `sqlserver.sql_batch_completed`
   - `sqlserver.query_post_execution_showplan` (execution plan)

4. **Set Filters (predicate):**
   - `duration` > 1000000 (1 sekunda w mikrosekundach)
   - `client_app_name` LIKE `%Płatnik%`

5. **Data Storage:**
   - Zapisz do pliku `.xel` lub oglądam live (Watch Live Data)

6. **Start session i odtwórz problem w Płatniku**

7. **Analizuj wyniki:**
   - Kliknij prawym na sesję → Watch Live Data
   - Kolumna `duration` pokazuje czas
   - `batch_text` / `statement` = treść zapytania
   - `query_plan_hash` → użyj do wyciągnięcia planu z cache (patrz niżej)

***

### **Opcja 4: Query Store (SQL 2016+, automatyczny monitoring)**

Jeśli SQL Server 2019 ma włączony Query Store (zwykle domyślnie wyłączony), możesz podejrzeć historię wolnych zapytań **bez żadnego ręcznego trace**:

```sql
-- Włącz Query Store dla bazy Płatnika:
ALTER DATABASE PlatnikDB SET QUERY_STORE = ON;

-- Sprawdź czy działa:
SELECT * FROM sys.database_query_store_options WHERE database_id = DB_ID('PlatnikDB');
```

**Poczekaj kilka godzin (lub wykonaj wolne operacje)**, potem:

```sql
-- Top 10 najwolniejszych zapytań:
SELECT TOP 10
    qt.query_sql_text AS QueryText,
    rs.avg_duration/1000.0 AS AvgDurationMS,
    rs.max_duration/1000.0 AS MaxDurationMS,
    rs.count_executions AS ExecutionCount,
    qp.query_plan AS ExecutionPlan
FROM sys.query_store_query q
JOIN sys.query_store_plan qp ON q.query_id = qp.query_id
JOIN sys.query_store_runtime_stats rs ON qp.plan_id = rs.plan_id
JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
ORDER BY rs.avg_duration DESC;
```

Kolumna `ExecutionPlan` to XML – skopiuj i zapisz jako `.sqlplan`, otwórz w SSMS.

***

### **Opcja 5: Wyciągnięcie planu z cache (dla niedawno wykonanych zapytań)**

Jeśli zapytanie właśnie się wykonało (w ciągu ostatnich minut/godzin), plan może być jeszcze w cache:

```sql
-- Znajdź wolne zapytania w cache:
SELECT TOP 20
    qs.total_elapsed_time / qs.execution_count / 1000.0 AS AvgDurationMS,
    qs.execution_count,
    SUBSTRING(qt.text, (qs.statement_start_offset/2)+1, 
        ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(qt.text)
        ELSE qs.statement_end_offset END - qs.statement_start_offset)/2)+1) AS QueryText,
    qp.query_plan AS ExecutionPlan
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
WHERE qt.text LIKE '%tabela_z_platnika%' -- Opcjonalny filtr
ORDER BY qs.total_elapsed_time / qs.execution_count DESC;
```

Kliknij na `ExecutionPlan` (niebieski link) → otwiera graficzny plan w SSMS.

***

## **Co robić z execution planem gdy już go masz?**

1. **Szukaj operacji z najwyższym kosztem (%)** 
   - Najgrubsze strzałki / największe ikony = bottleneck

2. **Missing Index Hints**
   - Kliknij prawym na plan → "Missing Index Details"
   - SQL wygeneruje `CREATE INDEX` statement – skopiuj i wykonaj
   - **UWAGA:** Nie twórz ślepo wszystkich sugerowanych indeksów (zbyt dużo indeksów spowalnia INSERT/UPDATE)

3. **Sprawdź warnings (żółte trójkąty)**
   - Implicit conversion (np. VARCHAR vs NVARCHAR) → popraw typ danych lub zapytanie
   - Missing statistics → `UPDATE STATISTICS [tabela]`

4. **Index/Table Scan na dużych tabelach** 
   - = brakuje indeksu na kolumnie w WHERE/JOIN
   - Dodaj indeks na tej kolumnie

***

## **TL;DR – Praktyczny workflow dla Płatnika:**

```
1. Włącz Query Store w bazie Płatnika (autopilot monitoring)
2. Wykonaj wolną operację w Płatniku
3. Uruchom zapytanie do Query Store (powyżej) → znajdź wolne zapytanie
4. Skopiuj execution plan → otwórz w SSMS
5. Sprawdź Missing Index + Warnings
6. Utwórz brakujące indeksy / UPDATE STATISTICS
```

**Jeśli Query Store nie działa lub baza jest za mała:**
- Extended Events (najlepsza opcja dla SQL Express)
- Lub DMV cache query (opcja 5) – zero setup, działa od razu

***

### Czego NIE wiem vs co założyłem:

**Założyłem:**
- Masz dostęp do SSMS (to darmowe, też dla SQL Express)
- Nie robisz tego codziennie → więc Extended Events może być zbyt techniczne

**NIE wiem:**
- Czy używasz SQL Express (brak Profilera) czy Standard/Enterprise
- Jak duża jest baza (Query Store może być overkill dla 100MB bazy)
- Czy w ogóle masz uprawnienia sysadmin na SQL (bez tego nie uruchomisz Profiler/Extended Events)

Jeśli masz SQL Express i nie chcesz bawić się w Extended Events – **Query Store + DMV cache (opcja 4+5) to najprostsze rozwiązanie bez instalowania niczego dodatkowego**.
