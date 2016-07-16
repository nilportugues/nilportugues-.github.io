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

## MVP
After understanding how it works I went for the refactoring to MVP.  

**Benefits**
- Maximize the amount of code that can be tested with automation. (Views are difficult to test.)
- Separate business logic from UI logic to make the code easier to understand and maintain.

**Downsides**
- More code, whcih is always the case when wirting decoupled code.

## Refactoring

**View responsabilities**

- View will be handling all classes extending from `android.view.View`. 
- View will call the adapters required to work with the Fragments.

**Presenter responsabilities**

- Presenter will handle the logic on how the View behaves based on the business logic provided by a service.
- Presenter gets the Model injected to be used. 

**Model responsabilities**

- Model retrieves data from a API or Database. We will fake this because it's out of the scope of this post.
- Note: Model is a Service in our case, which we named `interactor`.

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

```
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

    public TabsWithTextView(FragmentManager fragmentManager, ViewPager viewPager, TabLayout tabLayout, Toolbar toolbar, ProgressBar progressBar) {
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

    protected void buildTabsTitle() {
        mToolbar.setTitle("Text Tabs");
    }

    protected void buildTabsContent() {
        this.buildFragment(new FragmentOne(), this.userPokemon.get(0));
        this.buildFragment(new FragmentTwo(), this.userPokemon.get(1));
        this.buildFragment(new FragmentThree(), this.userPokemon.get(2));
    }

    protected void buildFragment(Fragment fragment, String title) {
        fragmentList.add(fragment);
        titleList.add(title);
    }

    protected void buildView() {
        TabsWithTextAdapter adapter = new TabsWithTextAdapter(mFragmentManager, fragmentList, titleList);
        mViewPager.setAdapter(adapter);
        mTabLayout.setupWithViewPager(mViewPager);
    }
}
```

### Step 3: Moving business logic to our new Presenter class.

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
        mView.setPresenter(this);
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

### Step 4: The Model

### Step 5: Rewriting the TabActivity

```java
package com.nilportugues.simplewebapi.users.ui.usetabstext;


import android.os.Bundle;
import android.support.design.widget.TabLayout;
import android.support.v4.view.ViewPager;
import android.support.v7.app.ActionBar;
import android.support.v7.widget.Toolbar;
import android.widget.ProgressBar;

import com.nilportugues.simplewebapi.R;
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

    protected void removeActionBar() {
        ActionBar actionBar = getSupportActionBar();
        if (actionBar != null) {
            actionBar.hide();
        }
    }

    protected void buildTabs() {
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