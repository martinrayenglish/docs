---
layout: default
title: Session State and the xDB
category: xdb
---

> This is a work in progress! 

## First of all - what are my options?

In a content delivery environment, you can choose to use **InProc** our **OutProc** session state management. 
**InProc** is short for 'In Process', and means that any information about a visitor's session is stored in memory. This is the default configuration when you install Sitecore, and it is your *only* option for content management environments. InProc is always, always going to be faster than OutProc, because you are not writing anything to disk.

**OutProc**, short for 'Out of Process', is when you store session state information somewhere that *isn't* in memory. For example, you might write your session state information to a SQL database. Sitecore offers two OutProc session state providers: **MongoDB** and **SQL**.

### Shared vs Private, InProc vs OutProc

The xDB stores [two kinds of session information - **shared** and **private**](https://doc.sitecore.net/products/sitecore%20experience%20platform/xdb%20configuration/session%20state). You can think of shared session state as the 'contact' session state - it has information about the contact, devices used, and engagement plan states. Private session state contains information about interactions - such as goals triggered. When you install Sitecore, **both** session stores are set to `InProc`.

#### Using `InProc` for both private and shared

This is the default setup, and fine if you have a single CD. If you have more than one CD and wish to continue using InProc session management, you must use **sticky sessions**. When a visitor hits your site, they are locked to a particular CD instance - all of their session activity is managed in memory by that machine. If your session is interrupted, you may lose your data.

#### Using `InProc` for private, `OutProc` for shared

This setup is **not recommended** even though it looks like you could have this configuration, as you are getting the worst of both worlds - reduced speed with OutProc, and increased unreliability with InProc. You still need to use sticky sessions to ensure that private session state is not lost if you are moved from one CD to another, and in the event that your session is interrupted, you risk losing everything in the private session store. The only 'pro' is that shared session state is available to any other, concurrent sessions that may be started up by that contact on a different device.

#### Using `OutProc` for private and shared

Slower, but most reliable. No matter which CD a visitor hits within a cluster, their session data is available in the session database.

### Which session state provider should I use?

That's up to you. There is no officially supported data that suggests one is faster than the other, although you should always perform your own tests. MongoDB is simpler to spin up, but if you are not comfortable supporting MongoDB, you can use the SQL provider.

### What about content management (CM) environments

You have to use `InProc` for CM environments - for both private and shared session state. There is no support for `OutProc`.

## If OutProc is slower, why would I use it - and what does it have to do with analytics? 

There are two key advantages to OutProc session management:

* You trade some speed for **increased reliability**
* You can share session information across **multiple devices** 

### Reliability

In Sitecore's xDB, information about a visitor's is built up during their session and flushed to MongoDB on session end. Imagine that you are using InProc session management - everything is stored in memory. If anything goes awry in your content delivery environment (for example, one out of three load-balanced instances goes down), visitors are at risk of losing their entire session. By contrast, if you are using OutProc session management, their session is maintained in a database - the information about that session is not lost.

Why does this matter? For marketers, Sitecore 8 is all about getting a full picture of the individual. Interfaces like the Experience Profile lets you use Sitecore like a CRM - it collects all manner of information about what a visitor has done over time, what part of the sales process they are in, and how they are interacting with your brand. This kind of detailed information might not matter as much to you if your orders rarely exceed £10 - but if you are a vendor of luxury holiday packages, it probably does. In this scenario, individual session data is likely to be worth a lot more; if even a single session is lost, a contact may not be seen as moving through the sales pipeline even though they are.

### Sharing session data across devices

There is a small chance that a visitor will have two concurrent sessions running. Consider the following scenario (note that if Bob does not identify himself by logging in or similar, Sitecore has no way of knowing that he is the same person on both devices):

* Bob logs onto travelling website and browses around for holidays - he triggers a number of goals and is moved from 'New Visitor' engagement state to 'Looking for Holidays'.
* Bob finishes work, leaves his computer, and immediately picks up his phone to continue the search on the commute home.
* He logs onto the same site, and contiues browsing. Bob now has two sessions running at the same time.
* If shared session state is being managed **out of process**, the xDB knows that Bob already has an active session - it is able to use information about the contact and his engagement level state. If this information was not available in shared session state, Bob would be seen as being a 'New Visitor' on his mobile phone despite that not being the case.


Sharring session data across devices **in a multi-CD setup** requires `OutProc` session management. Every CD needs access to the shared session state in order to know about contact details and engagement states.

### Cluster-forwarding 

In a geographically distributed setup, you have a cluster of CDs and session state server *per cluster*. The session state server needs to be as close to the CDs as possible to ensure the best possible performance. If for any reason a contact starts a session on cluster A, and a second, concurrent session on a different device in cluster B, they are [i]redirected to their original cluster[/i] by the CD environment. This is possible because the first session causes a lock to be placed on the contact, which ties them to a particular cluster. This behaviour is not possible without `OutProc` session management.

## Do I have to use OutProc session management?

No, not at all. Nothing is going to break, technically.