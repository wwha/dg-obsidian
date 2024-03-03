---
{"dg-publish":true,"permalink":"/automatic-set-alarms-with-shortcut/","created":"2024-03-03T22:03:40.314+08:00","updated":"2024-03-03T23:11:55.787+08:00"}
---

The Alarms are necessary for workdays. When it comes to holidays, the workdays are not always weekdays. Here, the Shortcut on iOS could help to set the alarms automatically, regardless of the any day of the week.

## Principle

1. If today is a workday, Monday to Friday, and it has an event with "休"，do not set the alarm; otherwise, set the alarm.
2. If today is during the weekend, and it has an event with "班", set the alarm; otherwise, do not set the alarm.

## Implementation

![asserts/5cef52777d7b1d9025a3042e4d352a79_MD5.jpeg](/img/user/asserts/5cef52777d7b1d9025a3042e4d352a79_MD5.jpeg)

[work alarm share link](https://www.icloud.com/shortcuts/29675182905945aa960d34293b438feb)
The Shortcut needs the parameters below:
1. A one time alarm for work.
2. Subscribe the calendar of "中国大陆节假日".
3. Create a Calendar of "work". Add custom non-work day by creating an all-day event in "work" Calendar containing "休"。
Run and verify if it works. 
In Automation section, create an automation to run the Shortcut before the alarm time daily.


## Reference

https://github.com/lanceliao/china-holiday-calender