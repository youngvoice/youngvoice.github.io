---
title: C++ atomic
description: ? 
categories: [cs, program, language, c/c++, atomic]
tags: [cs, program, language, c/c++, atomic]

---

# Background

There are many atomic aquire and release operation in glibc source code, so what is meaning of this?

# keynote

data race and optimizaiton

ordering

object layout

software MMs have converged on SC for data-race-free programs (SC-DRF).
Hardware MMS are disadvantaged unless SC acquire/release are the primary HW MM instructions.

# ref 

C++ and Beyond 2012: Herb Sutter 
<https://www.youtube.com/watch?v=A8eCGOqgvH4>
<https://www.youtube.com/watch?v=KeLBd2EJLOU>
