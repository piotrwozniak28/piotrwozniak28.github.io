---
layout: post
title: Table functions
subtitle: SELECT * FROM TABLE (...);
thumbnail-img: /assets/img/table_functions_050.png
tags: [DB, SQL, PL/SQL]
---

Funkcja to konstrukt, zwracający wartość do środowiska, z którego został wywołany

{: .box-note}
**Krótko o nazewnictwie**<br/><br/>
W nazwie funkcji nie używam czasownika ```get/fetch/generate```.<br><br/>Funkcja zawsze **coś** zwraca, więc lepiej od razu przejść do rzeczy i w nazwie odpowiedzieć na pytanie "**co** zwraca ta funkcja".<br/>
Im więcej mamy funkcji dotyczących np. obiektu ```employees```, tym bardziej specyficzni musimy być w nazewnictwie. 

{: .box-error}
**ŹLE:**<br/>get_employee_ids<br/>get_random_string<br/>get_meetings

{: .box-success}
**DOBRZE:**<br/>employee_ids<br/>random_string<br/>meeting_dates

## 1. Co zwraca funkcja

Funkcja może zwracać wartość skalarną lub złożoną (w praktyce oznacza to ```record``` lub ```collection``` np. ```Nested Table```).

### 1.1 Funkcja zwraca wartość skalarną

{% highlight sql linenos %}
CREATE OR REPLACE FUNCTION random_string(in_length IN PLS_INTEGER)
RETURN VARCHAR2
AUTHID DEFINER IS
   co_random_string_option CONSTANT CHAR(1) := 'u';
   l_random_string         VARCHAR2(5);
BEGIN
   l_random_string := DBMS_RANDOM.string (co_random_string_option
                                         ,in_length);
   RETURN l_random_string;
END random_string;
/
SELECT random_string(5) FROM dual;
{% endhighlight %}

<a href="/assets/img/table_functions_055.png"><img src="/assets/img/table_functions_055.png" alt="table_functions_055.png" target="_blank"></a>

### 1.2 Funkcja zwraca wartość złożoną

#### 1.2.1 Funkcja zwraca kolekcję

{% highlight sql linenos %}
CREATE OR REPLACE TYPE t_employees_type IS TABLE OF VARCHAR2 (100);
/
CREATE OR REPLACE FUNCTION random_employees_one_col (in_count IN PLS_INTEGER)
RETURN t_employees_type
AUTHID DEFINER IS
   co_lower_bound           CONSTANT SIMPLE_INTEGER := 1;
   co_emp_first_name_length CONSTANT SIMPLE_INTEGER := 5;
   co_emp_last_name_length  CONSTANT SIMPLE_INTEGER := 7;
   co_emp_sep               CONSTANT CHAR(1)        := ' ';
   co_random_string_option  CONSTANT CHAR(1)        := 'u';
   t_employees              t_employees_type        := t_employees_type ();
BEGIN
   t_employees.EXTEND (in_count);
   <<add_random_employees>>
   FOR i IN co_lower_bound..in_count
   LOOP
      t_employees (i) := DBMS_RANDOM.string (co_random_string_option
                                            ,co_emp_first_name_length) || 
                                             co_emp_sep                || 
                         DBMS_RANDOM.string (co_random_string_option
                                            ,co_emp_last_name_length);
   END LOOP add_random_employees;
   RETURN t_employees;
END;
/
SELECT random_employees_one_col(5)
  FROM dual;
/
{% endhighlight %}

<a href="/assets/img/table_functions_060.png"><img src="/assets/img/table_functions_060.png" alt="table_functions_060.png" target="_blank"></a>

#### 1.2.2 Funkcja zwraca kolekcję w postaci relacyjnej (tabeli) w pojedynczej kolumnie.

Nazwa ```Table Function``` odnosi się nie tyle do samego kodu funkcji, co do sposobu jej wywołania.
Funkcję ```random_employees_one_col``` z przykładu powyżej możemy uruchomić na 2 sposoby:

##### 1.2.2.1 Uruchamiamy jako "normalną" funkcję (czyli identycznie, jak w punkcie *1.2.1*)

```
SELECT random_employees_one_col(5)
  FROM dual;
/
```

<a href="/assets/img/table_functions_060.png"><img src="/assets/img/table_functions_060.png" alt="table_functions_060.png" target="_blank"></a>

##### 1.2.2.2 Uruchamiamy jako ```Table Function```

```
SELECT *
  FROM TABLE (random_employees_one_col(5));
/
```

<a href="/assets/img/table_functions_065.png"><img src="/assets/img/table_functions_065.png" alt="table_functions_065.png" target="_blank"></a>

Wystarczy tylko umieścić funkcję (razem z parametrem) w klauzuli ```TABLE```, którą odpytujemy tak, jak normalną tabelę - np. ```employees```.

{: .box-note}
Od wersji ```Oracle Database 12c Release 2 (12.2.0.1.0)```, użycie klauzli ```TABLE``` jest opcjonalne. Moglibyśmy po prostu napisać ```SELECT * FROM random_employees_one_col(5);```

Forma relacyjna może być zwrócona dla 1 z 2 typów danych: 
* ```Nested Table```
* ```Varray```

Każy inny typ wywoła błąd. Spróbujmy użyć "skalarnej" funkcji (typ zwracanych wartości to ```VARCHAR2```) z początku artykułu:

```
SELECT * 
  FROM TABLE (random_string(5));
/
```

<a href="/assets/img/table_functions_070.png"><img src="/assets/img/table_functions_070.png" alt="table_functions_070.png" target="_blank"></a>

#### 1.2.3 Funkcja zwraca kolekcję w postaci relacyjnej (tabeli) w wielu kolumnach.

Sprawa odrobinę się komplikuje. Musimy najpierw zdeklarować typ ```Object```, którego instancje zastąpią pojedyncze wartości (jak np. ```'RANDO MVALU'```) z poprzedniej kolekcji.

{% highlight sql linenos %}
CREATE OR REPLACE TYPE employee_ot IS OBJECT (
   first_name VARCHAR2(5 CHAR)
  ,last_name  VARCHAR2(7 CHAR));
/
CREATE OR REPLACE TYPE t_employees_type IS TABLE OF employee_ot;
/
--
create or replace FUNCTION radom_employees_multi_cols (in_count IN PLS_INTEGER)
RETURN t_employees_type
AUTHID DEFINER IS
   co_lower_bound           CONSTANT SIMPLE_INTEGER := 1;
   co_emp_first_name_length CONSTANT SIMPLE_INTEGER := 5;
   co_emp_last_name_length  CONSTANT SIMPLE_INTEGER := 7;
   co_random_string_option  CONSTANT CHAR(1)        := 'u';
   t_employees              t_employees_type        := t_employees_type ();
BEGIN
   t_employees.EXTEND (in_count);
   <<add_random_employees>>
   FOR i IN co_lower_bound..in_count
   LOOP
      t_employees (i) := employee_ot (DBMS_RANDOM.string (co_random_string_option
                                                         ,co_emp_first_name_length)
                                     ,
                                      DBMS_RANDOM.string (co_random_string_option
                                                         ,co_emp_last_name_length));
   END LOOP add_random_employees;
   RETURN t_employees;
END;
/
{% endhighlight %}

Tak jak w przypadku funkcji z pojedynczą kolumną, możemy uruchomić tę funkcję jako "normalną" funkcję lub jako ```Table Function```.

##### 1.2.3.1 Uruchamiamy jako "normalną" funkcję

```
SELECT radom_employees_multi_cols(50) 
  FROM dual;
/
```

<a href="/assets/img/table_functions_080.png"><img src="/assets/img/table_functions_080.png" alt="table_functions_080.png" target="_blank"></a>

##### 1.2.3.2 Uruchamiamy jako ```Table Function```

```
SELECT * 
  FROM TABLE (radom_employees_multi_cols(50));
/
```

<a href="/assets/img/table_functions_075.png"><img src="/assets/img/table_functions_075.png" alt="table_functions_075.png" target="_blank"></a>

---

Tabelę (a dokładniej - instancję kolekcji), stworzoną przez ```Table Function``` możemy odpytywać indentycznie jak "zwykłą" tabelę.<br/><br/>
Przykłady takich query:

##### NATURAL JOIN

```
SELECT *
  FROM TABLE (radom_employees_multi_cols(1))
            ,(radom_employees_multi_cols(1));
```

##### UNION

```
SELECT *
  FROM TABLE (radom_employees_multi_cols(1))
 UNION ALL
SELECT *
  FROM TABLE (radom_employees_multi_cols(1));
```

##### CREATE TABLE AS (...)

```
CREATE TABLE raem AS SELECT * 
  FROM TABLE (radom_employees_multi_cols(1));
/
SELECT * FROM raem;
```

##### CREATE VIEW

```
CREATE OR REPLACE VIEW random_employees_v AS
   SELECT *
     FROM TABLE (radom_employees_multi_cols(10));
/
SELECT *
  FROM random_employees_v;
```

##### CREATE MATERIALIZED VIEW

```
CREATE MATERIALIZED VIEW random_employees_mv AS
   SELECT *
     FROM TABLE (radom_employees_multi_cols(10));
/
SELECT *
  FROM random_employees_mv;
```

{: .box-note}
Widok to tylko konstrukt logiczny - każdy ```SELECT``` (z prawdopodobieństwem ekstremalnie bliskim ```P = 1```) wygeneruje zawartość kolekcji różną od poprzedniej.<br/>
W przypadku widoku zmaterializowanego, przy kolejnych ```SELECT```, dane będą oczywiście takie, jak w momencie jego stworzenia.

## Źródła:
1. [https://docs.oracle.com/cd/B19306_01/appdev.1...](https://docs.oracle.com/cd/B19306_01/appdev.102/b14289/dcitblfns.htm#CHDJEGHC)
2. [https://www.youtube.com/watch?v=YixSfXZ8wPI](https://www.youtube.com/watch?v=YixSfXZ8wPI)