---
published: false
title: Android Tabs Refactoring, from spaghetti to MVP
layout: post
---
This post is about my learnings in Android so far. 
Disclaimer notice, I have been coding for Android only 2 weeks now (this been the second). 

## Setup

The following libraries are being used throught this post and understanding of them is required to make out the most of this piece of writing: 

- `com.android.support:appcompat-v7:23.4.0`
- `com.android.support:design:23.4.0`
- `com.google.dagger:dagger:2.4`
- `com.jakewharton:butterknife:8.1.0`
- `io.reactivex:rxjava:1.1.6`
- `io.reactivex:rxandroid:1.2.1`

## The problem
As a newcomer, I found that the approach to build a Tabs interface, while understanding, was not clean at all. I was having in the very same place business logic, views, fragments and adapters.

This is the original code: 

```java
```

Yikes!

## The solution: MVP
After understanding how it works I went for the refactoring to MVP.  

**Benefits**

MVP approach offers very interesting benefits over the straight-forward approach:

- Provides a clear separation of UI and business logic.
    - View handles all classes extending from `android.view.View`. Also calls the adapters required to work with the Fragments.
    - Presenter handles the logic on how the View behaves based on the business logic provided by a service.
- Maintainability of UI increases due to almost no dependency on business logic. 
- Mocking of the Presenter allows testing the View's behavior.

**Downsides**

- More code, whcih is always the case when wirting decoupled code.

### Step 1: Contract between the Presenter and the View
### Step 2: Moving android.view.View to our new View class.
### Step 3: Moving business logic to our new Presenter class.
### Step 4: Rewriting the TabActivity