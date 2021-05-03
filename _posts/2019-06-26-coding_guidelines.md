---
layout: post
title: SQL and PL/SQL Coding Standards
subtitle: Trzeba mieć jakieś standardy
thumbnail-img: /assets/img/guidelines_050.jpg
tags: [SQL, PL/SQL, Workflow]
published: true
---

Trudno znaleźć mi jakieś argumenty, dlaczego możnaby stwierdzić inaczej.

Krótko mówiąc - dzięki standardom łatwiej działać z kodem. 
Takie stwierdzenie ma multum implikacji. Mniejsza podatność na błędy i szybsze czytanie kodu to tylko część z nich.
Poza tym, jeśli coś robić, to czemu tego nie robić dobrze? 

...Tym bardziej, że w tym przypadku nakłady pracy są pomijalne w stosunku do korzyści.

Najbardziej kompletne, najszerzej opisane i po prostu najlepsze standary dla SQL i PL/SQL to moim zdaniem:

### Trivadis PL/SQL & SQL Coding Guidelines

{: .box-success}
**pdf:** [https://trivadis.github.io/plsql-and-sql-coding-guideline..](https://trivadis.github.io/plsql-and-sql-coding-guidelines/master/9-appendix/PLSQL-and-SQL-Coding-Guidelines.pdf)<br/>**repo:** [https://github.com/Trivadis/plsql-and-sql-coding-guidelines](https://github.com/Trivadis/plsql-and-sql-coding-guidelines)

#### Dokument zawiera wytyczne co do nazewnictwa obiektów...
Jeśli ktoś - tak jak ja - ceni artykuły Stevena Feuersteina, to na pewno skojarzy pewne podobieństwa:) 
Na marginesie - sam Steven poleca guidelines od Trivadis.

#### Trivadis

{: .box-note}
<a href="/assets/img/guidelines_061.png"><img src="/assets/img/guidelines_061.png" alt="guidelines_061.png" target="_blank"></a>
<a href="/assets/img/guidelines_062.png"><img src="/assets/img/guidelines_062.png" alt="guidelines_062.png" target="_blank"></a>

#### Steven

{: .box-note}
<a href="/assets/img/guidelines_065.png"><img src="{{ 'assets/img/guidelines_065.png' | relative_url }}" alt="guidelines_065.png" target="_blank"></a><br/>
[Pdf z guidelines od Stevena](https://community.oracle.com/docs/DOC-1007838)

---

#### ...Są także dobre praktyki co do pisania kodu.
Przykłady są prezentowane w konwencji źle/dobrze - od razu nasuwa mi się skojarzenie ze świetnym poradnikiem do Material Design od Google

#### Trivadis

{: .box-note}
<a href="/assets/img/guidelines_063.png"><img src="/assets/img/guidelines_063.png" alt="guidelines_063.png" target="_blank"></a>

#### Google

{: .box-note}
<a href="/assets/img/guidelines_064.png"><img src="/assets/img/guidelines_064.png" alt="guidelines_064.png" target="_blank"></a>

Standardy są udostępniane na licencji Apache License 2.0.

---

<a href="https://xkcd.com/1513/"><img src="https://imgs.xkcd.com/comics/code_quality.png" alt="https://imgs.xkcd.com/comics/code_quality.png" target="https://xkcd.com/1513/">