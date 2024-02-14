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
func (c *Client) waitForNotification(ctx context.Context) {
  for {
          select {
          case notification := <-c.listener.Notify:
              fmt.Printf("got notification from DB, channel: %s", notification.Channel)
              // отправляем уведомление, полученное из БД-канала в соответствующий go-канал
              c.channels[notification.Channel] <- notification
          case <-time.After(90 * time.Second):
              go c.listener.Ping()
              fmt.Println("received no notifications from DB for 90 seconds, checking for new messages")
          case <-ctx.Done():
              // отправляем ошибку во все каналы ошибок для корректного завершения приложения (чтобы горутина не продолжала висеть в бэкграунде)
              for _, errc := range c.errc {
                  errc <- ctx.Err()
              }
              return
          }
      }
}
```
Здесь, уведомления приходят от всех каналов, имеющихся в БД. Определить канал пришедшего уведомления можно с 
помощью поля `Channel` структуры `*pq.Notification`, получаемую из go-канала `c.listener.Notify`<br>

🎯 Наша цель - иметь возможность получать уведомления от нескольких каналов и уметь маршрутизировать их. 
  
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
- представим, что у нас есть репозиторий `UsersRepository` для работы с БД, в котором мы хотим обрабатывать полученные уведомления
```go
func (p *pgUsers) ListenNotifications(ctx context.Context) error {
	log.Println("listening for notifications in repo")

  // регистрируем канал p.channel
  // получаем go-канал, из которого будем получать уведомления для конкретного p.channel
  // получаем go-канал для обработки ошибок из p.channel
	notificationc, errc, err := p.ListenChannel(ctx, p.channel)
	if err != nil {
		return fmt.Errorf("can't connect to channel %s, err: %w", p.channel, err)
	}

	for {
		select {
		case res := <-notificationc:
			// do smth with notification
			fmt.Printf("notification in repo %s\n", res.Extra)
		case err := <-errc:
			fmt.Println("got error")
			return err
		}
	}
}
```
Теперь сконфигурируем  `main.go`
```go
package main

import (
	"ListenNotifyArticle/adapters/postgres"
	"ListenNotifyArticle/pkg/pg"
	"context"
	"fmt"
	"golang.org/x/sync/errgroup"
	"log"
	"os"
	"os/signal"
	"syscall"
)

func main() {
	dsn := "user=postgres password=postgres host=localhost port=5432 dbname=postgres connect_timeout=3 sslmode=disable"

	client := pg.NewClient()
	err := client.Open(dsn)
	if err != nil {
		log.Fatalf("can't establish connection with postgres, err: %v", err)
	}

	err = client.AddListener(dsn)
	if err != nil {
		log.Fatalf("can't establish connection for listener, err: %v", err)
	}

	usersRepo := postgres.NewUsersRepo(client, "users")

	ctx := context.Background()
	group, ctx := errgroup.WithContext(ctx)
	group.Go(func() error {
		sig := make(chan os.Signal, 1)
		signal.Notify(sig, syscall.SIGINT, syscall.SIGTERM)
		select {
		case <-ctx.Done():
			return ctx.Err()
		case s := <-sig:
			return fmt.Errorf("signal received: %s", s)
		}
	})

  // для того, чтобы получать уведомления от другого канала, достаточно написать метод по аналогии с usersRepo.ListenNotifications(ctx) и запустить его в такой же горутине
	group.Go(func() error {
		go func() {
			<-ctx.Done()
			log.Println("app was interrupted")
			_, cancel := context.WithCancel(ctx)
			defer cancel()
		}()

		err := usersRepo.ListenNotifications(ctx)
		if err != nil {
			return fmt.Errorf("listening to channel error: %w", err)
		}
		return nil
	})

	if err = group.Wait(); err != nil {
		log.Printf("app stopped with err: %v", err)
	}
}

```
Протеструем получение уведомлений запустив приложение и выполнив:
```sql
insert into users(id, first_name, last_name) values ('1', 'John', 'Doe');
```
Посмотрим, какая информация отобразилась в логах:
```
connected to postgres
2024/02/14 22:56:54 listening for notifications in repo
got notification from DB, channel: usersnotification in repo {"timestamp" : "2024-02-14T19:57:46.67918+00:00", "action" : "insert", "db_schema" : "public", "table" : "users", "record" : {"id":"1","first_name":"John","last_name":"Doe"}, "old" : null}
```

🏆 Ура! Видим, что пришло уведомление о том, что в таблицу `users` была добавлена запись `{"id":"1","first_name":"John","last_name":"Doe"}`. Остальные операции (`update`, `delete`) предлагается 
протестировать читателю! <br>
Полностью рабочий проект доступен [здесь](https://github.com/KarineAyrs/listen-notify). <br> 
Спасибо за внимание!
