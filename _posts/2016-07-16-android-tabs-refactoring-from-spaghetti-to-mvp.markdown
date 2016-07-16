---
published: true
title: Android Tabs Refactoring, from spaghetti to MVP
layout: post
tags: [android, dagger, butterknife, rxjava, rxandroid, reactive]
categories: [Android, Refactoring]
---
This post is about my learnings in Android so far. 

*Disclaimer notice, I have been coding for Android only 2 weeks now (this been the second).*

## Setup

The following libraries are being used throught this post and understanding of them is required to make out the most of this piece of writing: 

**Libraries**

- `com.android.support:appcompat-v7:23.4.0`
- `com.android.support:design:23.4.0`
- `com.google.dagger:dagger:2.4`
- `com.jakewharton:butterknife:8.1.0`
- `io.reactivex:rxjava:1.1.6`
- `io.reactivex:rxandroid:1.2.1`



## The problem

As a newcomer, I found that the approach to build a Tabs interface, while understanding, was not clean at all. I was having in the very same place business logic, views, fragments and adapters.

This is what we're building:

<div style="text-align:center"><img src="https://nilportugues.github.io/public/images/2016-07-tabs-refactor.png" height="500"></div>

This is the original code: 

**TabsWithTextActivity.java**

```java
package com.nilportugues.simplewebapi.users.ui.activities.tabs;

import android.os.Bundle;
import android.support.design.widget.TabLayout;
import android.support.v4.app.Fragment;
import android.support.v4.view.ViewPager;
import android.support.v7.app.ActionBar;
import android.support.v7.widget.Toolbar;

import com.nilportugues.simplewebapi.R;
import com.nilportugues.simplewebapi.MyApplication;
import com.nilportugues.simplewebapi.users.di.components.UserComponent;
import com.nilportugues.simplewebapi.users.interactors.UserPokemonList;
import com.nilportugues.simplewebapi.users.ui.BaseFragmentActivity;
import com.nilportugues.simplewebapi.users.ui.usetabstext.TabsWithTextAdapter;
import com.nilportugues.simplewebapi.users.ui.userpager.FragmentOne;
import com.nilportugues.simplewebapi.users.ui.userpager.FragmentThree;
import com.nilportugues.simplewebapi.users.ui.userpager.FragmentTwo;

import java.util.ArrayList;
import java.util.List;

public class TabsWithTextActivity extends BaseFragmentActivity {

    @Inject UserPokemonList interactor;
    @BindView(R.id.tabs1_view_pager) ViewPager viewPager;
    @BindView(R.id.tabs1_layout) TabLayout tabLayout;
    @BindView(R.id.tabs1_toolbar) Toolbar toolbar;
    @BindView(R.id.tabs1_progressbar) ProgressBar progressBar;
    
    private List<Fragment> fragmentList = new ArrayList<>();
    private List<String> titleList = new ArrayList<>();
    private List<String> userPokemon;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        getComponent().inject(this);
        removeActionBar();
        loadPokemon();
    }

    @Override
    protected int getLayoutId() {
        return R.layout.users_tabs_text;
    }
    
    public UserComponent getComponent() {
        return ((MyApplication) getApplication()).getUserComponent();
    }
    
    private void removeActionBar() {
        ActionBar actionBar = getSupportActionBar();
        if (actionBar != null) {
            actionBar.hide();
        }
    }    

    private void loadPokemon() {
        progressBar.setVisibility(android.view.View.VISIBLE);
        interactor.execute(new UIThread(), new IOThread(), new Subscriber<List<String>>() {

            @Override
            public void onCompleted() {
                progressBar.setVisibility(android.view.View.GONE);
                buildTabs();
            }

            @Override
            public void onError(Throwable e) {
                progressBar.setVisibility(android.view.View.GONE);
                mToolbar.setTitle("Could not load tabs");
                buildView();
            }

            @Override
            public void onNext(List<String> strings) {
                setList(strings);
            }
        });    
    }

    private void buildTabs() {
        buildTabsTitle();
        buildTabsContent();
        buildView();
    }

    private void buildTabsTitle() {
        toolbar.setTitle("Text Tabs");
    }

    private void buildTabsContent() {
        this.buildFragment(new FragmentOne(), this.userPokemon.get(0));
        this.buildFragment(new FragmentTwo(), this.userPokemon.get(1));
        this.buildFragment(new FragmentThree(), this.userPokemon.get(2));
    }

    private void buildFragment(Fragment fragment, String title) {
        fragmentList.add(fragment);
        titleList.add(title);
    }
    
    private void buildView() {
        TabsWithTextAdapter adapter = new TabsWithTextAdapter(mFragmentManager, fragmentList, titleList);
        mViewPager.setAdapter(adapter);
        mTabLayout.setupWithViewPager(mViewPager);
    }    
}
```

**TabsWithTextAdapter.java**

This is just required to get the tabs working. It will load the fragments and allow us to manage them. 

For more on this, read the official [documentation](https://developer.android.com/reference/android/support/v4/app/FragmentPagerAdapter.html).

```java
package com.nilportugues.simplewebapi.users.ui.adapters.tabs;

import android.support.v4.app.Fragment;
import android.support.v4.app.FragmentManager;
import android.support.v4.app.FragmentPagerAdapter;

import java.util.List;

public class TabsWithTextAdapter extends FragmentPagerAdapter {

    private List<Fragment> fragmentList;
    private List<String> titleList;

    public TabsWithTextAdapter(FragmentManager fm, List<Fragment> fragmentList, List<String> titleList) {
        super(fm);
        this.fragmentList = fragmentList;
        this.titleList = titleList;
    }

    @Override
    public Fragment getItem(int position) {
        return fragmentList.get(position);
    }

    @Override
    public int getCount() {
        return fragmentList.size();
    }

    @Override
    public CharSequence getPageTitle(int position) {
        return titleList.get(position);
    }
}
```

**UserPokemonList.java (Model)**

The model uses Rx.Java and Rx.Android. It's out of the scope of this post to explain about these concepts.

While this example does not make any API Calls or Database queries, this is the way to go. 

Just don't block or do calculations on the UIThread, but on the IOThread and when done provide the data to UIThread.

```java
package com.nilportugues.simplewebapi.users.interactors;

import com.nilportugues.simplewebapi.shared.interactors.UseCase;

import java.util.ArrayList;
import java.util.List;

import rx.Observable;
import rx.Subscriber;

public class UserPokemonList extends UseCase {

    @Override
    protected Observable buildUseCaseObservable() {
        return Observable.create(
            new Observable.OnSubscribe<List>() {

                @Override
                public void call(Subscriber<? super List> subscriber) {
                    ArrayList<String> pokemon = new ArrayList<>();
                    pokemon.add("Bulbasaur");
                    pokemon.add("Charmander");
                    pokemon.add("Squirtle");

                    subscriber.onNext(pokemon);
                    subscriber.onCompleted(); //!IMPORTANT
                }
            }
        );
    }
}
```


## Model-View-Presenter

After understanding how it works I went for the refactoring to Model-View-Presenter (MVP from now on).  

**Benefits**

- Maximize the amount of code that can be tested with automation.
   - Views are difficult to test, even when testing the happy path and error treatments.
- Separate of concerns: business logic from UI logic remain separated.
   - UI can change without causing side effects.
   - Business logic, represented by the Model can be switched out to for demo or testing purposes as long as the Models implement the original interfaces, neither the Presenter nor the Views needs to change. 

**Downsides**

- More code but way simpler, which is always the case when wirting decoupled code.

## Refactoring

### Step 1: Contract between the Presenter and the View

```java
package com.nilportugues.simplewebapi.users.ui.usetabstext;

import com.nilportugues.simplewebapi.shared.ui.PresenterContract;
import com.nilportugues.simplewebapi.shared.ui.ViewContract;

import java.util.List;

public interface TabsWithTextContract {

    interface Presenter extends PresenterContract {
        void showLoading(boolean active);

        void setList(List<String> userPokemon);

        void showTabs(boolean active);
    }

    interface View extends ViewContract<Presenter> {
        void showLoading();

        void hideLoading();

        void setList(List<String> userPokemon);

        void showTabs();

        void showErroredTabs();
    }
}
```

### Step 2: Moving android.view.View to our new View class.

- View will be handling all classes extending from `android.view.View`. 
- View will call the adapters required to work with the Fragments.

```java
package com.nilportugues.simplewebapi.users.ui.usetabstext;

import android.support.design.widget.TabLayout;
import android.support.v4.app.Fragment;
import android.support.v4.app.FragmentManager;
import android.support.v4.view.ViewPager;
import android.support.v7.widget.Toolbar;
import android.widget.ProgressBar;

import com.nilportugues.simplewebapi.users.ui.userpager.FragmentOne;
import com.nilportugues.simplewebapi.users.ui.userpager.FragmentThree;
import com.nilportugues.simplewebapi.users.ui.userpager.FragmentTwo;

import java.util.ArrayList;
import java.util.List;

public class TabsWithTextView implements TabsWithTextContract.View{

    private FragmentManager mFragmentManager;
    private ViewPager mViewPager;
    private TabLayout mTabLayout;
    private Toolbar mToolbar;
    private ProgressBar mProgressBar;
    private List<Fragment> fragmentList = new ArrayList<>();
    private List<String> titleList = new ArrayList<>();
    private TabsWithTextContract.Presenter mPresenter;
    private List<String> userPokemon;

    public TabsWithTextView(
        FragmentManager fragmentManager,
        ViewPager viewPager,
        TabLayout tabLayout,
        Toolbar toolbar,
        ProgressBar progressBar
    ) {
        mFragmentManager = fragmentManager;
        mViewPager = viewPager;
        mTabLayout = tabLayout;
        mToolbar = toolbar;
        mProgressBar = progressBar;
    }

    @Override
    public void showLoading() {
        mProgressBar.setVisibility(android.view.View.VISIBLE);
    }

    @Override
    public void hideLoading() {
        mProgressBar.setVisibility(android.view.View.GONE);
    }

    @Override
    public void setList(List<String> userPokemon) {
        this.userPokemon = userPokemon;
    }

    @Override
    public void showTabs() {
        buildTabsTitle();
        buildTabsContent();
        buildView();
    }

    @Override
    public void showErroredTabs() {
        mToolbar.setTitle("Could not load tabs");
        buildView();
    }

    @Override
    public void setPresenter(TabsWithTextContract.Presenter presenter) {
        mPresenter = presenter;
    }

    private void buildTabsTitle() {
        mToolbar.setTitle("Text Tabs");
    }

    private void buildTabsContent() {
        this.buildFragment(new FragmentOne(), this.userPokemon.get(0));
        this.buildFragment(new FragmentTwo(), this.userPokemon.get(1));
        this.buildFragment(new FragmentThree(), this.userPokemon.get(2));
    }

    private void buildFragment(Fragment fragment, String title) {
        fragmentList.add(fragment);
        titleList.add(title);
    }

    private void buildView() {
        TabsWithTextAdapter adapter = new TabsWithTextAdapter(mFragmentManager, fragmentList, titleList);
        mViewPager.setAdapter(adapter);
        mTabLayout.setupWithViewPager(mViewPager);
    }
}
```

### Step 3: Moving business logic to our new Presenter class.

- Presenter will handle the logic on how the View behaves based on the business logic provided by a service.
- Presenter gets the View injected. Enables us to use the Visitor pattern. 
- Presenter gets the Model injected to be used. Model remains unchanged. 
- The View will change depending on the Rx.Observable

```java
package com.nilportugues.simplewebapi.users.ui.usetabstext;

import com.nilportugues.simplewebapi.shared.executors.IOThread;
import com.nilportugues.simplewebapi.shared.executors.UIThread;
import com.nilportugues.simplewebapi.users.interactors.UserPokemonList;

import java.util.List;

import rx.Subscriber;

public class TabsWithTextPresenter implements TabsWithTextContract.Presenter
{
    private TabsWithTextContract.View mView;
    private UserPokemonList mInteractor;

    public TabsWithTextPresenter(UserPokemonList interactor, TabsWithTextContract.View view) {
        mView = view;
        mView.setPresenter(this); //!IMPORTANT
        mInteractor = interactor;
    }

    @Override
    public void showLoading(boolean active) {
        if (active) {
            mView.showLoading();
            return;
        }
        mView.hideLoading();
    }

    @Override
    public void setList(List<String> userPokemon) {
        mView.setList(userPokemon);
    }

    @Override
    public void showTabs(boolean active) {
        showLoading(false);
        if (active) {
            mView.showTabs();
            return;
        }
        mView.showErroredTabs();
    }

    @Override
    public void subscribe() {
        showLoading(true);
        mInteractor.execute(new UIThread(), new IOThread(), new Subscriber<List<String>>() {

            @Override
            public void onCompleted() {
                showTabs(true);
            }

            @Override
            public void onError(Throwable e) {
                showTabs(false);
            }

            @Override
            public void onNext(List<String> strings) {
                setList(strings);
            }
        });
    }

    @Override
    public void unsubscribe() {
        mInteractor.unsubscribe();
    }
}
```

### Step 4: Rewriting the TabActivity

```java
package com.nilportugues.simplewebapi.users.ui.usetabstext;

import android.os.Bundle;
import android.support.design.widget.TabLayout;
import android.support.v4.view.ViewPager;
import android.support.v7.app.ActionBar;
import android.support.v7.widget.Toolbar;
import android.widget.ProgressBar;

import com.nilportugues.simplewebapi.R;
import com.nilportugues.simplewebapi.MyApplication;
import com.nilportugues.simplewebapi.users.di.components.UserComponent;
import com.nilportugues.simplewebapi.users.interactors.UserPokemonList;
import com.nilportugues.simplewebapi.users.ui.BaseFragmentActivity;

import javax.inject.Inject;

import butterknife.BindView;

public class TabsWithTextActivity extends BaseFragmentActivity {

    @Inject UserPokemonList interactor;
    @BindView(R.id.tabs1_view_pager) ViewPager viewPager;
    @BindView(R.id.tabs1_layout) TabLayout tabLayout;
    @BindView(R.id.tabs1_toolbar) Toolbar toolbar;
    @BindView(R.id.tabs1_progressbar) ProgressBar progressBar;


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        getComponent().inject(this);
        removeActionBar();
        buildTabs();
    }

    @Override
    protected int getLayoutId() {
        return R.layout.users_tabs_text;
    }
    
    public UserComponent getComponent() {
        return ((MyApplication) getApplication()).getUserComponent();
    }
    
    private void removeActionBar() {
        ActionBar actionBar = getSupportActionBar();
        if (actionBar != null) {
            actionBar.hide();
        }
    }

    private void buildTabs() {
        TabsWithTextView view = new TabsWithTextView(
                getSupportFragmentManager(),
                viewPager,
                tabLayout,
                toolbar,
                progressBar
        );

        TabsWithTextPresenter presenter = new TabsWithTextPresenter(interactor, view);
        presenter.subscribe();
    }
}
```


## Boilerplate code

These have been used in the examples of this post as boilerplate classes to abstract away unnecessary complexity. 

Here's the code for these:

**UseCase.java**

```java
package com.nilportugues.simplewebapi.shared.interactors;

import com.nilportugues.simplewebapi.shared.threads.BackgroundThread;
import com.nilportugues.simplewebapi.shared.threads.PostExecutionThread;

import rx.Observable;
import rx.Subscriber;
import rx.Subscription;
import rx.subscriptions.Subscriptions;

public abstract class UseCase {
    protected Subscription subscription = Subscriptions.empty();

    protected abstract Observable buildUseCaseObservable();

    @SuppressWarnings("unchecked")
    public void execute(
            PostExecutionThread postExecutionThread,
            BackgroundThread backgroundThread,
            Subscriber subscriber
    ) {
        this.subscription = this.buildUseCaseObservable()
                .subscribeOn(backgroundThread.getScheduler())
                .observeOn(postExecutionThread.getScheduler())
                .subscribe(subscriber);
    }

    public void unsubscribe() {
        if (!subscription.isUnsubscribed()) {
            subscription.unsubscribe();
        }
    }
}
```

**BackgroundThread.java**

```java
package com.nilportugues.simplewebapi.shared.threads;

import rx.Scheduler;

public interface BackgroundThread {
    Scheduler getScheduler();
}
```

**IOThread.java**

```java
package com.nilportugues.simplewebapi.shared.executors;

import com.nilportugues.simplewebapi.shared.threads.BackgroundThread;

import rx.Scheduler;
import rx.schedulers.Schedulers;

public class IOThread implements BackgroundThread {
    @Override
    public Scheduler getScheduler() {
        return Schedulers.io();
    }
}
```

**PostExecutionThread.java**

```java
package com.nilportugues.simplewebapi.shared.threads;

import rx.Scheduler;

public interface PostExecutionThread {
    Scheduler getScheduler();
}

```

**UIThread**

```java
package com.nilportugues.simplewebapi.shared.executors;

import com.nilportugues.simplewebapi.shared.threads.BackgroundThread;
import com.nilportugues.simplewebapi.shared.threads.PostExecutionThread;

import rx.Scheduler;
import rx.android.schedulers.AndroidSchedulers;

public class UIThread implements PostExecutionThread
{
    public Scheduler getScheduler() {
        return AndroidSchedulers.mainThread();
    }
}
```

**BaseFragmentActivity.java**

```java
package com.nilportugues.simplewebapi.shared.ui;

import android.os.Bundle;
import android.support.v7.app.AppCompatActivity;

import butterknife.ButterKnife;

public abstract class BaseFragmentActivity extends AppCompatActivity
{
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(getLayoutId());
        ButterKnife.bind(this);
    }

    abstract protected int getLayoutId();
}
```