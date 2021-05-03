---
layout: post
title: UTL_FILE.PUT_LINE
subtitle: Zapisz, bo zapomnisz
image: /img/utl_file_avatar.png
tags: [DB, PL/SQL]
---

[Neil deGrasse Tyson](https://www.youtube.com/watch?v=vGc4mg5pul4) to mój ulubiony astrofizyk

Jest na tyle dobrym mówcą, że kiedy opowiada o fizyce, to mam wrażenie, że ją rozumiem.

---

* W punkcie **1** zapiszemy do pliku jego treść jego [tweeta](https://twitter.com/neiltyson/status/150737457388863489)
* W punkcie **2** - tabelę z milionem wierszy 
* W punkcie **3** - tą samą tabelę, tyle że szybciej (porównanie w punkcie **5**)
* W punkcie **4** - krótko o buforze

## Let's get to business

### 1. Jak zapisać dane do pliku

{% highlight sql linenos %}
DECLARE
   l_directory_name   all_directories.directory_name%TYPE;
   l_open_mode        CHAR(1) DEFAULT 'w';
   l_file_name        VARCHAR2(100);
   l_file             utl_file.file_type;
   l_file_input       VARCHAR2(1000);
BEGIN
   l_directory_name   := 'UTL_FILE_DIR';
   l_file_name        := 'tyson_quote.txt';
   l_file_input       := 'According to the song, Rudolph’s nose is shiny, which means it reflects rather than emits light. Useless for navigating fog.';
   l_file             := utl_file.fopen(location => l_directory_name
                                       ,filename => l_file_name
                                       ,open_mode => l_open_mode);
   utl_file.put_line(l_file,l_file_input);
   utl_file.fclose(l_file);
END;
/
{% endhighlight %}

| Wiersz | Opis |
|:------:|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| 3 |W trybie **write** (```l_open_mode := 'w'```), zawartość pliku zostanie wyzerowana przy każdym kolejnym jego otwarciu.<br/>W trybie **append** (```l_open_mode := 'a'```), po 3 wykonaniach procedury miałbym 3 dodane kolejno pod sobą wiersze.|
| 15 |Mógłbym tu też użyć ```utl_file.fclose_all```.|

### 2. Jak zapisać dane do pliku - tyle że więcej

Najpierw więcej danych. Tworzę tabelę z milionem wierszy.

{% highlight sql linenos %}
DROP TABLE blurred_lines;
/
CREATE TABLE blurred_lines(
   id NUMBER(*,0)
);
/
   INSERT INTO blurred_lines
      SELECT level
        FROM dual CONNECT BY
         level <= 1000000;
/
{% endhighlight %}

A teraz właściwa procedura

{% highlight sql linenos %}
DECLARE
   l_directory_name    all_directories.directory_name%TYPE;
   l_open_mode         CHAR(1)DEFAULT 'w';
   l_file_name         VARCHAR2(100);
   l_file              utl_file.file_type;
   l_file_input        VARCHAR2(1000);
   l_timestamp_start   TIMESTAMP;
BEGIN
   l_timestamp_start   := systimestamp;
   l_directory_name    := 'UTL_FILE_DIR';
   l_file_name         := 'blurred_lines.txt';
   l_file              := utl_file.fopen(location  => l_directory_name
                                        ,filename  => l_file_name
                                        ,open_mode => l_open_mode);
   FOR i IN (SELECT *
               FROM blurred_lines) 
   LOOP 
      l_file_input := i.id;
      utl_file.put_line(l_file,l_file_input);
   END LOOP;
   utl_file.fclose(l_file);
   DBMS_OUTPUT.PUT_LINE(systimestamp - l_timestamp_start);
END;
/ 
{% endhighlight %}

| Wiersz | Opis |
|:------:|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| 15-20 |Wiersz po wierszu, zapisuję dane do pliku. Tryb **write** (```l_open_mode := 'w'```) może zostać. Plik nie zostanie nadpisany, dopóki go nie zamknę i ponownie otworzę.|
| 22 |Odczytuję, ile czasu zajęło wykonanie programu.<br/>Wynik i porównanie z kolejnym sposobem w punkcie **5**|

### 3. Jak zapisać dane do pliku - tyle że więcej i szybciej

Pętla powyżej wykonała 1kk iteracji - za każdym razem dodając nową linię w pliku ```utl_file.put_line(l_file,l_file_buffer);```<br/>A co, gdyby zapisywać więcej danych na raz?<br/><br/>Dane zostają te same. Kod przepisuję tak, aby pod zmienną zapisywaną przez w pliku przez ```put_line``` było więcej danych (w tym konkretnym przypadku - liczb).

{% highlight sql linenos %}
DECLARE
   l_directory_name    all_directories.directory_name%TYPE;
   l_open_mode         CHAR(1)DEFAULT 'w';
   l_file_name         VARCHAR2(100);
   l_file              utl_file.file_type;
   l_file_buffer       VARCHAR2(4000);
   l_buffer_size       PLS_INTEGER := 500;
   l_i                 PLS_INTEGER := 0;
   l_line_separator    VARCHAR2(10):= chr(10);
   l_timestamp_start   TIMESTAMP;
BEGIN
   l_timestamp_start   := systimestamp;
   l_directory_name    := 'UTL_FILE_DIR';
   l_file_name         := 'blurred_lines_without_input.txt';
   l_file              := utl_file.fopen(location  => l_directory_name
                                        ,filename  => l_file_name
                                        ,open_mode => l_open_mode);
   FOR i IN(SELECT *
              FROM blurred_lines)
   LOOP 
      l_i            := l_i + 1;
      IF l_i < l_buffer_size 
      THEN
         l_file_buffer := l_file_buffer || i.id
                          || l_line_separator
                          ;
      ELSE
         utl_file.put_line(l_file,l_file_buffer || i.id);
         l_i             := 0;
         l_file_buffer   := NULL;
      END IF;
   END LOOP;
   utl_file.put_line(l_file,l_file_buffer);
   utl_file.fclose(l_file);
   DBMS_OUTPUT.PUT_LINE(systimestamp - l_timestamp_start);
END;
/
{% endhighlight %}

| Wiersz | Opis |
|:------:|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| 14-19 |Zamiast wykonywać ```utl_file.put_line``` z każdą iteracją, tworzę bufor, w którym przechowam maksymalnie ```l_buffer_size``` wartości.|
| 7-8 |Wartość ```l_buffer_size``` musi być dobrana tak, żeby nie przekroczyć pojemności ```l_file_buffer``` (w tym przypadku 4kB).<br/>**Szerszy opis pod tabelą.**|
| 35 |Odczytuję, ile czasu zajęło wykonanie programu.<br/>Wynik i porównanie z poprzednim sposobem w punkcie **5**|

---

### 4. Rozmiar bufora

{: .box-note}
Każda wprowadzana liczba (```i.id```) zajmuje *n* bajtów, gdzie *n* to liczba cyfr, z której dana liczba się składa.<br/>W skrajnym przypadku, program będzie przekazywał do bufora (```l_file_buffer```) 499 liczb z 6 cyframi i 1 liczbę z 7 cyframi.<br/>Do każdej liczby dodajemy też 1 bajt za ```l_line_separator := chr(10)```.<br/>
Ostatecznie, aby mieć pewność że procedura się wykona, nasz bufor musi mieć minimum 499\*(6+1) + 1\*(7+1) = 3501 bajtów.

### Za mały

<a href="/img/utl_file_write_050.png"><img src="/img/utl_file_write_050.png" alt="utl_file_write_050.png" target="_blank"></a>

{: .box-error}
Przekazuję do bufora 2 wartości z tabeli ```blurred_lines``` (1000000, 999999) wraz z ```l_line_separator := chr(10)``` po każdej z nich.<br/>Razem potrzebuję 1\*(6+1) + 1\*(7+1) = **15** bajtów, podczas gdy ```l_buffer_size``` jest zdeklarowany na max **14** bajtów.<br/>
Baza zwraca błąd. 

### Idealny

<a href="/img/utl_file_write_055.png"><img src="/img/utl_file_write_055.png" alt="utl_file_write_055.png" target="_blank"></a>

{: .box-success}
Tym razem zdeklarowałem bufor idealnie na taką wartość (**15B**), jaka zostanie wprowadzona. Transakcja wykonana, plik poprawnie zapisany.

### 5. Bez bufora czy z buforem - porównanie

Bez owijania w bawełnę:

{: .box-error}
**Bez bufora**<br/><br/><a href="/img/utl_file_write_060.png"><img src="/img/utl_file_write_060.png" alt="utl_file_write_060.png" target="_blank">

{: .box-success}
**Z buforem**<br/><br/><a href="/img/utl_file_write_065.png"><img src="/img/utl_file_write_065.png" alt="utl_file_write_065.png" target="_blank">

Zaoszczędziliśmy **8 sekund!!!**  
Co w tym czasie można zrobić, pozostawiam wyobraźni developerów.

---

...A tak poważnie: skróciliśmy wykonywanie danego zadania o **ponad 57%**. W przypadku innego zestawu danych, można się spodziewać jeszcze lepszych rezultatów. 

Kolejny, podobny w wykonaniu sposób na tak poważny skok wydajnościowy to usunięcie non-query DML (```INSERT|UPDATE|DELETE```) z loopów.

## Źródła:
1. [https://docs.oracle.com/cd/B28359_01/appdev.1...](https://docs.oracle.com/cd/B28359_01/appdev.111/b28395/oci03typ.htm#i429972)
2. [https://docs.oracle.com/cd/B19306_01/appdev.1...](https://docs.oracle.com/cd/B19306_01/appdev.102/b14258/u_file.htm#BABGGEDF)
3. [https://connor-mcdonald.com/2018/06/18/juicin...](https://connor-mcdonald.com/2018/06/18/juicing-up-utl_file/)
4. [https://en.wikipedia.org/wiki/Kilo-](https://en.wikipedia.org/wiki/Kilo-)