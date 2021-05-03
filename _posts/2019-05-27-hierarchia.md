---
layout: post
title: Hierarchiczne queries
subtitle: Who's the boss?
image: /img/connect_by_050.png
tags: [DB, SQL]
---

Zaczniemy od problemu

*(działamy na tabeli 'employees' z domyślnego schematu HR od Oracle. Do pobrania choćby [tutaj](https://pl.scribd.com/doc/2719944/FULL-Script-Human-Resources-for-Oracle-Database))*

{: .box-warning}
Wyświetl ```first_name``` i ```last_name``` każdego z pracowników wraz z ```first_name``` i ```last_name``` ich menadżera.

{% highlight sql linenos %}
SELECT e1.employee_id
      ,e1.manager_id
      ,e1.first_name
      ,e1.last_name
      ,'is employee of'
      ,e2.first_name
      ,e2.last_name
  FROM employees e1
  JOIN employees e2
    ON e1.manager_id = e2.employee_id
 ORDER BY e1.manager_id NULLS FIRST;
{% endhighlight %}

<a href="/img/connect_by_051.png"><img src="/img/connect_by_051.png" alt="connect_by_051.png" target="_blank"></a>

Wystarczył self-join. Trochę więcej niż "SELECT gwiazdka" ale nadal nic skomplikowanego.

---

Wiemy już, kto dla kogo jest bezpośrednim przełożonym.<br/>Nie jesteśmy jednak w stanie odpowiedzieć na pytanie:

{: .box-warning}
 **Ilu** szefów ma nad sobą dany pracownik?

Z pomocą przychodzą nam hierarchiczne queries, które są idealne do obsługiwania relacji typu dziecko-rodzic.

Poniżej prosty przykład query z wykorzystaniem ```START WITH``` i ```CONNECT BY```

{% highlight sql linenos %}
 SELECT first_name
       ,last_name
       ,employee_id
       ,manager_id
       ,level
   FROM employees 
  WHERE level <= 1
  START WITH manager_id IS NULL
CONNECT BY PRIOR employee_id =  manager_id;
{% endhighlight %}

<a href="/img/connect_by_054.png"><img src="/img/connect_by_054.png" alt="connect_by_054.png" target="_blank"></a>

Różnica pomiędzy powyższym a "konwencjonalnym" query, zawiera się w 3 wierszach (8, 9, 5)

| Wiersz | Opis |
|:------:|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| 8 |Definiuje wartość, którą uznajemy za root (```level = 1```) Dla tego zbioru danych, zapis ```START WITH employee_id = 100``` dałby ten sam wynik, jednak będzie mniej elastyczny (bo co, jeśli ```employee_id``` "szefa szefów" się zmieni?).| 
| 9 |Określa, w jaki sposób mamy tworzyć kolejne stopnie hierarchii. Przykład zilustrowany poniżej|
| 5 |**Opcjonalna** pseudokolumna, która pokazuje poziom drzewa (zaczynając od root - czyli w naszym przypadku wiersza, gdzie ```manager_id IS NULL```). Można jej użyć wyłącznie w połączeniu ze ```START WITH``` i ```CONNECT BY```|

Wychodzimy od deklaracji z wiersza #8 i zaczynamy rysować drzewo od pracownika, który nie ma menadżera.<br/>
Warunek z wiersza #9 (```CONNECT BY PRIOR employee_id =  manager_id```) określa, jak tworzyć kolejne stopnie hierarchii. 

Mimo że query zapisane jest prawidłowo, to otrzymaliśmy tylko 1 wiersz. 

Winowajcą jest ```WHERE level <=1``` Stopnie hierarchii/drzewa zaczynamy liczyć od 1, więc zostaliśmy wyłącznie z wierszami (w tym przypadku - wierszem) spełniającymi warunek startowy.

W konsekwencji - kolejne warunki ```CONNECT BY``` (wiersze 10-13 poniżej) nie wpływają na wynik. Zawsze dostaniemy tylko ten 1. ```level``` i kolejne stopnie hierarchii nie zostaną utworzone.

{% highlight sql linenos %}
 SELECT first_name
       ,last_name
       ,employee_id
       ,manager_id
       ,level
   FROM employees 
  WHERE level <= 1
  START WITH manager_id IS NULL
CONNECT BY PRIOR employee_id =  manager_id
             AND manager_id < 5000
             AND employee_id >= 80000
             AND manager_id IS NULL
             AND employee_id/manager_id = 0;
{% endhighlight %}

<a href="/img/connect_by_055.png"><img src="/img/connect_by_055.png" alt="connect_by_055.png" target="_blank"></a>

---

Okej, zobaczmy kto pracuje bezpośrednio pod Stevenem Kingiem. 

{% highlight sql linenos %}
 SELECT first_name
       ,last_name
       ,employee_id
       ,manager_id
       ,level
   FROM employees 
  WHERE level <= 2
  START WITH manager_id IS NULL
CONNECT BY PRIOR employee_id =  manager_id;
{% endhighlight %}

<a href="/img/connect_by_060.png"><img src="/img/connect_by_060.png" alt="connect_by_060.png" target="_blank"></a>

Lista została utworzona w następujący sposób:
1. Zwróć **wiersze** spełniające warunek ```START WITH manager_id IS NULL```.
2. Dla każdego z **tych wierszy**,  zwróć wiersze, które w kolumnie ```manager_id``` mają wartość ```PRIOR employee_id```.
```PRIOR``` odnosi się do poprzedniego poziomu hierarchii (```level```).

   ...Czyli innymi słowy: 

   Dla ```employee_id = 100``` (```level = 1```) zwróć wszystkie wiersze z ```manager_id = 100``` (```level = 2```). Z perspektywy ```level = 2```, ```level = 1``` jest ```PRIOR```

Najłatwiej to zrozumieć przez analizę wyników query. Poniżej "odsłonięte" kolejne 2 poziomy.

{% highlight sql linenos %}
 SELECT first_name
       ,last_name
       ,employee_id
       ,manager_id
       ,level
   FROM employees 
  WHERE level <= 3
  START WITH manager_id IS NULL
CONNECT BY PRIOR employee_id =  manager_id;
{% endhighlight %}

<a href="/img/connect_by_065.png"><img src="/img/connect_by_065.png" alt="connect_by_065.png" target="_blank"></a>

{% highlight sql linenos %}
 SELECT first_name
       ,last_name
       ,employee_id
       ,manager_id
       ,level
   FROM employees 
  WHERE level <= 4
  START WITH manager_id IS NULL
CONNECT BY PRIOR employee_id =  manager_id;
{% endhighlight %}

<a href="/img/connect_by_070.png"><img src="/img/connect_by_070.png" alt="connect_by_070.png" target="_blank"></a>

Przy używaniu funkcji hierarchicznych, Oracle umożliwia nam użycie dodatkowych funkcji - np. ```SYS_CONNECT_BY_PATH```, która zwraca ścieżkę dla danej wartości - rozpoczynając od root.

{% highlight sql linenos %}
 SELECT first_name
       ,last_name
       ,employee_id
       ,manager_id
       ,level
       ,LPAD(' ',3 * level - 3) || SYS_CONNECT_BY_PATH(last_name, '/') AS path
   FROM employees 
  WHERE level <= 4
  START WITH manager_id IS NULL
CONNECT BY PRIOR employee_id =  manager_id;
{% endhighlight %}

<a href="/img/connect_by_075.png"><img src="/img/connect_by_075.png" alt="connect_by_075.png" target="_blank"></a>

---

Stwórzmy jeszcze sytuację, gdzie dane zawierają pętlę (np. szef A ma podwładnego B, który ma podwładnego C, który jest szefem dla A).

```
DROP TABLE hierarchy;
/
CREATE TABLE hierarchy (
   id NUMBER(*,0) PRIMARY KEY 
  ,name CHAR(1)
  ,boss_id NUMBER(*,0));
/
INSERT INTO hierarchy (id, name, boss_id) VALUES (1, 'A', '3');
INSERT INTO hierarchy (id, name, boss_id) VALUES (2, 'B', '1');
INSERT INTO hierarchy (id, name, boss_id) VALUES (3, 'C', '2');
/
```

---

{% highlight sql linenos %}
 SELECT id
       ,name
       ,boss_id
   FROM hierarchy
CONNECT BY PRIOR id = boss_id;
{% endhighlight %}

<a href="/img/connect_by_080.png"><img src="/img/connect_by_080.png" alt="connect_by_080.png" target="_blank"></a>

 Oracle umożliwia użycie parametru ```CONNECT BY **NOCYCLE**```, który spowoduje zwrócenie danych (domyślnie taki układ spowodowałby błąd).

{% highlight sql linenos %}
 SELECT id
       ,name
       ,boss_id
       ,CONNECT_BY_ISCYCLE
   FROM hierarchy
CONNECT BY NOCYCLE PRIOR id = boss_id;
{% endhighlight %}

<a href="/img/connect_by_085.png"><img src="/img/connect_by_085.png" alt="connect_by_085.png" target="_blank"></a>

Identyfikacja wiersza, który zawiera pętlę, jest możliwa dzięki pseudokolumnie ```CONNECT_BY_ISCYCLE```.

---

{: .box-note}
Dzięki CONNECT BY możemy wygenerować dowolną liczbę wierszy. Wystarczy, że napiszemy warunek tworzenia kolejnych poziomów hierarchii, który w oczekiwanym zakresie będzie prawdziwy.

{% highlight sql linenos %}
 SELECT level
   FROM dual
CONNECT BY level <= 100;
{% endhighlight %}

<a href="/img/connect_by_090.png"><img src="/img/connect_by_090.png" alt="connect_by_090.png" target="_blank"></a>
