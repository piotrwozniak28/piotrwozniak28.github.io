---
layout: post
title: Normalizacja
subtitle: 1NF czy nie 1NF
image: /assets/img/normalizacja_avatar.jpg
tags: [DB]
---

## Definicja jest jednoznaczna

...kiedy w oparciu wyłącznie o jej treść, można bezbędnie określić, czy dany obiekt spełnia jej wymagania.
<span style="color:grey; font-size:10px;">(Przyjąłem śmiałe założenie, że wymagania są sprawdzalne - tj. urządzenia pomiarowe są dostępne, sprawne itp.)</span>

![](/img/normalizacja_050.jpg)

#### Weźmy przykład z geometrii euklidesowej.

[**Wielokąt**, wielobok, mat. płaska figura geometryczna będąca jedną (ograniczoną) z 2 części płaszczyzny, na które dzieli tę płaszczyznę łamana zwyczajna zamknięta (wraz z tą łamaną).
](https://encyklopedia.pwn.pl/haslo/wielokat;3995779.html).

Istnieją szczególne przypadki wielokąta. Wypisałem je poniżej w porządku, gdzie każdy kolejny zawiera w sobie wymagania poprzedniego.

![](/img/normalizacja_055.png)

Znane matematyczne catchphrase głosi:

{: .box-note}
Każdy kwadrat jest prostokątem, ale nie każdy prostokąt jest kwadratem

Załóżmy, że wymienione figury są w tej samej kolejności umieszczone na osi (w prawo rośnie).<br/>
Przy spełnionym warunku **X > Y**, możemy ukłuć ogólne stwierdzenie:

{: .box-note}
Każdy **X** jest **Y**, ale nie każdy **Y** jest **X**.

---

Niestety, a może stety, nie zawsze operujemy na tak ścisłych definicjach.

## Definicja jest niejednoznaczna

[Edgar Frank Codd](https://en.wikipedia.org/wiki/Edgar_F._Codd) - brytyjski informatyk, twórca [relacyjnego modelu baz danych](https://en.wikipedia.org/wiki/Relational_model) opisuje następujący konstrukt (który po czasie nazwiemy pierwszą postacią normalną) z użyciem pojęcia "atomic". 

{: .box-note}
In the relational model there is only one type of compound data: the relation.<br/>
The values in the domains on which each relation is defined are required to be atomic with respect to the DBMS.<br/>
A relational database is
a collection of relations of assorted degrees.<br/>
All of the query and manipulative operators are upon relations, and all of them generate relations as results.

<a href="https://www.amazon.com/Relational-Model-Database-Management-Version/dp/0201141922"><img src="/img/normalizacja_060.png" alt="normalizacja_060.png" target="_blank"></a>

{: .box-note}
**Mini-słowniczek**, co dany termin w praktyce oznacza:<br/><br/>
**Relation** to tabela<br/>
**Attribute** to kolumna   
**Domain** to **Type (typ)**<br/>

### DYGRESJA: A czym jest typ?

{: .box-note}
**Typ** ma nazwę - np. ```BOOLEAN```, ```INTEGER```, ```VARCHAR2```, ```CHAR```, ```NATURALN```.<br/>
**Typ** jest zbiorem wszystkich* możliwych wartości, jakie dana wartość(sic!) może przyjąć.<br/>Na przykład ```BOOLEAN``` to typ, który zawiera dwie wartości - ```TRUE``` i ```FALSE```.<br/><br/>
**\*** Dla ścisłości należałoby stwierdzić, że typ zawiera wszystkie wartości, jakie dany system jest w stanie reprezentować. Mając to na uwadze - zbiór liczb całkowitych co prawda jest nieskończony, ale typ ```INTEGER``` (```NUMBER(*,0)```) jest tylko kontenerem, który zawiera liczby podległe nadrzędnej zasadzie. Ogranicza go moc obliczeniowa skończonego komputera.<br/><br/>
Więcej na ten temat w rozdziale **2** książki:<br/>
[Type Inheritance and Relational Theory. Subtypes, Supertypes, and Substitutability - C. J. Date](https://helion.pl/ksiazki/type-inheritance-and-relational-theory-subtypes-supertypes-and-substitutability-c-j-date,e_099r.htm#format/e)
<br/><br/>
W praktyce wystarczy wiedzieć, że **Type/Domain** to nazwany zbiór wartości.

---

{: .box-error}
Użyte przez Codda kryterium "atomowości" wartości, zrodziło wiele interpretacji o tym, czym 1NF tak naprawdę jest.

### Dlaczego?

Weźmy fragment z polskiej Wikipedii

<a href="/img/normalizacja_065.png"><img src="/img/normalizacja_065.png" alt="normalizacja_065.png" target="_blank"></a>

Wspomniana elementarność (atomowość, niepodzielność) wartości, rozumiana bezwzględnie, występuje dla komputera przecież dopiero na poziomie bitu (0 lub 1).

#### W rozumieniu bardziej abstrakcyjnym - "atomowość" zagnieżdżona jest w kontekście.<br/>
Odnosi się do najmniejszej użytecznej (np. dla danej aplikacji) jednostki informacji.

{: .box-error}
Definicje 1NF, takie jak ta z wikipedii, nie zawierają tego **"to zależy".**

Przyjmując elementarność wartości atrybtów za **relatywną**, obie tabele poniżej mogą (ale nie muszą) być w 1NF.

| Name | Surname |
|------|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| Urszula |Woźniak|
| Marek |Woźniak|
| Zofia |Woźniak|
| Piotr |Woźniak|

---

| Name |
|------|
| Urszula Woźniak|
| Marek Woźniak|
| Zofia Woźniak|
| Piotr Woźniak|


Dopiero w momencie, gdy zasadne będzie żądanie używanie np. tylko imienia, to 1NF zostaje naruszona.

[Na zakończenie - cytat z bloga Fabiana Pascala, specjalisty w dziedzinie modelu relacyjnego](http://www.dbdebunk.com/2016/03/real-data-science-first-normal-form-in.html
)

{: .box-note}
So what is really atomic depends on how you plan to use your data.

Takie ujęcie dodaje ambiwalencji, ale zwiększa zrozumienie. Przynajmniej u mnie;)