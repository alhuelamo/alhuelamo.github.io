---
title: "Native Apple Silicon JDKs"
date: 2021-11-27T18:33:14+01:00
slug: 2021-11-27-apple-silicon-jdk
type: posts
draft: true
categories:
  - tooling
tags:
  - mac
  - jvm
  - scala
  - sbt
---

{{<figure src="/img/joey-banks-F0otTOnRYUU-unsplash.jpg" caption="Photo by [Joey Banks](https://unsplash.com/@joeyabanks) in [Unsplash](https://unsplash.com/s/photos/m1-apple)">}}

I bought an M1 Mac Mini, and I have been using it as my main personal desktop — games aside — for the last months. I'm extreamly pleased with this machine: it's small, it feels fast and snappy, it's quiet — I have never ever heard the fans — and its power consumption [is very low](https://support.apple.com/en-us/HT201897).

Apple's plan is to carry out a 2-year transition from November 2020 to replace its Intel-based Macs by M-something ones. Changing base architectures is a challenging process, and one would expect it to be quite painful for users: early adopters should expect compatibility issues with applications, frameworks, or other kinds of software. However, Apple managed to smooth the process by introducing the Rosetta 2 compatibility layer. The '2' means that this is not a new concept: they already released the first version of Rosetta when they [transitioned from PowerPC](https://en.wikipedia.org/wiki/Mac_transition_to_Intel_processors) back in 2005. Rosetta 2 reimplements the same concept for the ongoing transition, which is that Intel based applications may run seamlessly on Apple Silicon based Macs with no significant performance penalties. For my particular use case this has worked very well. I could ran eveything I needed, even if it was not released for M1 processors yet, with no major impact in performance, if any.

# JDKs and Rosetta 2

The JVM is no exception.

When I started using my brand new Mac Mini, one of the first things I installed was [SDKMAN](https://sdkman.io), a package manager I used for my basic JVM tooling. First things first, I installed Adopt OpenJDK 11 — since 11 is the version we use at work for most of our data pipelines — and boom, there was it: `java -version` worked, no questions asked, and it didn't feel slower than in my corporate Intel-powered Macbook Pro. So, I was happy.

Then some time passed, and after speaking with some collegues, I realized of one — not so obvious — thing: I was not using an Apple-Silicon-native JDK — I would argue this is another symptom of how well Rosetta 2 works, because I was actually fine with performance of my processes. You know, Java is not well know for its faster startup times, so the fact that SBT took 1 or 2 seconds to start up was not out of place. However, it could be better.

In SDKMAN you can run

```sh
sdk list java
```

which shows a table of the available JDK versions. Here is a potion of that table:

```txt
================================================================================
Available Java Versions
================================================================================
 Vendor        | Use | Version      | Dist    | Status     | Identifier
--------------------------------------------------------------------------------
 AdoptOpenJDK  |     | 16.0.1.j9    | adpt    |            | 16.0.1.j9-adpt
               |     | 16.0.1.hs    | adpt    |            | 16.0.1.hs-adpt
               |     | 11.0.11.j9   | adpt    |            | 11.0.11.j9-adpt
               |     | 11.0.11.hs   | adpt    |            | 11.0.11.hs-adpt
               |     | 8.0.292.j9   | adpt    |            | 8.0.292.j9-adpt
               |     | 8.0.292.hs   | adpt    |            | 8.0.292.hs-adpt
 Corretto      |     | 17.0.1.12.1  | amzn    |            | 17.0.1.12.1-amzn
               |     | 17.0.0.35.2  | amzn    |            | 17.0.0.35.2-amzn
               |     | 16.0.2.7.1   | amzn    |            | 16.0.2.7.1-amzn
               |     | 11.0.13.8.1  | amzn    |            | 11.0.13.8.1-amzn
               |     | 11.0.12.7.2  | amzn    |            | 11.0.12.7.2-amzn
               |     | 8.312.07.1   | amzn    |            | 8.312.07.1-amzn
               |     | 8.302.08.1   | amzn    |            | 8.302.08.1-amzn
```

You can see there is nothing indicating whether each JDK is Intel or Apple Silicon native.

## Filtering M1-native JDKs

After doing some ~~thorough~~ [research](https://github.com/sdkman/sdkman-cli/issues/830) I found out we can tell SDKMAN — version 5.10.0+ — to filter out non-native packages. In the file `~/.sdkman/etc/config` we can set the following setting:

```txt
sdkman_rosetta2_compatible=false
```

Which by default is set to true. Then, we can open a new terminal to reload SDKMAN and run the same previous query:

```txt
================================================================================
Available Java Versions
================================================================================
 Vendor        | Use | Version      | Dist    | Status     | Identifier
--------------------------------------------------------------------------------
 Corretto      |     | 17.0.1.12.1  | amzn    |            | 17.0.1.12.1-amzn
               |     | 17.0.0.35.2  | amzn    |            | 17.0.0.35.2-amzn
 Java.net      |     | 18.ea.25     | open    |            | 18.ea.25-open
               |     | 18.ea.24     | open    |            | 18.ea.24-open
               |     | 18.ea.6.lm   | open    |            | 18.ea.6.lm-open
               |     | 18.ea.5.lm   | open    |            | 18.ea.5.lm-open
               |     | 17           | open    |            | 17-open
               |     | 17.0.1       | open    |            | 17.0.1-open
 Liberica      |     | 17.0.1.fx    | librca  |            | 17.0.1.fx-librca
               |     | 17.0.1       | librca  |            | 17.0.1-librca
               |     | 17.0.0.fx    | librca  |            | 17.0.0.fx-librca
               |     | 17.0.0       | librca  |            | 17.0.0-librca
               |     | 16.0.2       | librca  |            | 16.0.2-librca
               |     | 11.0.13      | librca  |            | 11.0.13-librca
               |     | 11.0.12      | librca  |            | 11.0.12-librca
               |     | 8.0.312      | librca  |            | 8.0.312-librca
               |     | 8.0.302      | librca  |            | 8.0.302-librca
 Microsoft     |     | 17.0.1       | ms      |            | 17.0.1-ms
               |     | 17.0.0       | ms      |            | 17.0.0-ms
               |     | 16.0.2.7.1   | ms      |            | 16.0.2.7.1-ms
 Oracle        |     | 17.0.1       | oracle  |            | 17.0.1-oracle
               |     | 17.0.0       | oracle  |            | 17.0.0-oracle
 SapMachine    |     | 17           | sapmchn |            | 17-sapmchn
               |     | 17.0.1       | sapmchn |            | 17.0.1-sapmchn
 Temurin       |     | 17.0.1       | tem     |            | 17.0.1-tem
               |     | 17.0.0       | tem     |            | 17.0.0-tem
 Zulu          |     | 17.0.1       | zulu    |            | 17.0.1-zulu
               |     | 17.0.1.fx    | zulu    |            | 17.0.1.fx-zulu
               |     | 17.0.0       | zulu    |            | 17.0.0-zulu
               |     | 17.0.0.fx    | zulu    |            | 17.0.0.fx-zulu
               |     | 16.0.2       | zulu    |            | 16.0.2-zulu
               |     | 11.0.13      | zulu    |            | 11.0.13-zulu
               |     | 11.0.12      | zulu    |            | 11.0.12-zulu
               |     | 8.0.312      | zulu    |            | 8.0.312-zulu
               |     | 8.0.302      | zulu    |            | 8.0.302-zulu
================================================================================
```

We can see some differences, for instance, `AdoptOpenJDK` lines are gone.

# Performance boost

I decided to give a try to Zulu JDK 11, and so I installed Zulu JDK 11. The first I did after was to start up sbt on my Scala playground project. SBT usually takes several seconds to boot up, but even with that I could notice it was substancially faster than with the non-native JDK.

I decided then to run a very simple experiment: I would run `sbt clean compile` for 10 times and then I would compute the average.

These were the results.

| JDK          | Version | Platform      | Average time for 10 runs |
| ------------ | ------- | ------------- | ------------------------ |
| AdoptOpenJDK | 11.0.11 | x86-64        | 24.524 seconds           |
| Temurin      | 17.0.1  | Apple Silicon | 11.804 seconds           | 
| Zulu         | 11.0.13 | Apple Silicon | 12.912 seconds           |
| Liberica     | 11.0.13 | Apple Silicon | 14.170 seconds           |

The results show how native JDKs can better leverage these new Apple ships. In this silly experiment, Zulu turned out to be twice as fast as the AdoptJDK running on Rosetta 2, while Temurin JDK 17, the new name for AdoptOpenJDK, which also features the [latests performance upgrades](https://www.optaplanner.org/blog/2021/09/15/HowMuchFasterIsJava17.html), is even faster. However, the gratest gains result from running the platform native JDKs.

# Conclusion

This Apple's hardware architecture transition is resulting so smooth it fooled me: Rosetta 2 did not make noticeable I wasn't using a non-native framework, but after discovering *the truth*, boy am I happy. SBT feels snappier, so as bloop and other tooling. We just need to choose the right version of our tools for the platform we work on.
