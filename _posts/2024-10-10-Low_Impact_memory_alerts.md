---
title: A Low-Impact Memory Leak Alert System
date: 2024-10-10 08:00:00 +0000
#categories: [TOP_CATEGORY, SUB_CATEGORY]
tags: [memory leaks, ci/cd, efficiency, memory, ML, AI applications]     # TAG names should always be lowercase
---

In June 2024, I had the opportunity to present my work on a **low-impact memory leak alert system** at the *SPIE Astronomical Telescopes and Instrumentation* conference. The project grew from a practical need: in large, long-running software environments, small memory leaks can quietly accumulate until they cause instability or performance degradation. Detecting them early is difficult. Traditional tools are often heavy, intrusive, or impractical for routine testing, so the goal was to build something much lighter that could quietly run in the background during unit and integration tests, catching problems before they ever reached production.

## Motivation

Memory leaks are rarely spectacular. They don't crash a system immediately, but rather creep up over time, stealing memory little by little until something breaks. By the time they're noticed, the cause may be buried deep within a codebase or hidden behind complex runtime interactions.  

When thinking about this, I wanted a way to raise early alerts, not to pinpoint the exact line of code leaking memory, but simply to tell developers: *something here looks wrong*. That would be enough to trigger a closer investigation using more heavyweight tools later on.  

For this to work, the tool had to be lightweight enough to run every night in continuous integration (CI) testing, language-independent enough to monitor any process, and simple enough that developers could trust its results without spending hours configuring or interpreting them.

## System Design

The alert system was written in Python and designed to integrate directly with **Pytest**, since that's widely used for test automation. It runs a background thread during tests, periodically recording each process's virtual memory usage, process name, and PID. The idea was to make data collection almost invisible: minimal overhead, minimal data stored, and no interruption to normal testing.  

Once the tests finish, the system analyses this recorded data using what I describe as a **hybrid approach**: the data collection is dynamic (it happens live during testing), but the analysis itself is static (it runs after the fact). This balance keeps the test environment responsive while still allowing thorough examination of trends.  

Pre-processing steps clean up and resample the data to ensure consistent time intervals, then pass it to a detection algorithm based on **Linear Backward Regression (LBR)**. This method fits a line to the final section of a process's memory usage history. If the line fits well and the projected memory use would exceed system limits within a given period, for example, an hour, that process is flagged as suspicious.  

This approach may sound simple, but that's its strength. It doesn't require massive training data or deep models. It just looks for clear, sustained growth where none should exist. The result is interpretable and fast enough to run as part of every test cycle.

## In Practice

The system is now used as part of ongoing continuous integration pipelines, where it automatically monitors memory usage across different modules as tests run. It has a very low footprint, so it can run alongside other test jobs without noticeably slowing them down.  

Developers can use it to view how memory behaves across runs, check whether revisions remain stable, or quickly identify processes behaving abnormally. When it raises an alert, they can then move to more detailed profiling tools to find the precise cause.  

The tool has already proven its worth in practice. In one case, it identified a leak that had escaped normal testing a set of unjoined threads that were quietly retaining memory across cycles. The alert system flagged the behaviour within seconds of analysis, long before the issue became visible during runtime.  

At the same time, it also helped confirm non-issues, showing that recent changes hadn't altered memory behaviour in expected ways. That reassurance is just as valuable. Importantly, the whole analysis phase running across hundreds of processes from a full test suite typically takes under a minute, which makes it completely viable for nightly testing.

## Reflection and Presentation

As an aside, presenting this work at SPIE was a highlight of my year. The poster I presented was done so as part of a session focused on software reliability and maintainability in complex systems topics that feel increasingly relevant as software underpins more of our research infrastructure. It was great to share something that wasn't theoretical but instead designed to help developers day-to-day.  

The discussions afterward were particularly interesting. Many engineers from other projects described facing similar problems: long-lived systems, complex dependencies, and limited tolerance for downtime. A tool that could provide simple, early warnings resonated with a lot of people.  

Since then, the system has continued to evolve, with an open-source version available for anyone to use or adapt. It's designed to be practical, so others can integrate it into their own CI setups or testing frameworks without needing to rebuild everything from scratch.

## Looking Ahead

While the core of the tool is stable, there's still room to grow. Future updates could explore new detection models beyond simple regression, ones capable of recognising cyclic or 'sawtooth' leak patterns, where memory usage resets intermittently but still trends upward overall. There's also potential in adaptive thresholding, where the system could learn baseline memory profiles for specific processes and flag deviations relative to their usual behaviour.  

The main idea, however, will remain the same: to keep memory leak detection easy, fast, and integrated into everyday testing rather than something reserved for emergency debugging sessions.

## Further Reading

For anyone interested in more technical details or trying the tool themselves, the following links might be useful:

- **GitHub Repository**  [MemoryTools on GitHub](https://github.com/BenjaminCarpenter480/memorytools): Source code, examples, and documentation for the alert system.  
- **Conference Write-Up**  [Low-Impact Memory Alert System (SPIE Proceedings PDF)](https://observatorysciences.co.uk/wp-content/uploads/2024/07/BPC_Low-Impact-Memory-Alert-System-SPIE-proceedings.pdf): the full technical paper.  
- **Poster Presentation**  [SPIE Digital Library Poster Download](https://www.spiedigitallibrary.org/proceedings/Download?urlId=10.1117%2F12.3018392): the accompanying visual summary presented at the conference.  
ystem
