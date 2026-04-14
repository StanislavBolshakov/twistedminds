---
title: FH_DATE_PAST_20XX The date is grossly in the future.
author: ["Stanislav"]
date: 2010-01-25T12:58:27+00:00
url: /2010/01/spamassasin-2010-problem/
categories:
  - Tech
tags:
  - mdaemon
  - spamassasin
  - баг
  - решаем проблемы
---

Сегодня получил жалобу на то, что некоторые письма не доходят и посетив Spam Trap MDaemon’a обнаружил кучу миловидных писем, которые были забракованы по причине лишних 3.2 баллов из-за:

> FH_DATE_PAST_20XX The date is grossly in the future.

оказывается я прошляпил в бюллетенях баг SpamAssasin’a, из-за которого все письма, отправленные после 2010 считались пришедшими из будущего :)

Лечится изменением регэкспа с

> FH_DATE_PAST_20XX
>
> header   FH_DATE_PAST_20XX Date =~ /20[**1**-9][0-9]/ [if-unset: 2006]

на

> FH_DATE_PAST_20XX
>
> header   FH_DATE_PAST_20XX Date =~ /20[**2**-9][0-9]/ [if-unset: 2006]

и перезапуском демона.
