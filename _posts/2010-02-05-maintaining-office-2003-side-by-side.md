---
layout: post
title: Maintaining Office 2003 Programmability with Side-by-Side Office Install
categories: [.net]
---

In order to build for Office 2003 interop/automation while having Office 2007 installed in a side-by-side installation, you need to remove the .NET programmability support from each 2007 product. This ensures that the latest version available in the GAC is version 11 (2003), since most of the time .NET builds will pull from there instead of a local reference.

To do this, go to add/remove programs and call up the “Change Features” window for Office 2007, find the item called “.NET Programmability Support” under each individual product you have installed, and remove it. There are also items under Office Tools for Graph and Smart Tags.

Then, your GAC will include only version 11 assemblies for each product rather than 11 and 12. You should be able to build as you did before under 2003 alone.  