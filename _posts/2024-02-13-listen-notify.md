---
title: Механизм LISTEN/NOTIFY в Postgres. Его реализация и поддержка в Golang
date: 2024-02-13 21:54:00
categories: [Postgres, Golang]
tags: [программирование]
author: karine_ayrs
pin: true
---

БД Postgres поддерживает механизм [LISTEN](https://www.postgresql.org/docs/current/sql-listen.html) и [NOTIFY](https://www.postgresql.org/docs/current/sql-notify.html), позволяющий отправлять асинхронные уведомления через соединение с базой. Сегодня мы рассмотрим на примере, 
как можно его применить и имплементировать в Golang. 

## NOTIFY

Более подробно о команде можно почитать в [официальной документации](https://www.postgresql.org/docs/current/sql-notify.html), а здесь перечислим основные моменты.

