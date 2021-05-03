---
layout: post
title: Pipelined Table Functions
subtitle: PIPE ROW!
thumbnail-img: /assets/img/pipelined_table_functions_050.png
tags: [DB, SQL, PL/SQL]
---

```Pipelined Function``` to szczególny rodzaj  ```Table Function```

O ```Table Functions``` napisałem [tutaj](https://piotrwozniak.net/table_functions/).


{: .box-note}
**Różnica między pipelined a "klasyczną" table function:**<br/><br/>```Table Function``` zwraca wiersze dopiero po wygenerowaniu całego wyniku (a konkretniej - po wykonaniu klauzli ```RETURN```).<br/>```Pipelined Function``` zwraca wiersze w miarę jak są obliczane.

### Przykładowa ```Pipelined Function```
Nasza funkcja będzie zwracać zadaną liczbę elementów (a konkretnie: ```n+1``` elementów) z ciągu z silnią (```n!```).

{% highlight sql linenos %}
CREATE TYPE t_factorial_type IS TABLE OF NUMBER;
/
CREATE OR REPLACE FUNCTION factorial (in_argument SIMPLE_INTEGER) 
RETURN t_factorial_type DETERMINISTIC PIPELINED IS
   co_empty_product       CONSTANT SIMPLE_INTEGER := 0;
   co_empty_product_value CONSTANT SIMPLE_INTEGER := 1;
   co_lower_bound         CONSTANT SIMPLE_INTEGER := 1;
   l_product              NUMBER                  := 1;
BEGIN
   CASE WHEN in_argument = 0 THEN
           PIPE ROW (co_empty_product_value);     
        WHEN in_argument > co_empty_product THEN
           PIPE ROW (co_empty_product_value);     
           FOR i IN co_lower_bound..in_argument 
           LOOP <<calculate_factorial>> 
              l_product := l_product * i;
              PIPE ROW (l_product);     
        END LOOP calculate_factorial;
   END CASE;
RETURN;
END;
/
{% endhighlight %}

| Wiersz | Opis |
|:------:|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| 4 |Dodajemy keyword ```PIPELINED```.<br/>```DETERMINISTIC``` jest tu opcjonalne. Odrobinę na ten temat pod tabelą.|
| 12 |Określamy, co ma być zwracane. Zawartość wiersza można potraktować jako "dynamiczny RETURN"| 
| 20 |Spełniamy semantyczny obowiązek (wartość zwracamy przecież z użyciem ```PIPE ROW```)|

{: .box-note}
**DETERMINISTIC** - to klauzula, która oznacza funkcję jako deterministyczną (sic!).<br/><br/>
Funkcja jest determnistyczna, jeśli jej output jest zależny wyłącznie od inputu.<br/>
Innymi słowy - dla określonych danych wejściowych (np. dla ```5```), zawsze otrzymamy te same dane wyjściowe (dla tego przykładu - ```[1,1,2,6,24,120]```).<br/>
Temat jest zdecydowanie bardziej złożony - polecam lekturę załączników na dole artykułu.

Taką funkcję uruchamiamy (w postaci relacyjnej) na 1 z 2 tożsamych sposobów:

```
SELECT * 
  FROM TABLE (factorial(100)); 
```

...lub bezpośrednio, bez ```FROM TABLE(...)``` (dla Oracle 12.2 i wyższych)

```
SELECT * 
  FROM factorial(100); 
```

<a href="/assets/img/pipelined_table_functions_060.png"><img src="/assets/img/pipelined_table_functions_060.png" alt="pipelined_table_functions_060.png" target="_blank"></a>

Wynik dla 84! ma 125 cyfr. Wynik dla 85! przekracza maksymalną wartość ```precision``` (126). 
Zamiast kolejnych wartości, otrzymujemy Inf[i](https://www.youtube.com/watch?v=jzy2dgEUOhY)nity.

---





## Źródła:
1. [https://docs.oracle.com/cd/B19306_01/appdev.1...](https://docs.oracle.com/cd/B19306_01/appdev.102/b14289/dcitblfns.htm#CHDJEGHC)
2. [http://stevenfeuersteinonplsql.blogspot.com/2...](http://stevenfeuersteinonplsql.blogspot.com/2017/05/deterministic-functions-caching-and.html)
2. [http://rwijk.blogspot.com/2008/04/determinist...](http://rwijk.blogspot.com/2008/04/deterministic-clause.html)
