---
layout: post
title: Obsługa błędów
subtitle: Każdy jest wyjątkowy...
image: /assets/img/obsluga_wyjatkow_050.jpg
tags: [DB, PL/SQL]
---

Jeśli uprawiałeś kiedyś boks to wiesz, że jak dobry byś nie był - nie unikniesz wszystkich ciosów

Zostaniesz uderzony. Im szybciej zaakceptujesz ten fakt, tym lepiej dla nosa i całej reszty.

Podobnie jest z programowaniem. Pojawią się błędy. Zobacz, co możesz z tym zrobić.


## Let's get to business

Wstęp złożony z antypraktyk byłby nie mniej edukacyjny niż dobry sparing.

...Jednak w tym artykule przejdę od razu do 2 optymalnych (!na mój stan wiedzy!) rozwiązań.

## 1. Custom Error Log

{% highlight sql linenos %}
CREATE TABLE error_log (
  ID               NUMBER (*,0),
  CREATED_ON       DATE DEFAULT SYSDATE,
  CREATED_BY       VARCHAR2(100),
  INFO             CLOB,
  CALLSTACK        CLOB,
  ERRORSTACK       CLOB,
  ERRORBACKTRACE   CLOB,
  CONSTRAINT error_log_pk PRIMARY KEY (id)
);
/

CREATE SEQUENCE ERROR_LOG_SEQ;
/

CREATE OR REPLACE TRIGGER ERROR_LOG_BIR BEFORE
  INSERT ON ERROR_LOG
  FOR EACH ROW
BEGIN
  :NEW.ID := ERROR_LOG_SEQ.NEXTVAL;
END;
/
{% endhighlight %}

| Wiersz | Opis |
|:------:|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| 20 |Możesz też użyć ```SELECT ERROR_LOG_SEQ.NEXTVAL INTO :NEW.ID FROM dual;``` We wczesnych release PL/SQL taki dummy SELECT był koniecznością.| 
| 13-22 |Korzystam z Oracle Database 11g Release 2, więc używam sekwencji + triggera. Dla Oracle DB 12c i wyższych, wystarczy zamiast tego w wierszu #2 (za definicją typu danych) dopisać: ``` GENERATED ALWAYS AS IDENTITY```|



{% highlight sql linenos %}
CREATE OR REPLACE PROCEDURE LOG_ERROR (
  INFO_I   IN       ERROR_LOG.INFO%TYPE
) IS
--
  PRAGMA AUTONOMOUS_TRANSACTION;
--
BEGIN
  INSERT INTO ERROR_LOG (
    CREATED_BY,
    INFO,
    CALLSTACK,
    ERRORSTACK,
    ERRORBACKTRACE
  ) VALUES (
    USER,
    INFO_I,
    DBMS_UTILITY.FORMAT_CALL_STACK,
    DBMS_UTILITY.FORMAT_ERROR_STACK,
    DBMS_UTILITY.FORMAT_ERROR_BACKTRACE
  );
--
  COMMIT;
END;
/
{% endhighlight %}


| Wiersz | Opis |
|:------:|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| 5 |Autonomous Transaction **(AT)** jest w pełni niezależna od transakcji wywołującej - tzn. zanim program (funkcja/procedura) w którym zachodzi AT zostanie zamknięty, wszystkie polecenia DML wewnątrz tego programu muszą zostać commit lub rollback. Dopiero wtedy kontrola zostanie przekazana do obiektu wywołującego. Przez tę cechę - a konkretnie - przez możliwość zacommitowania transakcji wewnątrz zrollbackowanej transakcji, **AT jest idealna do tworzenia logów**.|


Masz teraz niezależną procedurę logowania błędów.

{: .box-note}
**TEST:** Wykonaj insert + commit dla tabeli error_log bez jednoczesnego insert + commit w zewnętrznej transakcji.

{% highlight sql linenos %}
CREATE TABLE error_log_test (
  id NUMBER(*, 0)
);
/
CREATE OR REPLACE PROCEDURE log_error_test (
  number_i   IN         error_log_test.id%TYPE
) IS
BEGIN
  INSERT INTO error_log_test VALUES ( number_i );

  RAISE program_error;
EXCEPTION
  WHEN OTHERS THEN
    log_error('number_i = ' || number_i);
    RAISE;
END;
/
BEGIN
  log_error_test(round(dbms_random.value(0, 42), 0));
END;
/
{% endhighlight %}


| Wiersz | Opis |
|:------:|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| 11 |Celowo wywołuję program_error i przekazuję kontrolę do exception handlera.|
| 14 |Zapisuję dane o obecnym stanie transakcji do ERROR_LOG.|
| 15 |Z poziomu obsługi wyjątku, wykonuję RERAISE - spowrotem do otaczającego bloku.

{: .box-warning}
Dlaczego RERAISE wyjątku po zapisaniu wartości w logu?

### RAISE bez RERAISE

<a href="/img/obsluga_wyjatkow_055.png"><img src="/img/obsluga_wyjatkow_055.png" alt="obsluga_wyjatkow_055.png" target="_blank"></a>

{: .box-error}
Commit transakcji autonomicznej (zapis do ERROR_LOG) i commit wpisu do tabeli ERROR_LOG_TEST.

---

### RAISE z RERAISE

<a href="/img/obsluga_wyjatkow_060.png"><img src="/img/obsluga_wyjatkow_060.png" alt="obsluga_wyjatkow_060.png" target="_blank"></a>

{: .box-success}
Commit transakcji autonomicznej (zapis do ERROR_LOG) i rollback wpisu do tabeli ERROR_LOG_TEST.<br/><br/>
Czyli dokładnie to, o co chodziło.

## 2. Rozwiązanie Out of the Box

1. [https://www.uxora.com/oracle/sql/4-quest-error-manager](https://www.uxora.com/oracle/sql/4-quest-error-manager)
2. [https://github.com/oraopensource/logger](https://github.com/oraopensource/logger)

Oba gotowce są polecane przez autora 10 książek o PL/SQL, pracownika Oracle, [Stevena Feuersteina](https://blogs.oracle.com/author/steven-feuerstein).

## Źródła:

1. [https://livesql.oracle.com/apex/livesql/file/...](https://livesql.oracle.com/apex/livesql/file/content_CQHFTVKY3XROM2RLM3E13EC40.html)
2. [http://stevenfeuersteinonplsql.blogspot.com/2...](http://stevenfeuersteinonplsql.blogspot.com/2017/02/now-not-to-handle-exceptions.html)
3. [https://docs.oracle.com/cd/B28359_01/appdev.1...](https://docs.oracle.com/cd/B28359_01/appdev.111/b28370/errors.htm#LNPLS00702)
4. [http://www.dba-oracle.com/t_plsql_re_raise_ex...](http://www.dba-oracle.com/t_plsql_re_raise_exception.htm)