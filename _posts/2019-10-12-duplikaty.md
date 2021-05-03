---
layout: post
title: Duplikaty
subtitle: ...i ich usuwanie
image: /assets/img/duplikaty_050.jpg
tags: [DB, SQL]
---

Stwórzmy coś sztampowego

{% highlight sql linenos %}
DROP TABLE names;
CREATE TABLE names(id NUMBER(*,0) GENERATED ALWAYS AS IDENTITY
                  ,name VARCHAR2(10));
/
INSERT INTO names (name) VALUES ('Tomasz');
INSERT INTO names (name) VALUES ('Piotr');
INSERT INTO names (name) VALUES ('Piotr');
INSERT INTO names (name) VALUES ('Marek');
INSERT INTO names (name) VALUES ('Marek');
INSERT INTO names (name) VALUES ('Marek');
INSERT INTO names (name) VALUES (NULL);
INSERT INTO names (name) VALUES (NULL);
INSERT INTO names (name) VALUES (NULL);
INSERT INTO names (name) VALUES (NULL);
/
{% endhighlight %}

Przyjmijmy, że mamy usunąć wszystkie wartości, które są zduplikowane (a właściwie - zmultiplikowane).
W pierwszym odruchu, może nam przyjść na myśl takie query

{% highlight sql linenos %}
SELECT name
  FROM names
 GROUP BY name
HAVING COUNT(*) > 1;
/
{% endhighlight %}

Dostaniemy wartości, które występują ```> 1``` raz, niemniej nie wiemy ILE razy każda z wartości jest powielona.
Jednokrotne usunięcie wierszy z pomocą tego query, nie da nam żadnej pewności co do do unikalności - w tym przypadku - imion.

{% highlight sql linenos %}
DELETE FROM names 
 WHERE rowid IN (
   SELECT MAX(rowid)
     FROM names
    GROUP BY name
   HAVING COUNT(*) > 1);
/
SELECT *
  FROM names;
/
{% endhighlight %}

Piotr jest tylko jeden (pun intended), ale nadal mamy 2 Marków i 3 NULLE.

Możemy albo uruchamiać powyższe query do skutku, albo napisać coś lepszego. 
Proponuję 2 rozwiązania:

### 1. Analityczne query

{% highlight sql linenos %}
DELETE FROM names 
 WHERE ROWID IN (
   SELECT rowid 
     FROM (
      SELECT rowid
            ,ROW_NUMBER() OVER(PARTITION BY name ORDER BY rowid) AS ronu
        FROM names)
    WHERE ronu > 1);
/
{% endhighlight %}

### 2. Correlated subquery

{% highlight sql linenos %}
DELETE FROM names nam1
 WHERE rowid > (SELECT MIN(rowid)
                  FROM names nam2
                 WHERE nam1.name = nam2.name
                    OR (1=1
                        AND nam1.name IS NULL 
                        AND nam2.name IS NULL));
/
{% endhighlight %}

Bywa i tak, że przed usuwaniem warto napisać jeszcze szybkie query dla lepszego zobrazowania sytuacji

{% highlight sql linenos %}
SELECT id
      ,name
      ,rowid
      ,1 AS is_duplicate
  FROM names nam1
 WHERE rowid > (SELECT MIN(rowid)
                  FROM names nam2
                 WHERE nam1.name = nam2.name
                    OR (1=1
                        AND nam1.name IS NULL 
                        AND nam2.name IS NULL))
UNION ALL
SELECT id
      ,name
      ,rowid
      ,0
  FROM names nam1
 WHERE rowid = (SELECT MIN(rowid)
                  FROM names nam2
                 WHERE nam1.name = nam2.name
                    OR (1=1 
                        AND nam1.name IS NULL 
                        AND nam2.name IS NULL))
 ORDER BY rowid;
 /
{% endhighlight %}