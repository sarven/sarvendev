---
title: Problematic second
date: 2024-07-01
categories: [Interesting]
permalink: /2024/07/problematic-second/
image:
  path: /assets/img/2024-07-01/featured.png
---
Today, each of us has many possibilities to check the current time. We have smartphones, watches, computers, TVs, refrigerators, ovens, etc. Everyone knows what a year is and what it signifies. We understand time zones and UTC time. Everything seems trivial. However, from the point of view of IT systems, there are many surprises that we can encounter. Unfortunately, this causes us to face problems that occur very rarely.

There are many problems with time that exist in different IT systems, depending on the type of system. Mostly, they are related to the accuracy of measuring time and synchronizing the time between machines in distributed systems. In this article, I’m going to focus on one strange situation that has only occurred 27 times in history.

## Leap second
Initially a second was defined as 1/86400 of day. Then in 1960 the definition was changed to 1/31 556 925,9747 of tropical year. However, it turned out, that the move of the Earth around the Sun isn’t always the same, and sometimes there are differences cause by tides, movement of masses within the planet as well as earthquakes. Therefore, in 1967, the value of second was defined in the completely different way:
> This is equal to 9,192,631,770 periods of radiation corresponding to the transition between the two levels F = 3 and F = 4 of the ground state hyperfine structure S1/2 of the caesium 133Cs atom

The most accurate clocks are atomic clocks, which just measure time based on the rate at which electrons pass between the two shells of an atom. However, there remained the question of compensating for the difference between the values measured by atomic clocks and the actual time the Earth revolves around the Sun.

For this purpose, the leap second was invented. IERS (International Earth Rotation and Reference Systems Service) decides when that second should be added. Usually, it is added on the last day of June or December. A deduction of one second is also possible, but as of today this has not yet occurred.

The list of leap seconds:
> 1. (+11s) June 30, 1972  
> 2. (+12s) December 31, 1972  
> 3. (+13s) December 31, 1973  
> 4. (+14s) December 31, 1974  
> 5. (+15s) December 31, 1975  
> 6. (+16s) December 31, 1976  
> 7. (+17s) December 31, 1977  
> 8. (+18s) December 31, 1978  
> 9. (+19s) December 31, 1979  
> 10. (+20s) June 30, 1981  
> 11. (+21s) June 30, 1982  
> 12. (+22s) June 30, 1983  
> 13. (+23s) June 30, 1985  
> 14. (+24s) December 31, 1987  
> 15. (+25s) December 31, 1989  
> 16. (+26s) December 31, 1990  
> 17. (+27s) June 30, 1992  
> 18. (+28s) June 30, 1993  
> 19. (+29s) June 30, 1994  
> 20. (+30s) December 31, 1995  
> 21. (+31s) June 30, 1997  
> 22. (+32s) December 31, 1998  
> 23. (+33s) December 31, 2005  
> 24. (+34s) December 31, 2008  
> 25. (+35s) June 30, 2012  
> 26. (+36s) June 30, 2015  
> 27. (+37s) December 31, 2016  

## Time and IT systems
Timing is actually present in most projects. We need to be able to specify when a particular user action occurred, when a particular email is to be sent. It can also be important to determine the exact order of events. In the case of a single machine, this is not very complicated. However, in the case of distributed systems, where we are dealing with multiple nodes, the situation is more complicated. The hardware times given by the usual crystal oscillator are not entirely accurate and there can be differences between nodes. When the sequence of events is important, a discrepancy in the clock readings of the nodes is unacceptable. This is why NTP (Network Time Protocol) is used, which matches the time indicated by a particular machine to the time given by a group of servers.

There are at least two clocks in computers: real-time clocks and monotonic clocks.

## Real-time clocks
They simply return the actual date and time. In Unix-based systems, the clock is based on the date January 1, 1970, 00:00:00, and from this moment, it counts the number of seconds. These clocks are problematic because they are synchronized by NTP, which can cause them to jump backward and forward. Therefore, they aren’t entirely suitable for measuring the passage of time, such as timeouts.

## Monotonic clocks
Monotonic clocks are more precise, providing values in microseconds or even more accurately. They are implemented as counters, which, for example, could start when the system starts. NTP can only adjust the counting speed, slowing down or speeding up the particular clock. Since there are no jumps, they are a better choice for measuring the passage of time.

## Consequences of bad synchronization
The result of poor synchronization or out-of-sync clocks can lead to data loss. The system is better off crashing completely than operating incorrectly and unnoticed, which can cause data corruption. Therefore, in situations where we rely on clocks, they must be properly synchronized and monitored. If any clock is malfunctioning, it should be considered completely out of order.

## Ordering events
For example, in replication with multiple leaders, the strategy of last record wins is used. So, if two nodes have unsynchronized clocks, the difference can be even 2 or 3 ms, a situation can arise where the order of writes is switched, leading to incorrect data. However, it turns out, that even the synchronization by NTP could be not sufficient, as it is also heavily influenced by network delays. Therefore, logical clocks based on incrementing counters should be rather used to order events. They do not measure time, but only determine the relative ordering of events.

Some databases like e.g. Spanner from Google use the TrueTime API, which returns, respectively, the range of time that can currently be (that is, it takes into account the error that can occur) [earliest, latest]. Such a database appropriately manages transactions referring to this error, so as to ensure the correct order. Of course, to have the proper performance the range of time shouldn’t be too large. Therefore, Google in each of their data center uses GPS or atomic clock to determine the current time, allowing the clocks to be synchronized to an accuracy of about 7ms.

As you can see, time in IT projects is not a trivial thing. However, let’s focus on the leap second and the problems associated with it.

## Problems and failures
Programmers know that if something happens rarely, it is expected that it will be different from what was assumed because nobody has experienced it before. In the case of leap seconds, such a situation occurred 27 times in 52 years, averaging once every two years. This is extremely rare, so various problems and software errors were inevitable. The worst period was between 1999 and 2005, during which leap seconds were essentially forgotten for six years.

30 June 2012, a serious drop in performance was reported on Reddit, at first administrators blamed a network slowdown that had also occurred a few days earlier. After deeper investigation it turned out that Linux machines had very high CPU usage. The problem was a bug in the Linux kernel, the “hrtimer” module went crazy when an extra second was added. Reddit was not the only one affected, as other companies also noted such problems. Linus Torvalds commented on situation as follows:

> Almost every time we have a leap second, we find something. It’s really annoying, because it’s a classic case of code that is basically never run, and thus not tested by users under their normal conditions.

## Explanation of the “hrtimer” problem
Hrtimer is the module responsible for waking up applications from an idle state. Applications that, for example, are waiting for the operating system to perform some task, and only later will they be able to perform their actions. When a leap second was added, the module went crazy and woke up all applications, resulting in high CPU usage and a gigantic drop in performance, or even a complete system crash in some cases.

In the case of Reddit, the problem mainly affected the Cassandra database and Java, while similar problems related to the same bug were encountered in MySQL databases, Tomcat servers, or Hadoop.

Interestingly, Opera’s servers crashed the day before, after the NTP servers announced a leap second.

## Power consumption
The Hetzner Online data center reported that on the night of June 30-July 1, 2012, energy consumption jumped significantly to above 1 megawatt. This was caused by the bug described above. Processors suddenly started running at 100% load, which translated into an increase in power consumption. The graph shows 2:00 a.m., probably because the time zone is included here. A similar spike in energy consumption also occurred at OVH. Likely, other centers observed the same anomaly, although only these two reported such observations.

![Power consumption](/assets/img/2024-07-01/power-consumption.png)

## The problem of the year 2000
Several decades ago, we did not have such comfort to use memory without almost no limit.
![Tweet RAM](/assets/img/2024-07-01/tweet.png)

Therefore, the year was recorded by giving only the last two digits. The first problems began to appear in the 1990s, when determining, for example, the validity of a credit card, which went beyond the year 2000. Many people talked about the disaster that would occur because of this, a real apocalypse was predicted. The end of the world, admittedly, did not happen, but it is still one of the stupidest mistakes made by programmers. Thinking only about the near future is never good. In the end, it took a lot of work to bring the software up to scratch, which of course came at a cost.

## Implementations of the leap second
The simplest implementation is simply announcing a leap second by NTP servers. Then at the end of the minute, when this second is to be added the value 59 appears twice.

Another approach is the one used in Windows mainly, the system ignores the leap second. Only on the next synchronization with NTP does the time jump to the correct value.

The above two solutions have their drawbacks, because in the first one, we have the same time twice, and in the second one we have a strange jump, so Google decided to approach the subject differently. They came up with a solution called “leap smear”, which gradually adds milliseconds to the time throughout the day, instead of adding a whole second at one point. To be precise, for every second of the day when a leap second is to be added, a value of 1/86400 is added, which simply corresponds to spreading that one extra second over the seconds of the day (24*60*60 = 86400).

## Future time problems
The next problem with time may occur on January 19, 2038. On Unix-based systems, time is stored with the number of seconds from January 1, 1970 00:00:00. The number of seconds is stored in a signed integer variable with a maximum positive value of 2,147,483,647 (32 bits), which corresponds to the time January 19, 2038 03:14:07. The solution to the problem will be switching to writing time using 64 bits, which will significantly increase the range of values (2^63-1). Using 64 bits, the next such problem will not occur until the year 292,277,026,596, that is, in 292 billion years. Operating systems are more likely to be adapted as seamlessly as possible. However, it is known that there will always be someone who still uses some old undeveloped software and then has a serious problem.

## End of the leap second
Following years of discussions by various standards bodies, the decision to abolish the leap second by or before 2035 was made at the 27th General Conference on Weights and Measures in November 2022. Starting in 2035, leap seconds will no longer be a concern. International Bureau of Weights and Measures will permit UTC and UT1 to drift apart until at least 2135, in the hope that scientists will develop a better method for accounting for lost time—or that computers will become more adept at managing clock changes.

## Summary
As you can see, handling time is not easy, especially in the case of leap seconds, where the actual operation of the solution in production is checked very rarely. Such problems are bound to keep happening. It is not surprising, since these are rather relatively low-level solutions, which are used by a whole bunch of other things. However, it must be admitted that the most predictable and safe solution seems to be the one from Google called “Leap smear”, where there is not some unnatural jump once every x years, but everything is done smoothly without affecting normal operation. If you want to read more about weird things related to time, check this site: [https://yourcalendricalfallacyis.com/](https://yourcalendricalfallacyis.com/)

P.S. On the day the leap second is added, if possible, it is better not to get on a plane, and it is best to stay at home. For sure something will not work again, we will see what this time 😀.
