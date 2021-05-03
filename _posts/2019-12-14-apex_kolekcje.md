---
layout: post
title: Kolekcje APEX
subtitle: Select2 używając APEX Collections
thumbnail-img: /assets/img/apex_kolekcje_050.jpg
tags: [DB, SQL, PL/SQL, APEX]
---

Select2 to jeden z lepszych pluginów dla APEXa. 

Krótko mówiąc - usprawnia i uładnia wybieranie wartości. Za opisem z github:

{: .box-note}
The Select2 APEX plugin is based on Select2 – an open-source jQuery plugin that greatly improves the functionality and user-friendliness of regular select lists. In Oracle Application Express, the Select2 plugin can serve as a replacement for these four standard item types:<br/><br/>- Select List<br/>- Shuttle<br/>- Text Field with autocomplete<br/>- List Manager

Przedstawię krótko, jak można zaimplementować go w swojej aplikacji. 

Przyjmijmy, że chcemy mieć możliwość zapraszania osób na spotkania.<br/>Lista dostępnych do wyboru imion+nazwisk powinna się zawężać w miarę podawania kolejnych znaków (podobnie jak np. przy tworzeniu nowej grupy na Facebooku lub nowym wydarzeniu w Google Calendar)

<a href="/assets/img/apex_kolekcje_055.png"><img src="/assets/img/apex_kolekcje_055.png" alt="apex_kolekcje_055.png" target="_blank"></a>

W naszym przykładzie, Select2 ułatwi zebranie ID pracowników do danego Page Item.

<a href="/assets/img/apex_kolekcje_060.png"><img src="/assets/img/apex_kolekcje_060.png" alt="apex_kolekcje_060.png" target="_blank"></a>

Kiedy będziemy już mieli ID w Itemie, to wystarczy je tylko przekazać do obiektu (w naszym przypadku będzie to APEX Collection, jednak mogłaby to być zwyczajna tabela) i wyświetlić - np. w Interactive Report lub Interactive Grid.

Żywy przykład powie więcej niż 1000 słów. Tutaj link do apki z opisywaną LOV i kodem:<br/>
[https://apex.oracle.com/pls/apex/f?p=280000:120](https://apex.oracle.com/pls/apex/f?p=280000:120)

{: .box-warning}
Dlaczego używam ```APEX Collection``` zamiast którejś z kolekcji PL/SQL (```Associative Array```, ```Varray```, ```Nested Table```)?

Aplikację udostępniam publicznie, więc zależy mi, aby dokonane zmiany były **tymczasowe pomiędzy różnymi sesjami APEX**.<br/>
Z drugiej strony, zmiany powinny być **trwałe pomiędzy transakcjami bazodanowymi**, które odbędą się podczas korzystania w danej sesji. Czyli np. jeśli użytkownik odświeży daną stronę lub "przeklika się" po różnych innych przykładach, to zmiany które wprowadził na początku, powinny nadal obowiązywać.<br/>Klasyczne ```temporary tables``` nie nadawałyby się do tego zadania, dlatego Oracle wprowadziło ```APEX Collections```




## Źródła:
1. [https://github.com/nbuytaert1/apex-select2](https://github.com/nbuytaert1/apex-select2)
2. [https://docs.oracle.com/database/121/AEAPI/ap...](https://docs.oracle.com/database/121/AEAPI/apex_collection.htm#AEAPI531)