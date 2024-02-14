---
title: Механизм LISTEN/NOTIFY в Postgres. Его реализация и поддержка в Golang
date: 2024-02-13 21:54:00 +0300
categories: [Программирование]
tags: [golang, postgres]
author: karine_ayrs
pin: true
toc: false
image: assets/img/pg-go-handshake.png
---

БД Postgres поддерживает механизм [LISTEN](https://www.postgresql.org/docs/current/sql-listen.html) и [NOTIFY](https://www.postgresql.org/docs/current/sql-notify.html), позволяющий отправлять асинхронные уведомления через соединение с базой. Сегодня мы рассмотрим на примере, 
как можно его применить и имплементировать в Golang. 

#### Содержание
- [NOTIFY](#notify)
- [LISTEN](#listen)
- [LISTEN/NOTIFY  на стороне БД](#работа-на-стороне-бд)

## NOTIFY

  Более подробно о команде можно почитать в [официальной документации](https://www.postgresql.org/docs/current/sql-notify.html), а здесь перечислим основные моменты.

  `NOTIFY channel [, payload]`

  - отправляет событие (уведомление) с опциональным `payload` (строка) каждому клиенту, который ранее выполнил `LISTEN channel`
  - нотификации видны всем пользователям
  - информация, отправляемая клиенту включает в себя название канала нотификаций `channel`, PID сервера, полезную нагрузку `payload`
  - обычно, название канала совпадает с названием таблицы, при модификации которой нужно послать уведомление
  - полезная практика - указать `NOTIFY` в триггере - тогда уведомление происходит автоматически при изменении таблицы
  - если `NOTIFY` исполняется внутри транзакции, то нотификация не будет доставлена до тех пор, пока транзакция не завершится
  - если в один и тот же канал приходят идентичные уведомления внутри одной транзакции, только одна сущность нотификации будет доставлена слушателям
  - обычно, клиент, который выполняет `NOTIFY`, слушает тот же канал (куда пушит нотификации) сам. Можно этого избежать отслеживая PID (если PID сервера, отправляющего сообщения, который указан в уведомлении, совпадает с  PID’ом сессии (доступен в libpq). В таком случае нотификацию можно игнорировать)

> Нужно следить за переполнением внутренней очереди сообщений - ее размер 8GB (можно настроить). Если она заполнится, то `NOTIFY` не сможет запушить сообщения. При достижении 4GB придет предупреждение
{: .prompt-warning }

## LISTEN 
Более подробно о команде можно почитать в [официальной документации](https://www.postgresql.org/docs/current/sql-listen.html), а здесь перечислим основные моменты.

`LISTEN channel`
- `LISTEN` регистрирует текущую сессию как слушателя канала уведомлений `channel` . Если сессия уже зарегистрирована, то ничего не происходит
- как только срабатывает команда `NOTIFY channel` в текущей сессии или другой, подключенной к одной и той же базе данных, все сессии слушающие на данный момент этот канал уведомлений уведомляются и каждая, в свою очередь уведомит подключенное клиентское приложение
- сессию можно разрегистрировать с помощью команды `UNLISTEN` - регистрации автоматически очищаются когда сессия заканчивается
- метод в клиентском приложении для поддержки механизма событий (уведомлений) - зависит от драйвера

## Работа на стороне БД 
Создадим таблицу `users` 

```sql
create table users(
  id text not null,
  first_name text not null,
  last_name text not null,
  unique(id)
);
```
🌟 Допустим, мы хотим отправлять нотификации при любых изменениях данных в таблице (операции `INSERT`, `UPDATE`, `DELETE`). <br>
⚡ Создадим триггер на изменение, добавление, удаление данных и для каждой строки будем отправлять нотификацию в
канал `users` c помощью функции `pg_notify()`
```sql
CREATE TRIGGER users_notify
    AFTER INSERT OR UPDATE OR DELETE
    ON users
    FOR EACH ROW
EXECUTE PROCEDURE notify_trigger();

CREATE OR REPLACE FUNCTION notify_trigger() RETURNS trigger AS
$trigger$
DECLARE
    rec     users;
    dat     users;
    payload TEXT;
BEGIN
    CASE TG_OP
        WHEN 'UPDATE' THEN rec := NEW;
                           dat := OLD;
        WHEN 'INSERT' THEN rec := NEW;
        WHEN 'DELETE' THEN rec := OLD;
        ELSE RAISE EXCEPTION 'Unknown TG_OP: "%". Should not occur!', TG_OP;
        END CASE;
    payload := json_build_object('timestamp', CURRENT_TIMESTAMP, 'action', LOWER(TG_OP), 'db_schema', TG_TABLE_SCHEMA,
                                 'table', TG_TABLE_NAME, 'record', row_to_json(rec), 'old', row_to_json(dat));

    PERFORM pg_notify('users', payload);
    RETURN rec;
END;
$trigger$ LANGUAGE 'plpgsql';

```

---
title: Механизм LISTEN/NOTIFY в Postgres. Его реализация и поддержка в Golang
date: 2024-02-13 21:54:00 +0300
categories: [Программирование]
tags: [golang, postgres]
author: karine_ayrs
pin: true
toc: false
image: assets/img/pg-go-handshake.png
---

БД Postgres поддерживает механизм [LISTEN](https://www.postgresql.org/docs/current/sql-listen.html) и [NOTIFY](https://www.postgresql.org/docs/current/sql-notify.html), позволяющий отправлять асинхронные уведомления через соединение с базой. Сегодня мы рассмотрим на примере, 
как можно его применить и имплементировать в Golang. 

#### Содержание
- [NOTIFY](#notify)
- [LISTEN](#listen)
- [LISTEN/NOTIFY  на стороне БД](#работа-на-стороне-бд)
- [LISTEN/NOTIFY  на стороне Golang](#работа-на-стороне-golang)

## NOTIFY

    Более подробно о команде можно почитать в [официальной документации](https://www.postgresql.org/docs/current/sql-notify.html), а здесь перечислим основные моменты.

    `NOTIFY channel [, payload]`

    - отправляет событие (уведомление) с опциональным `payload` (строка) каждому клиенту, который ранее выполнил `LISTEN channel`
    - нотификации видны всем пользователям
    - информация, отправляемая клиенту включает в себя название канала нотификаций `channel`, PID сервера, полезную нагрузку `payload`
    - обычно, название канала совпадает с названием таблицы, при модификации которой нужно послать уведомление
    - полезная практика - указать `NOTIFY` в триггере - тогда уведомление происходит автоматически при изменении таблицы
    - если `NOTIFY` исполняется внутри транзакции, то нотификация не будет доставлена до тех пор, пока транзакция не завершится
    - если в один и тот же канал приходят идентичные уведомления внутри одной транзакции, только одна сущность нотификации будет доставлена слушателям
    - обычно, клиент, который выполняет `NOTIFY`, слушает тот же канал (куда пушит нотификации) сам. Можно этого избежать отслеживая PID (если PID сервера, отправляющего сообщения, который указан в уведомлении, совпадает с  PID’ом сессии (доступен в libpq). В таком случае нотификацию можно игнорировать)

> Нужно следить за переполнением внутренней очереди сообщений - ее размер 8GB (можно настроить). Если она заполнится, то `NOTIFY` не сможет запушить сообщения. При достижении 4GB придет предупреждение
{: .prompt-warning }

## LISTEN 
Более подробно о команде можно почитать в [официальной документации](https://www.postgresql.org/docs/current/sql-listen.html), а здесь перечислим основные моменты.

`LISTEN channel`
- `LISTEN` регистрирует текущую сессию как слушателя канала уведомлений `channel` . Если сессия уже зарегистрирована, то ничего не происходит
- как только срабатывает команда `NOTIFY channel` в текущей сессии или другой, подключенной к одной и той же базе данных, все сессии слушающие на данный момент этот канал уведомлений уведомляются и каждая, в свою очередь уведомит подключенное клиентское приложение
- сессию можно разрегистрировать с помощью команды `UNLISTEN` - регистрации автоматически очищаются когда сессия заканчивается
- метод в клиентском приложении для поддержки механизма событий (уведомлений) - зависит от драйвера

## Работа на стороне БД 
Создадим таблицу `users` 

```sql
create table users(
    id text not null,
    first_name text not null,
    last_name text not null,
    unique(id)
);
```
🌟 Допустим, мы хотим отправлять нотификации при любых изменениях данных в таблице (операции `INSERT`, `UPDATE`, `DELETE`). <br>
⚡ Создадим триггер на изменение, добавление, удаление данных и для каждой строки будем отправлять нотификацию в
канал `users` c помощью функции `pg_notify()`
```sql
CREATE TRIGGER users_notify
        AFTER INSERT OR UPDATE OR DELETE
        ON users
        FOR EACH ROW
EXECUTE PROCEDURE notify_trigger();

CREATE OR REPLACE FUNCTION notify_trigger() RETURNS trigger AS
$trigger$
DECLARE
        rec     users;
        dat     users;
        payload TEXT;
BEGIN
        CASE TG_OP
                WHEN 'UPDATE' THEN rec := NEW;
                                                      dat := OLD;
                WHEN 'INSERT' THEN rec := NEW;
                WHEN 'DELETE' THEN rec := OLD;
                ELSE RAISE EXCEPTION 'Unknown TG_OP: "%". Should not occur!', TG_OP;
                END CASE;
        payload := json_build_object('timestamp', CURRENT_TIMESTAMP, 'action', LOWER(TG_OP), 'db_schema', TG_TABLE_SCHEMA,
                                                                  'table', TG_TABLE_NAME, 'record', row_to_json(rec), 'old', row_to_json(dat));

        PERFORM pg_notify('users', payload);
        RETURN rec;
END;
$trigger$ LANGUAGE 'plpgsql';

```

## Работа на стороне Golang
Рассмотрим код получения уведомлений из postgres
```go
for {
        select {
        case notification := <-c.listener.Notify:
            fmt.Printf("got notification from DB, channel: %s", notification.Channel)
            c.channels[notification.Channel] <- notification
        case <-time.After(90 * time.Second):
            go c.listener.Ping()
            fmt.Println("received no notifications from DB for 90 seconds, checking for new messages")
        case <-ctx.Done():
            for _, errc := range c.errc {
                errc <- ctx.Err()
            }
            return
        }
    }
```
Здесь, уведомления приходят от всех каналов, имеющихся в БД. Определить канал пришедшего уведомления можно с 
помощью поля `Channel` структуры `*pq.Notification`, получаемую из go-канала `c.listener.Notify`

Наша цель - иметь возможность получать уведомления от нескольких каналов и уметь маршрутизировать их. 
  
Для этого в клиенте для БД 
создадим `map`, который будет маппить название канала БД на `chan *pq.Notification`, а в том месте программы, где должна находиться обработка 
сообщений из канала БД, будем слушать `chan *pq.Notification`. Аналогично сделаем и `map` для обработки ошибок и корректного завершения программы.
Получение уведомлений будет запускаться в фоновом режиме с помощью горутины. 

```go
type Client struct {
    DB         *sql.DB
    listener   *pq.Listener
    // маппим канал из БД на go-канал для сообщения (уведомления)
    channels   map[string]chan *pq.Notification
    // маппим канал из БД на go-канал для обработки ошибок 
    errc       map[string]chan error
    waitCalled bool
}
```

- напишем метод, который создает подключение к БД для listener'а
```go
func (c *Client) AddListener(dataSourceName string) error {
    c.listener = pq.NewListener(dataSourceName, time.Second*10, time.Minute, func(event pq.ListenerEventType, err error) {
        if err != nil {
            fmt.Printf("listener error: %s", err.Error())
        }
    })
    return nil
}
```
- теперь, нужен метод, который будет "регистрировать" определенный канал БД и запускать горутину для получения уведомлений. Этот метод будет возвращать `chan *pq.Notification, chan error, error`

```go
func (c *Client) ListenChannel(ctx context.Context, channelName string) (chan *pq.Notification, chan error, error) {
    channel, ok := c.channels[channelName]
    if !ok {
        // регистрируем БД-канал
        err := c.listener.Listen(channelName)
        if err != nil {
            if !errors.Is(err, pq.ErrChannelAlreadyOpen) {
                return nil, nil, fmt.Errorf("can't listen to channel %s, err: %w", channelName, err)
            }
        }

        if err = c.listener.Ping(); err != nil {
            return nil, nil, fmt.Errorf("listener ping failed, %w", err)
        }
        // создаем go-канал, откуда будем получать уведомления для определенного channelName
        c.channels[channelName] = make(chan *pq.Notification)
        channel = c.channels[channelName]
    }

    errc, ok := c.errc[channelName]
    if !ok {
        // создаем go-канал для обработки ошибок от канала channelName
        c.errc[channelName] = make(chan error)
        errc = c.errc[channelName]
    }
    // если горутина получения уведомлений не запущена, запускаем ее
    if !c.waitCalled {
        go c.waitForNotification(ctx)
        c.waitCalled = true
    }
    return channel, errc, nil
}
```


