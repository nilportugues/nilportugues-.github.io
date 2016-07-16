---
published: false
title: Android Tabs Refactoring, from spaghetti to MVP
layout: post
---
This post is about my learnings in Android. 

I have been coding for Android only 2 weeks now (this been the second). As a newcomer, I found that the approach to build a Tabs interface, while understanding, was not clean at all. I was having in the very same place business logic, views, fragments and adapters.

This is the original code: 

```java
```

After understanding how it works I went for the refactoring to MVP.  MVP approach offers very interesting benefits:

- Provides a clear separation of UI and business logic.
    - View handles all classes extending from `android.view.View`. Also calls the adapters required to work with the Fragments.
    - Presenter handles the logic on how the View behaves based on the business logic provided by a service.
- Maintainability of UI increases due to almost no dependency on business logic. 
- Mocking of the Presenter allows testing the View's behavior.