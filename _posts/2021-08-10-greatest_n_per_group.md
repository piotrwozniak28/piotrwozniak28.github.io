---
layout: post
title: greatest-n-per-group
subtitle: One query to select them all
thumbnail-img: /assets/img/greatest-n-per-group_avatar.jpg
tags: [DB, SQL]
---

## Wybierzemy osoby, które mają najwięcej

#### Krótki spacer po różnych sposobach na wybór:
* maksymalnej wartości ze zbioru
* maksymalnych wartości z podzbiorów wraz ze skorelowanymi informacjami
* n-tych wartości z podzbiorów wraz ze skorelowanymi informacjami

Wszystko w obrębie Shire - do Mordoru jeszcze trochę;)<br/><br/>
...Tak sobię myślę, że blog ma się tak do postów, jak Hobbit do posiłków.<br/>
Dobrze, jeśli są regularne, obfite i częste

Używam domyślnego schematu ```hr```

### 1. Jakie są najwyższe zarobki w firmie?

```sql
SELECT MAX (salary) AS max_salary
  FROM employees;
```

<a href="/assets/img/greatest-n-per-group_050.png"><img src="/assets/img/greatest-n-per-group_050.png" alt="greatest-n-per-group_050.png" target="_blank"></a>

### 2. Jakie są najwyższe zarobki w danych oddziałach firmy?

```sql
SELECT department_id
      ,MAX (salary) AS max_salary
  FROM employees
 GROUP BY department_id
 ORDER BY department_id;
```

<a href="/assets/img/greatest-n-per-group_051.png"><img src="/assets/img/greatest-n-per-group_051.png" alt="greatest-n-per-group_051.png" target="_blank"></a>

### 3. Jakie są najwyższe zarobki w danych oddziałach firmy?<br/>Wyświetl wszystkie informacje o osobach, które tyle zarabiają.

W tym miejscu pojawia się problem. Nie możemy po prostu dodać kolejnych kolumn pod SELECT.

```sql
SELECT department_id
      ,first_name
      ,MAX (salary) AS max_salary
  FROM employees
 GROUP BY department_id
 ORDER BY department_id;
```

<a href="/assets/img/greatest-n-per-group_060.png"><img src="/assets/img/greatest-n-per-group_060.png" alt="greatest-n-per-group_060.png" target="_blank"></a>

Żeby wykonać takie query, musielibyśmy dodać też 2. kryterium grupowania.

```sql
SELECT department_id
      ,first_name
      ,MAX (salary) AS max_salary
  FROM employees
 GROUP BY department_id
         ,first_name
 ORDER BY department_id;
```

<a href="/assets/img/greatest-n-per-group_061.png"><img src="/assets/img/greatest-n-per-group_061.png" alt="greatest-n-per-group_063.png" target="_blank"></a>

{: .box-error}
**W REZULTACIE:** wyniki zostaną okrojone tylko o te osoby, które dla danego ```department_id``` otrzymują mniejsze ```salary``` od osoby z tym samym ```first_name```

{: .box-success}
**Ten problem jest dobrze opisany**
Na Stack Overflow jest nawet osobny [TAG [greatest-n-per-group]](https://stackoverflow.com/questions/tagged/greatest-n-per-group) - specjalnie dla tego typu wyzwań (stąd też nazwa tego artykułu).<br/><br/>
Proponuję 2 podejścia:<br/><br/>**Self-join**<br/>**Funkcja analityczna**

### 3.1 SELF JOIN

```sql
SELECT e1.department_id
      ,e1.first_name
      ,e1.salary
  FROM employees e1
  LEFT JOIN employees e2 ON e1.department_id = e2.department_id
   AND e2.salary > e1.salary
 WHERE e2.salary IS NULL;
```

Temu query należy się kilka słów wyjaśnienia. Podam kilka kroków, w jakich sam - na drodze reverse engineeringu próbowałbym je zrozumieć.

---

**Chcę poznać zasadę działania.** Zbiór danych, na których się uczę, powinien być nie większy, niż to absolutnie konieczne. Mógłbym na szybko stworzyć jakieś dummy data, ale najpierw sprawdzę, czy mogę działać na tym, co już mam.

```sql
SELECT department_id
       ,COUNT (*)
  FROM employees e1
 GROUP BY department_id
 ORDER BY 1;
```

<a href="/assets/img/greatest-n-per-group_062.png"><img src="/assets/img/greatest-n-per-group_062.png" alt="greatest-n-per-group_062.png" target="_blank"></a>


Okej.  ```employees``` zawiera już ```department_id = 60``` z 5 wierszami, który posłuży mi za wygodny przykład.<br/>Stwórzmy najpierw prostego INNER JOINa. 

```sql
SELECT e1.first_name
      ,e1.salary
      ,'E1 <-- --> E2'
      ,e2.first_name
      ,e2.salary
  FROM employees e1
  JOIN employees e2 ON 1=1
   AND e1.department_id = e2.department_id
 WHERE 1=1
   AND e1.department_id = 60;
```

<a href="/assets/img/greatest-n-per-group_063.png"><img src="/assets/img/greatest-n-per-group_063.png" alt="greatest-n-per-group_063.png" target="_blank"></a>

W ramach danego ```department_id```, łączymy każdy wiersz z każdym wierszem.<br/>Czyli w rezultacie dla ```e1.department_id = 60``` mamy 5\*5 = 25 wierszy.<br/><br/>**\***Możemy to sobie zwizualizować jako 2 koncentryczne, mające tę samą średnicę zbiory na [diagramie Venna](https://en.wikipedia.org/wiki/Venn_diagram).<br/>**\*\***Tak długo, jak ```department_id``` nie ma wartości ```NULL```, to niezależnie od typu ```JOIN``` - wynik query nie będzie się różnił.

{: .box-note}
**\*Możemy, ALE:**<br/><br/>
Pamiętajmy, że diagramy Venna nie do końca oddają to, co zachodzi podczas ```JOIN``` [Say NO to Venn Diagrams When Explaining JOINs](https://blog.jooq.org/2016/07/05/say-no-to-venn-diagrams-when-explaining-joins/).<br/><br/>
[Tutaj dobra wizualizacja z Reddit \(r/programming\) do powyższego artykułu](https://i.imgur.com/xLTcOGi.png).

---

### Rozważmy przypadek, gdzie ```NULL``` jednak wystąpi:

```sql
SELECT e1.department_id
      ,e1.first_name
      ,e1.salary
      ,'E1 <-- --> E2'
      ,e2.department_id
      ,e2.first_name
      ,e2.salary
 FROM employees e1
 FULL JOIN employees e2 ON 1=1
  AND e1.department_id = e2.department_id
WHERE 1=1
  AND e1.department_id = 60
   OR e2.department_id IS NULL;
```

<a href="/assets/img/greatest-n-per-group_064.png"><img src="/assets/img/greatest-n-per-group_064.png" alt="greatest-n-per-group_064.png" target="_blank"></a>

{: .box-warning}
Dlaczego porównując (z samą sobą) **1** kolumnę ```department_id```, otrzymuję **2** wiersze, które nie spełniają warunku?

{: .box-success}
Być może to oczywista oczywistość, jednak u mnie wymagała chwili namysłu.<br/><br/>Za wszystko odpowiada fakt, że ```NULL != NULL```.<br/><br/>...A jeśli równość ```e1.department_id = e2.department_id``` nie występuje, to warunek nie został spełniony zarówno z perspektywy ```e1```, jak i ```e2``` (pomimo tego, że pod aliasami to ta sama tabela!). 

---

### Okej, wracamy do głównego nurtu.

O ``JOIN`` po ```department_id``` już zadbaliśmy. Dodajemy teraz relację z ```salary```. ...A właściwie to relacj**e** (wyjaśnienie w tabeli pod query)

{% highlight sql linenos %}
SELECT e1.department_id
      ,e1.first_name
      ,e1.salary
      ,'E1 <-- --> E2'
      ,e2.department_id
      ,e2.first_name
      ,e2.salary
 FROM employees e1
 FULL JOIN employees e2 ON 1=1
  AND e1.department_id = e2.department_id
  AND (e2.salary > e1.salary
       OR e2.salary < e1.salary
       OR e2.salary = e1.salary   
      )
WHERE 1=1
  AND e1.department_id = 60
   OR e2.department_id = 60
ORDER BY e1.salary NULLS FIRST
{% endhighlight %}

<a href="/assets/img/greatest-n-per-group_066.png"><img src="/assets/img/greatest-n-per-group_066.png" alt="greatest-n-per-group_066.png" target="_blank"></a>

| Wiersz | Opis |
|:------:|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| 9-12 |Aby lepiej zrozumieć działanie nierówności w ```INNER JOIN```, proponuję wyjść od neutralnego (przy założeniu, że w ```salary``` nie ma wartości ```NULL```) konstruktu.<br/><br/>Z tego miejsca, będziemy zakomentowywać dane wiersze  i sprawdzać różnice w wyniku zapytania.|
| 14-15 |Od momentu wprowadzenia nierówności, zapytanie zwróci takie wartości z ```e1```, dla których nie było odpowiednika w ```e2``` - i vice versa. Chcę działać ubytkowo, więc - póki co - piszę query,które obejmuje nawet nieużyteczne później przypadki.|

#### Komentujemy zbędne kolumny i warunki. 

{% highlight sql linenos %}
SELECT e1.department_id
      ,e1.first_name
      ,e1.salary
--      ,'E1 <-- --> E2'
--      ,e2.department_id
--      ,e2.first_name
--      ,e2.salary
 FROM employees e1
 FULL JOIN employees e2 ON 1=1
  AND e1.department_id = e2.department_id
  AND (e2.salary > e1.salary
--       OR e2.salary < e1.salary
--       OR e2.salary = e1.salary   
      )
WHERE 1=1
  AND e1.department_id = 60
--   OR e2.department_id = 60
    AND e2.salary IS NULL
--ORDER BY e1.salary NULLS FIRST
{% endhighlight %}

| Wiersz | Opis |
|:------:|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| 11,18 |Łączymy tam, gdzie pensja jest wyższa.<br/><br/>...No ale od najwyższej wyższa nie będzie - przecież taka jest cecha najwyższości(!) ;)<br/><br/>W takim przypadku, query zwróci ```e2.salary = NULL```, którą wybieramy w wierszu ```#18```.|

#### Pozostaje nam tylko uporządkować kod - zostajemy z wejściowym query.

```sql
SELECT e1.department_id
      ,e1.first_name
      ,e1.salary
  FROM employees e1
  LEFT JOIN employees e2 ON e1.department_id = e2.department_id
   AND e2.salary > e1.salary
 WHERE e2.salary IS NULL;
```

---

### 3.2 Funkcja analityczna

```sql
SELECT department_id
      ,first_name
      ,salary
  FROM (
   SELECT employees.*
         ,MAX (salary) OVER (
      PARTITION BY department_id
   ) max_salary
     FROM employees
)
 WHERE salary = max_salary
 ORDER BY department_id;
```

<a href="/assets/img/greatest-n-per-group_053.png"><img src="/assets/img/greatest-n-per-group_053.png" alt="greatest-n-per-group_053.png" target="_blank"></a>

Tutaj tłumaczenia będzie znacznie mniej.<br/>Zakładam, że konstrukt **subquery** lub **CTE** jest znany.<br/>Musimy użyć jednego z nich, bo ([za dokumentacją Oracle](https://docs.oracle.com/cd/E11882_01/server.112/e41084/functions004.htm#SQLRF06174)):

{: .box-note}
Analytic functions are the last set of operations performed in a query except for the final ORDER BY clause.<br/>All joins and all WHERE, GROUP BY, and HAVING clauses are completed before the analytic functions are processed.<br/>Therefore, analytic functions can appear only in the select list or ORDER BY clause.

### Funkcje agregujące vs analityczne - krótko o różnicy

Funkcje agregujące zwracają **pojedynczy** wiersz (wynik) dla wiersza lub grupy wierszy.<br/>

```sql
SELECT MAX (salary) AS max_salary
  FROM employees;
```

Funkcje analityczne zwracają **wiele** wierszy (wyników) dla wiersza lub grupy wierszy.
```sql
SELECT MAX (salary) OVER() AS max_salary
  FROM employees;
```

<a href="/assets/img/greatest-n-per-group_067.png"><img src="/assets/img/greatest-n-per-group_067.png" alt="greatest-n-per-group_067.png" target="_blank"></a>

O ile dla funkcji agregujących **grupy** są definiowane "globalnie" - tzn. dla danego query (klauzula ```WHERE```), to funkcje analityczne tworzą własne, "wewnętrzne" grupowanie.<br/>**Możemy więc wyświetlić zgrupowane rezultaty dla niezgrupowanych danych(!).**<br/><br/>
...Najłatwiej na przykładzie.

```sql
SELECT department_id
      ,MAX (salary) AS max_salary
  FROM employees
 GROUP BY department_id
 ORDER BY department_id;
```

```sql
SELECT department_id
      ,MAX (salary) OVER(PARTITION BY department_id) AS max_salary
  FROM employees
 ORDER BY department_id;  
```

<a href="/assets/img/greatest-n-per-group_068.png"><img src="/assets/img/greatest-n-per-group_068.png" alt="greatest-n-per-group_068.png" target="_blank"></a>

Korzystając z opisanej własności, obok zagregowanych wyników, wyświetlmy wszystkie (niezagregowane) wiersze z tabeli.

```sql
SELECT employees.*
      ,MAX (salary) OVER(PARTITION BY department_id) AS max_salary
  FROM employees
 ORDER BY department_id;
```

<a href="/assets/img/greatest-n-per-group_069.png"><img src="/assets/img/greatest-n-per-group_069.png" alt="greatest-n-per-group_069.png" target="_blank"></a>

Funkcje analityczne nie mogą zostać użyte w ```GROUP BY```. 

Żeby zagregować wyniki tak, aby wyświetlić wiersze tylko o pracownikach z maksymalnymi zarobkami w danym ```department_id```, musimy użyć subquery lub CTE.

...Co prowadzi nas do początkowego kodu:

```sql
SELECT department_id
      ,first_name
      ,salary
  FROM (
   SELECT employees.*
         ,MAX (salary) OVER (
      PARTITION BY department_id
   ) max_salary
     FROM employees
)
 WHERE salary = max_salary
 ORDER BY department_id;
```



### 4. Jakie są n-te zarobki w danych oddziałach firmy?<br/>Wyświetl wszystkie informacje o osobach, które tyle zarabiają.

One and only - **analytic query**. Na tym etapie nie widzę potrzeby, żeby męczyć się z innym rozwiązaniem.<br/>Pozyskanie informacji o osobach, zarabiających **n-te** z kolei zarobki (powiedzmy że **4-te**) można różnie zrozumieć.<br/>Poniżej query (i rezultat) użycia 3 funkcji, które w różny sposób przypisują kolejne wartości danym wierszom.<br/>Powinny objąć większość (jeśli nie wszystkie) możliwe odpowiedzi na postawione wcześniej pytanie.

Pierwsza różnica jest widoczna w wierszu #13 Query Result.

Funkcje ```RANK``` postawiły Stevena na tym samym (2) stopniu podium. Funkcja ```ROW_NUMBER``` po prostu kontynuowała numeracje wierszy - przydzielając Stevenowi trzecie miejsce ze względu na alfabetyczne sortowanie ```ORDER BY salary, first_name```.

Kolejna różnica - wiersz #14.

Funkcja ```DENSE_RANK``` dała 3 miejsce na podium dla Jamesa.

```RANK``` "przeskakuje" w numeracji o tyle wartości, ile na poprzedniej pozycji jest duplikatów.

Zaznaczyłem wiersze #16-20, które zawierają  5 tych samych wartości ```salary``` wewnątrz tego samego ```department_id```. Warto je przeanalizować wraz z sąsiadującymi wierszami. 

```sql
WITH department_salaries AS (
   SELECT first_name
         ,department_id   
         ,salary
         ,RANK () OVER (PARTITION BY department_id
                           ORDER BY salary) AS rank
         ,DENSE_RANK () OVER (PARTITION BY department_id
                                  ORDER BY salary) AS dense_rank
         ,ROW_NUMBER () OVER (PARTITION BY department_id
                                  ORDER BY salary
                                          ,first_name) AS row_number
     FROM employees
)
SELECT *
  FROM department_salaries
```

<a href="/assets/img/greatest-n-per-group_056.png"><img src="/assets/img/greatest-n-per-group_056.png" alt="greatest-n-per-group_056.png" target="_blank"></a>

### 5. (mini) BONUS

Przy okazji pisania tego posta, trafiłem na [artykuł z ciekawą tezą - "Nie używaj znaku nierówności **'>' (większe niż)** w programowaniu.](http://llewellynfalco.blogspot.com/2016/02/dont-use-greater-than-sign-in.html) 

**tl;dr**

Zapis nierówności wyłącznie z pomocą '**<**' jest lepszy, bo ogranicza liczbę tożsamych możliwości i graficznie oddaje sytuację na osi liczbowej. 

```
      x < 1 < 5 < x   
   ---x---1---5---x--->
```