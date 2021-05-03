---
layout: post
title: UTL_FILE.FGETATTR
subtitle: Był sobie plik...
thumbnail-img: /assets/img/utl_file_getattr_avatar.png
tags: [DB, PL/SQL]
---
W dzisiejszych czasach łatwo stać się ofiarą manipulacji 

...Szczególnie, jeśli developer Oracle używa wbudowanego pakietu UTL_FILE. 

Jeśli **Plik** już istnieje, to z UTL_FILE wykonasz na nim te akcje: 
- kopia,
- zmiana nazwy,
- wylistowanie atrybutów.

A jeśli **Plik** dopiero ma istnieć:
- ...to go stworzysz!

<a href="/assets/img/utl_file_050.jpg"><img src="/assets/img/utl_file_050.jpg" alt="utl_file_050.jpg" target="_blank"></a>

## Let's get to business

Stwórz directory i nadaj mu uprawnienia - z poziomu usera **SYS**:

```sql
CREATE OR REPLACE DIRECTORY UTL_FILE_DIR AS 'C:\UTL_FILE_DIR\';
GRANT READ, WRITE ON DIRECTORY UTL_FILE_DIR TO HR;
GRANT EXECUTE ON UTL_FILE TO HR;
```
Dalej działaj z poziomu usera **HR**

---
## 1. UTL_FILE.FGETATTR

FGETATTR służy do sprawdzenia, czy w danym directory dany plik istnieje. 

Jeśli tak, to możesz sprawdzić niektóre z jego właściwości.

Na początek prosty blok anonimowy. Krok po kroku, będziesz go ulepszać.

{% highlight sql linenos %}
DECLARE
    l_file_directory CONSTANT VARCHAR2(100) DEFAULT 'UTL_FILE_DIR';
    l_file_name      CONSTANT VARCHAR2(100) DEFAULT 'UTL_TEST.csv';
    l_file_exists    BOOLEAN;
    l_file_length    NUMBER;
    l_file_blocksize NUMBER;
    l_file_path      VARCHAR2(1000);
BEGIN

  utl_file.fgetattr(location    => l_file_directory,
                    filename    => l_file_name, 
                    fexists     => l_file_exists, 
                    file_length => l_file_length, 
                    block_size  => l_file_blocksize
                   );

  SELECT DIRECTORY_PATH 
    INTO l_file_path 
    FROM ALL_DIRECTORIES 
   WHERE DIRECTORY_NAME = l_file_directory;

  IF l_file_exists 
  THEN
    dbms_output.put_line('**START*********************************');
    dbms_output.put_line('');
    dbms_output.put_line('Directory:                  ' || l_file_directory);
    dbms_output.put_line('Path:                       ' || l_file_path);
    dbms_output.put_line('Name:                       ' || l_file_name);
    dbms_output.put_line('Size in bytes:              ' || l_file_length);
    dbms_output.put_line('System block size in bytes: ' || l_file_blocksize);
    dbms_output.put_line('');
    dbms_output.put_line('***********************************END**');
  ELSE
    dbms_output.put_line('**START*********************************');
    dbms_output.put_line('');
    dbms_output.put_line('File doesn''t exist or isn''t accessible with current priviliges.');
    dbms_output.put_line('');
    dbms_output.put_line('Directory:                  ' || l_file_directory);
    dbms_output.put_line('Path:                       ' || l_file_path);
    dbms_output.put_line('Name:                       ' || l_file_name);
    dbms_output.put_line('');
    dbms_output.put_line('***********************************END**');
  END IF;
END;
{% endhighlight %}

| Wiersz | Opis |
|:------:|---------------------------------------------------------------------------------------|
| 09-14 | Przypisz wartości parametrów pliku do wcześniej zdefiniowanych zmiennych lokalnych. |
| 16-19 | FGETATTR nie zwraca ścieżki pliku, więc pozyskaj ją w inny sposób. |

---

## 2. Blok anonimowy -> procedura
Porzuć hardkodowane dane na rzecz parametrów, podawanych przy wywoływaniu procedury.

{% highlight sql linenos %}
CREATE OR REPLACE PROCEDURE output_file_info (
  directory_name_i IN all_directories.directory_name%TYPE,
  file_name_i      IN VARCHAR2
  )
IS
  l_file_exists    BOOLEAN;
  l_file_length    NUMBER;
  l_file_blocksize NUMBER;
  l_directory_path all_directories.directory_path%TYPE;
BEGIN
  utl_file.fgetattr(
    location    => directory_name_i,
    filename    => file_name_i, 
    fexists     => l_file_exists, 
    file_length => l_file_length, 
    block_size  => l_file_blocksize
    );
                   
  SELECT DIRECTORY_PATH 
    INTO l_directory_path 
    FROM ALL_DIRECTORIES 
   WHERE DIRECTORY_NAME = directory_name_i;
   
  IF l_file_exists 
  THEN
    dbms_output.put_line('**START*********************************');
    dbms_output.put_line('');
    dbms_output.put_line('Directory:                  ' || directory_name_i);
    dbms_output.put_line('Path:                       ' || l_directory_path);
    dbms_output.put_line('Name:                       ' || file_name_i);
    dbms_output.put_line('Size in bytes:              ' || l_file_length);
    dbms_output.put_line('System block size in bytes: ' || l_file_blocksize);
    dbms_output.put_line('');
    dbms_output.put_line('***********************************END**');
  ELSE
    dbms_output.put_line('**START*********************************');
    dbms_output.put_line('');
    dbms_output.put_line('File doesn''t exist or isn''t accessible with current priviliges.');
    dbms_output.put_line('');
    dbms_output.put_line('Directory:                  ' || directory_name_i);
    dbms_output.put_line('Path:                       ' || l_directory_path);
    dbms_output.put_line('Name:                       ' || file_name_i);
    dbms_output.put_line('');
    dbms_output.put_line('***********************************END**');
  END IF;
END output_file_info;
{% endhighlight %}

| Wiersz | Opis |
|:------:|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1,8 | Zadeklaruj typ przez %TYPE% lub %ROWTYPE%. W ten sposób, nie będziesz musiał zmienić kodu procedury w razie, gdyby zmianie uległ typ np. danej kolumny. |

Zapisaną procedurę wywołaj poleceniem

```sql
EXEC OUTPUT_FILE_INFO('UTL_FILE_DIR', 'UTL_TEST.csv');
```

---
## 3. Procedura -> pakiet
Procedurę umieść w pakiecie. Jednocześnie, wynieś przypisywanie wartości dla zmiennej l_directory_path poza procedurę - do funkcji **directory_path**.

{% highlight sql linenos %}
CREATE OR REPLACE PACKAGE BODY FILE_API AS

  FUNCTION directory_path (
    directory_name_i IN all_directories.directory_name%TYPE
   ) 
    RETURN 
      all_directories.directory_path%TYPE
  IS
    l_directory_path all_directories.directory_path%TYPE;
  BEGIN
    SELECT directory_path
      INTO l_directory_path
      FROM ALL_DIRECTORIES  
     WHERE directory_name = directory_name_i;
    RETURN l_directory_path;
    
    EXCEPTION

  WHEN NO_DATA_FOUND THEN NULL;
  WHEN OTHERS THEN RAISE;
    
  END directory_path;

  PROCEDURE output_file_info (
    directory_name_i IN all_directories.directory_name%TYPE,
    file_name_i      IN VARCHAR2
    )
  IS
    l_file_exists    BOOLEAN;
    l_file_length    NUMBER;
    l_file_blocksize NUMBER;
    l_directory_path  all_directories.directory_path%TYPE;
  BEGIN
    utl_file.fgetattr(
      location    => directory_name_i,
      filename    => file_name_i, 
      fexists     => l_file_exists, 
      file_length => l_file_length, 
      block_size  => l_file_blocksize
      );

l_directory_path := directory_path(directory_name_i);

    IF l_file_exists 
    THEN                            
      dbms_output.put_line('**START*********************************');
      dbms_output.put_line('');
      dbms_output.put_line('Directory:                  ' || directory_name_i);
      dbms_output.put_line('Path:                       ' || l_directory_path);
      dbms_output.put_line('Name:                       ' || file_name_i);
      dbms_output.put_line('Size in bytes:              ' || l_file_length);
      dbms_output.put_line('System block size in bytes: ' || l_file_blocksize);
      dbms_output.put_line('');
      dbms_output.put_line('**END***********************************');
    ELSE
        RAISE program_error;
    END IF;
    EXCEPTION
  WHEN OTHERS THEN
    log_error('output_file_info: '  ||
              'directory_name_i = ' ||
              directory_name_i      ||
              ', file_name_i = '    ||
              file_name_i);

    RAISE;

  END output_file_info;

END FILE_API;
{% endhighlight %}

| Wiersz | Opis |
|:------:|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| 58-66 | Obsługuję wyjątki za pomocą wcześniej zdefiniowanej procedury - więcej szczegółów w [tym artykule](https://piotrwozniak.net/2019-03-01-obsluga-bledow/). |

Kod do uruchomienia procedury wewnątrz pakietu:

```sql
BEGIN
  FILE_API.output_file_info('UTL_FILE_DIR', 'UTL_TEST.csv');
END;
```