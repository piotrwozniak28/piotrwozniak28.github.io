---
layout: post
title: UNION vs UNION ALL
subtitle: Operatory zbiorowe
image: /img/set_operators_050.png
tags: [DB, SQL]
---

## Są 4 operatory zbiorowe

Operatory zbiorowe (```Set Operations```) są dobrze ilustrowane przez diagramy Venna (w przeciwieństwie do ```JOINów```)<br/>Wspomniałem już o tym [w tym poście](https://piotrwozniak.net/2019-02-10-greatest_n_per_group/)

<a href="/img/set_operators_055.png"><img src="/img/set_operators_055.png" alt="set_operators_055.png" target="_blank"></a>

## Zasady odnośnie pisania ```Set Operations```:

{% highlight sql linenos %}
SELECT col_1
      ,1
--      ,2
  FROM test_1
-- ORDER BY col_1
 UNION ALL
SELECT col_2
      ,2
  FROM test_2
 ORDER BY col_1
--         ,col_2
;
{% endhighlight %}


<a href="/img/set_operators_060.png"><img src="/img/set_operators_060.png" alt="set_operators_060.png" target="_blank"></a>


#### TYP DANYCH vs GRUPA TYPÓW DANYCH

{: .box-note}
**Jedna z "grup typów danych" to ```Character data types```.**<br/><br/>
Należą do niej:<br/>CHAR<br/>NCHAR<br/>VARCHAR2<br/>NVARCHAR2<br/><br/>W obrębie danej grupy, ```Set Operators``` dokonują niejawnej konwersji (np. rezultat łączenia ```CHAR``` z ```VARCHAR2``` to ```VARCHAR2```).<br/>Jeśli spróbujemy wykonać operację zbiorową (np. ```MINUS```) dla kolumn z różnych grup typów, to Oracle zwróci błąd.<br/><br/>Więcej informacji (np. o pierwszenństwie typów danych podczas konwersji) [w dokumentacji](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Data-Types.html)

## UNION vs UNION ALL

```UNION``` robi wszystko to, co ```UNION ALL```, ale dotatkowo usuwa duplikaty.
Zabieg usunięcia duplikatów niesie za sobą pewne konsekwencje. 

#### 1. Query wykona się wolniej

Stwórzmy 2 testowe tabele z danymi.

{% highlight sql linenos %}
DROP TABLE test_1;
DROP TABLE test_2;
/
CREATE TABLE test_1 AS 
    SELECT level AS num
      FROM dual
   CONNECT BY level <= 4;
/
CREATE TABLE test_2 AS
    SELECT level AS num
      FROM dual
     WHERE MOD(level,2) = 0
   CONNECT BY level <= 4;
/
{% endhighlight %}

Zbieramy statystyki dla planu wykonania query z użyciem ```UNION ALL```

{% highlight sql linenos %}
SELECT /*+ gather_plan_statistics */ 
       *
  FROM test_1
 UNION ALL
SELECT *
  FROM test_2;
/
SELECT * 
  FROM dbms_xplan.display_cursor(); 
/
{% endhighlight %}

<a href="/img/set_operators_065.png"><img src="/img/set_operators_065.png" alt="set_operators_065.png" target="_blank"></a>

...A teraz dla ```UNION```

{% highlight sql linenos %}
SELECT /*+ gather_plan_statistics */ 
       *
  FROM test_1
 UNION 
SELECT *
  FROM test_2;
/
SELECT * 
  FROM dbms_xplan.display_cursor(); 
/
{% endhighlight %}

<a href="/img/set_operators_070.png"><img src="/img/set_operators_070.png" alt="set_operators_070.png" target="_blank"></a>

Aby usunąć duplikaty, wyniki zostały posortowane.<br/>
Wynik takiego query będzie identyczny jak ```SELECT DISTINCT...```

{% highlight sql linenos %}
SELECT DISTINCT /*+ gather_plan_statistics */ 
               *
  FROM (SELECT *
          FROM test_1
         UNION ALL 
        SELECT *
          FROM test_2);
/
SELECT * 
  FROM dbms_xplan.display_cursor(); 
/
{% endhighlight %}

...Jednak w drugim przykładzie została użyta bardziej wydajna (zaimplementowana w DBMS 10.2.0) wersja algorytmu sortującego - ```HASH UNIQUE```.

<a href="/img/set_operators_075.png"><img src="/img/set_operators_075.png" alt="set_operators_075.png" target="_blank"></a>

Możemy zabronić użycia tego algorytmu. Wtedy plan wykonania będzie identyczny jak w przypadku ```UNION```

{% highlight sql linenos %}
ALTER SESSION SET "_gby_hash_aggregation_enabled" = FALSE;
/
SELECT DISTINCT /*+ gather_plan_statistics */ 
               *
  FROM (SELECT *
          FROM test_1
         UNION ALL 
        SELECT *
          FROM test_2);
/
SELECT * 
  FROM dbms_xplan.display_cursor(); 
/
{% endhighlight %}

#### 2. Żeby usunąć duplikaty, trzeba najpierw porównać wartości. Oracle nie wspiera tej funkcjonalności dla BLOB I CLOB.

Tak jak poprzednio - tworzymy 2 tabele. W każdej 1 kolumna - tym razem celowo zdeklarowany CLOB.

{% highlight sql linenos %}
DROP TABLE test_3;
DROP TABLE test_4;
/
CREATE TABLE test_3 (
   large_object CLOB);
/
CREATE TABLE test_4 (
   large_object CLOB);
/
INSERT INTO test_3 (large_object) VALUES (1);
INSERT INTO test_3 (large_object) VALUES (2);
INSERT INTO test_3 (large_object) VALUES (3);
INSERT INTO test_3 (large_object) VALUES (4);
INSERT INTO test_4 (large_object) VALUES (2);
INSERT INTO test_4 (large_object) VALUES (4);
--   ***********************************
SELECT /*+ gather_plan_statistics */ 
       *
  FROM test_3
 UNION ALL
SELECT *
  FROM test_4;
/
SELECT * 
  FROM dbms_xplan.display_cursor(); 
/
SELECT /*+ gather_plan_statistics */ 
       *
  FROM test_3
 UNION 
SELECT *
  FROM test_4;
/
SELECT * 
  FROM dbms_xplan.display_cursor(); 
/
{% endhighlight %}

O ile ```UNION ALL``` nie sprawia problemu, to ```UNION``` już nie przejdzie.

<a href="/img/set_operators_080.png"><img src="/img/set_operators_080.png" alt="set_operators_080.png" target="_blank"></a>

Moglibyśmy porównać zawartość tych kolumn np. za pomocą funkcji ```TO_CHAR```, która zawsze zwraca ```VARCHAR2```.

{% highlight sql linenos %}
SELECT /*+ gather_plan_statistics */ 
       TO_CHAR(large_object)
  FROM test_3
 UNION 
SELECT TO_CHAR(large_object)
  FROM test_4;
/
SELECT * 
  FROM dbms_xplan.display_cursor(); 
/
{% endhighlight %}

## Źródła:
1. [https://docs.oracle.com/en/database/oracle/or...](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Data-Types.html)
2. [https://asktom.oracle.com/pls/asktom/f?p=100:...](https://asktom.oracle.com/pls/asktom/f?p=100:11:0::::P11_QUESTION_ID:32961403234212)
3. [https://docs.oracle.com/cd/B19306_01/server.1...](https://docs.oracle.com/cd/B19306_01/server.102/b14200/queries004.htm#i2054381)