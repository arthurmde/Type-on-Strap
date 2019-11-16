---
layout: post
title: "The Bolsonaro's Bug - The end of Daylight Saving Time in Brazil may affect your system"
lang-ref: bolsonaro-bug
locale: en
feature-img: "assets/img/others/bolsonaro-face.png"
thumbnail: "assets/img/others/bolsonaro-face.png"
categories:
  - Hacking
  - Design
tags:
  - ruby
  - rails
  - emberjs
  - javascript
  - bug
  - design
  - frontend
  - backend
excerpt_separator: <!--more-->
---

Several software products and applications had bugs related to Brazil's
time zone recently due to [Bolsonaro's arbitrary decree](https://time.is/time_zone_news/no_dst_in_brazil_in_2019)
that ends the DST (Daylight Saving Time / Summer Time). Many
people are still using browsers operating with daylight saving time.
You may have noticed this if you use WhatsApp or Telegram in your browser.
At [Peerdustry](http://peerdustry.com), we also faced an interesting bug in
our platform which we affectionately named it **Bolsonaro's Bug** that deserves
to be discussed in more detail.

<!--more-->

# Context

Before explaining our specific bug, let me give you a simplified model of our
system. The Peerdustry platform is composed of a Rails API Backend
and a Frontend based on EmberJS.

One of the main flows implemented is related
to the quotation process in which a **Client** requests a quotation for
producing a mechanical part which will be evaluated and answered by
several **Manufacturers** that comprise our network. Such manufacturers are
chosen by the system's administrators through the creation of a task for each of
them to answer to the quotation. These tasks must be answered within before
a certain **deadline** established by the **Client**.

Recently, some manufacturers complained that their tasks were expiring
before the **deadline**. In addition, our administrators also reported that
they have seen strange dates and times in the system.
We immediately imagined that these bugs arose due to changes in daylight
saving time. We were right! However, this could be impacting multiple
modules of the system. Handling with date and time, time zones,
and different formats can be tricky, hindering the investigation to
actually find out where the problems were.

# A Simpler System

For simplification purposes, let's drive into the problem considering a
simpler system with only the most important components to the problem:
A college Web
system composed of a Rails Backend and an EmberJS Frontend. In this system,
a **Professor** can generate tasks for **Students** that
must be accomplished before a given **deadline**. Otherwise, they will expire.

The **Professor** informs the **deadline** date while creating the tasks for
**Students** by selecting a date through the
[Pikaday JS component](https://github.com/Pikaday/Pikaday).

![Pikaday JS Calendar]({{ "/assets/img/others/pikaday.png" }})

Before sending this data to the Backend, the Frontend will format it as
a timestamp attribute set at the end of the chosen date with
[MomentJS' endOf function](https://momentjs.com/docs/#/manipulating/end-of/)
which considers the browser time zone.
For instance, if the professor chose 15/11/2019 as the deadline, the formatted
data to be sent to the Backend will be `15/11/2019 at 11:59:59 pm` (or `23h59m59s`).
It is worth noticing that every timestamp attribute is formatted and stored
in **ISO-8601 UTC**. The **GMT** format is only used for UI presentation purposes.

Each student will be given a task that expires at the task's
**deadline**, which will become unavailable after this date.
To this end, whenever a task is created for a student, the
Backend will schedule an asynchronous job with
[Sidekiq](https://github.com/mperham/sidekiq)
to be run at the **deadline** to mark the task as expired if it has not been
accomplished yet.

Students can track their pending tasks through a page that presents
the list of tasks and their respective deadlines. Our deadlines are displayed
for end-users formatted as simple Brazilian dates (e.g.; 24/11/2019)
since it implicitly indicates that the task is available until the end of
the informed day, as illustrated below.

![Tasks List]({{ "/assets/img/others/bolsobug-task-table.png" }})

We also use the [MomentJS lib](https://momentjs.com/docs/#/displaying/format/)
to display such dates, which also considers the browser's time zone.

**So far, so good.**

# The Bug

After the Bolsonaro's decree, we made sure that our servers would not be using
DST wrongly so that the Backend's jobs would run in the proper time.
Given that the Backend is working with the right time zone (**UTC -3**) and
that the Frontend always provides the **deadlines** in UTC format, the Backend
will always schedule the jobs to expire the pending tasks in the **received
timestamp**.

However, the problem emerges when
either the **Professor** or the **Student** is using the platform in an
outdated browser that still operates considering Brazil's DST.
Some users of the system may have either their browsers with UTC -3 (Brazil's
default time zone)
or UTC -2 (former Brazil's DST time zone) which led us to some very odd
situations.

Let's imagine that a Professor needs to create a task with a deadline to
01/01/2020. We have the following situations:

## 1. When the Professor's browser is correctly operating with UTC -3

In this scenario, the deadline informed by the Professor is right
since we do not have DST anymore and the original Brazilian time zone is UTC-3.

If the Professor's input is 01/01/2020, the Frontend will send 02 Jan 2020
02:59:59 UTC to the Backend (01/01/2019 23:59:59 UTC-3).
As the Backend's time zone is also right, it will schedule the jobs to expire
tasks at the time the Professor expected.

#### 1.A. When the Student's browser is correctly operating with UTC -3

In this case, the student reading the message will not be confused,
since MomentJS lib is using the
proper time zone to display the date. In other words, the Student will see
the deadline date `01/01/2019`, which is correct.

#### 1.B. When the Student's browser is incorrectly operating with UTC -2 (DST)

In this case, the MomentJS lib will apply the UTC -2 time zone to the deadline
received form the Backend in UTC format, obtaining `02 Jan 2020 00:59:59 UTC -2`.
Since we display only the date and hide the time, the user would see 02/01/2020
(Brazil's date format)
instead of 01/01/2020 as the deadline for his task, leading him to a
misunderstanding of the correct date. While the Student thinks he will be able
to finish his task until 02/01/2020 (Brazil's date format), at this date the
task will not be available anymore.

## 2. When the Professor's browser is incorrectly operating with UTC -2 (DST)

In this scenario, we have a problem regardless of the Student's browsers since
the deadline provided to the Backend is incorrect.

If the Professor's input is 01/01/2020, the Frontend will send 02 Jan 2020
01:59:59 UTC to the Backend (01/01/2019 22:59:59 UTC -3).
This means that the deadline will expire 1 hour earlier than expected.

#### 2.A. When the Student's browser is correctly operating with UTC -3

In this case, the student has no confusion about the date, even though the
MomentJS lib is using a time zone different from the original to display the
date. Applying the UTC -3 to the original deadline will produce `01 Jan 2020
22:59:59 UTC -3`.

Thus, the Student would see 01/01/2020 as the deadline date, which is
correct. However, he will be expecting to have the deadline available until
23:59:59h, which will not occur.

You could argue that displaying the time along with the date to the Student in
the system would minimize the problem: 01/01/2020 22:59h. But the time is
likely to be ignored by him as he is used to having tasks available until
23:59h.

#### 2.B. When the Student's browser is incorrectly operating with UTC -2 (DST)

Although MomentJS lib will use the same time zone of the original deadline
to display the date, we still have some problems.

Applying the UTC -2 to the original deadline will produce 01 Jan 2020 23:59:59
--02:00. In this case, the Student would see 01/01/2020 as the deadline date,
which is correct. However, he will face the same problem of UTC -3 users
since he expects to have the deadline available until 23:59h, which will not
occur. Even worse, we cannot display the time to him as we did in the last
example since the displayed time would be wrong (displaying 23:59h even though
it will expire at 22:59h).

# How to fix?

There are some approaches to minimize the impact of the Bolsonaro's bug.
Most of them are quite hacky for me.

In general, if you make sure that your system stores and process date/time data
on UTC, your concern lies mostly on your presentation layer.

In the specific context of the Peerdustry's platform, both roles,
Manufacturers and Clients, hardly ever use the platform after 7 pm (the end
of their companies business hours), which means that the main problem
is displaying the wrong deadline date for the **Manufacturers** (Scenario 1.B).
In this sense, if we change the Frontend to always set the deadline to
`22:59:59 UTC -3` before sending it to the Backend, the Manufacturers will always
see the correct **date**. Even though the tasks will expire one hour earlier
than expected, almost nobody would be impacted by this.

**This approach could never be applied to a college system =D**

It is also possible to change the time zone used by
[MomentJS](https://momentjs.com/time zone/docs/) to mimic Brazil's new
time zone rules. However, this is the kind of approach that will give you
headaches when you have users on more than one time zone, besides jeopardizing
the internationalization of your system.

In my opinion, the most appropriate solution to bugs similar to our
Bolsonaro's bug is:
* Display the time to along with the deadline date
* Inform users when their browsers are operating with outdated time zone
information, warning them about possible bugs and requesting them to upgrade
their browsers.


**What about you? Have you faced any odd bug after Bolsonaro's decree?**

<hr>

<small>The cover image is from Fábio Rodrigues Pozzebom/Agência Brasil [CC BY 2.0](https://creativecommons.org/licenses/by/2.0) via Wikimedia Commons</small>

<span>**#ELENAO - Let's get moving on! ;)**</span>


