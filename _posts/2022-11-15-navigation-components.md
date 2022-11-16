---
layout: post
title: Update UI components with NavigationUI
subtitle: Toolbar & Navigation drawer
comments: true
tags: [Android, Navigation]
---
이번 프로젝트에서 만든 안드로이드 어플리케이션은 여러 개의 액티비티, 그리고 각 액티비티마다 여러 개의 프래그먼트로 구성되어있다. Jetpack이 제공하는 Navigation components는 이와 같이 하나의 메인 액티비티와 다수의 프래그먼트로 구성된 앱을 위해 디자인된 것이라고 알려져 있다. 실제로 이를 사용하며 개발을 하면서 느낄 수 있었던 주된 장점은 이러했다. 

- Fragment transactions을 손쉽게 처리
- Up/Back 버튼 클릭 이벤트를 손쉽게 처리
- Navigation drawer나 Bottom navigation과 같은 UI 패턴을 손쉽게 작성할 수 있음
- App bar의 navigation flow와도 연결할 수 있음 
- Navigation editor를 이용하여 navigation flow를 시각적으로 확인할 수 있음

이제 본격적으로 어떻게 navigation components를 프로젝트에 적용했는지 살펴보자. 먼저 다음의 두 의존성을 프로젝트에 추가하였다. 
```java
    implementation "androidx.navigation:navigation-fragment:2.5.3"
    implementation "androidx.navigation:navigation-ui:2.5.3"
```

메인 액티비티에 종속된 다수의 프래그먼트 간의 navigation을 돕는 주된 구성요소는 크게 navigation graph, NavHostFragment, NavController이다. 메인 액티비티는 NavHostFragment를 뷰로 가지고 있고 NavHostFragment는 res 파일에서 정의된 navigation graph와 관련되어 있다. navigation graph는 관련된 fragment destinations과 그들간 연결 고리인 actions로 구성되어 있다. 그리고 그래프에서 정의한 논리적인 네비게이션 플로우대로 실제로 작동하게 해주는 것이 navcontroller이다. 액션이 발생하면 NavHostFragment에 현재 프래그먼트 destination의 띄워진다. 만일 액티비티가 여러 개라면 각각의 액티비티가 자기만의 navigation graph, NavHostFragment와 NavController를 가지게 되는 셈이다. 

## Create navigation components with a ToolBar

**navigation_company.xml**

```xml
<?xml version="1.0" encoding="utf-8"?>
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/navigation_company"
    app:startDestination="@id/companyListFragment">

    <fragment
        android:id="@+id/companyListFragment"
        android:name="com.example.application.company.CompanyListFragment">
        <action
            android:id="@+id/action_companyListFragment_to_companyItemFragment"
            app:destination="@id/companyItemFragment" />
        <action
            android:id="@+id/action_companyListFragment_to_companySearchFragment"
            app:destination="@id/companySearchFragment" />

    </fragment>
    <fragment
        android:id="@+id/companyItemFragment"
        android:name="com.example.application.company.CompanyItemFragment">
        <action
            android:id="@+id/action_companyItemFragment_to_companyReservationFragment"
            app:destination="@id/companyReservationFragment" />
    </fragment>
    <fragment
        android:id="@+id/companyReservationFragment"
        android:name="com.example.application.company.CompanyReservationFragment">
        <action
            android:id="@+id/action_companyReservationFragment_to_companyReservationCheckFragment"
            app:destination="@id/companyReservationCheckFragment" />
    </fragment>
    <fragment
        android:id="@+id/companyReservationCheckFragment"
        android:name="com.example.application.company.CompanyReservationCheckFragment">
        <action
            android:id="@+id/action_companyReservationCheckFragment_to_companyReservationCompletedFragment"
            app:destination="@id/companyReservationCompletedFragment"
            app:popUpTo="@id/companyReservationFragment"
            app:popUpToInclusive="true" />
        <action
            android:id="@+id/action_companyReservationCheckFragment_to_companyReservationFragment"
            app:destination="@id/companyReservationFragment"
            app:popUpTo="@id/companyReservationCheckFragment"
            app:popUpToInclusive="true" />
    </fragment>
    <fragment
        android:id="@+id/companyReservationCompletedFragment"
        android:name="com.example.application.company.CompanyReservationCompletedFragment">
        <action
            android:id="@+id/action_companyReservationCompletedFragment_to_companyItemFragment"
            app:destination="@id/companyItemFragment" />
    </fragment>
    <fragment
        android:id="@+id/companySearchFragment"
        android:name="com.example.application.company.CompanySearchFragment" >
        <action
            android:id="@+id/action_companySearchFragment_to_companyItemFragment"
            app:destination="@id/companyItemFragment"
            app:popUpTo="@id/companySearchFragment"
            app:popUpToInclusive="true" />
    </fragment>
</navigation>
```

![navigation-graph](/assets/img/navigation-flow.png){: .mx-auto.d-block :}
디자인 모드에서 destination을 추가하고 드래그를 통해 arrow로 연결해서 손쉽게 navigation graph를 만들수도 있다.

어플리케이션에서 제공하는 업체 관련 UI 컴포넌트는 CompanyActivity와 여기에 종속된 CompanyListFragment, CompanyItemFragment, CompanyReservationFragment, CompanyReservationCheckFragment, CompanyReservationCompletedFragment이다. 이들 간 네비게이션 플로우에 대한 navigation xml 파일을 보면 일부 액션들에 popUpTo와 popUpToInclusive 옵션이 추가된 것을 볼 수 있다. 예를 들면 CompanySearchFragment에서 CompanyItemFragment로  navigate할 때 액션의 popUpTo를 navigation의 출발점인 CompanySearchFragment, 그리고 popUpTo를 true로 해주었기에 네비게이션 액션 발생 직전 CompanySearchFragment는 백스택에서 삭제된다. 따라서 CompanyItemFragment에서 뒤로가기를 하게 되면 백스택 상단에 있는, 동시에 CompanySearchFragment 바로 밑에 있었던 CompanyListFragment로 이동할 수 있게 된다. 그리고 CompanyReservationCheckFragment에서 CompanyReservationCompletedFragment로 넘어가는 액션의 popUpTo를 CompanyReservationFragment, popUpTo를 true로 해주었기 때문에 예약완료 화면에서 뒤로 간다면 예약 폼을 작성하기 바로 이전의 화면인 예약한 업체 아이템의 화면(CompanyItemFragment)으로 넘어가는 것이다. 

**activity_company.xml**
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <androidx.appcompat.widget.Toolbar
        android:id="@+id/tool_bar"
        android:layout_width="match_parent"
        android:layout_height="?attr/actionBarSize"
        tools:ignore="MissingConstraints">

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="center"
            android:text="@string/d_bugging"
            android:textColor="@color/black"
            android:textSize="16sp"
            android:textStyle="bold" />

        <LinearLayout
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="end"
            android:layout_marginEnd="5dp"
            android:orientation="horizontal">

            <ImageView
                android:id="@+id/tool_bar_search_icon"
                android:layout_width="24dp"
                android:layout_height="24dp"
                android:src="@drawable/ic_search"
                android:visibility="visible"
                app:tint="@color/turquoise_green" />

            <TextView
                android:id="@+id/cancel_search"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:textSize="12sp"
                android:visibility="gone"
                android:textColor="@color/shamrock"
                android:text="@string/cancel_search"
                android:layout_gravity="center"/>

            <ImageView
                android:id="@+id/tool_bar_home_icon"
                android:layout_width="24dp"
                android:layout_height="24dp"
                android:layout_marginHorizontal="10dp"
                android:src="@drawable/ic_home"
                app:tint="@color/turquoise_green" />
        </LinearLayout>

    </androidx.appcompat.widget.Toolbar>

    <View
        android:id="@+id/tool_bar_line"
        android:layout_width="match_parent"
        android:layout_height="1dp"
        android:background="@color/pinkish_grey"
        android:visibility="visible" />

    <androidx.fragment.app.FragmentContainerView
        android:id="@+id/nav_host_fragment_container"
        android:name="androidx.navigation.fragment.NavHostFragment"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:defaultNavHost="true"
        app:layout_constraintTop_toBottomOf="@+id/tool_bar"
        app:navGraph="@navigation/navigation_company" />
</LinearLayout>
```
여기서 FragmentContainerView가 NavHostFragment로서 기능한다고 할 수 있다. 안드로이드에서 제공하는 App bar 위젯으로는 크게 Toolbar, CollapsingToolbarLayout, ActionBar가 있는데 나의 경우 제목과 아이콘과 같은 커스텀 뷰를 추가해주기 위해 Toolbar를 채택하였다. 

Toolbar 역시 나중에 navController와 연결해줄 것이다. 이들을 연결하면 navigation graph에서 정의해놓은대로 top destination을 제외한 나머지에 대해서는 업버튼을 보여주고, top destination의 경우에는 drawer 버튼을 보여준다. 단, drawer 버튼이 시각적으로 보여지고 동작되려면 액티비티의 최상위 뷰가 DrawerLayout이고 appBarConfiguration에서 셋팅을 해줘야 한다. 또한 navigation_graph에서 지정해준 destination의 label에 따라 title을 업데이트해줄 수 있는데, 커스텀 텍스트 뷰를 사용할 것이기 때문에 navigation editor에서 지정된 모든 destination들에 대한 label은 지워주었다.


**CompanyActivity.java**

```java
// ...
    private void setNavController() {
        Toolbar toolBar = binding.toolBar;
        NavHostFragment navHostFragment = (NavHostFragment) getSupportFragmentManager().findFragmentById(R.id.nav_host_fragment_container);
        NavController navController = navHostFragment.getNavController();
        AppBarConfiguration configuration = new AppBarConfiguration.Builder(R.id.companyListFragment, R.id.companyReservationCompletedFragment)
                .setFallbackOnNavigateUpListener(this::onSupportNavigateUp)
                .build();
        NavigationUI.setupWithNavController(toolBar, navController, configuration);
    }
// ...
```
여기서 AppBarConfiguration의 Builder의 인자로 id들을 넘겨주면 해당 destination들은 top destination으로 인식되어 이 destination들 위에서는 업버튼이 보이지 않게 커스터마이징해줄 수 있다. CompanyListFragment의 경우 홈버튼 아이콘이나 백버튼을 클릭하면 HomeActivity로 이동할 수 있게 되어있다. setFallbackOnNavigateUpListener는 더이상 navigate up해서 보여질 프래그먼트가 없을 경우 발생할 수 있는 이벤트에 대한 리스너를 정의해줄 수가 있는데 onSupportNavigateUp 리스너가 백스택 최상단에 있는 HomeActivity로 이동할 수 있게 해준다. 그리고 NavigationUI 클래스의 setupWithNavController 메소드를 통해 toolBar, navController, configuration을 모두 연결해주면 toolBar가 navController의 로직대로, 그리고 configuration의 설정에 따라 이벤트에 반응하게 되는 것이다. 

**CompanyListFragment.java**
```java
// ...
    @Override
    public void navigate(int id) {
        NavController navController = NavHostFragment.findNavController(this);
        Bundle bundle = new Bundle();
        bundle.putInt("companyId", id);
        navController.navigate(R.id.action_companyListFragment_to_companyItemFragment, bundle);
    }
//
```
어떤 fragment에서 다른 fragment로 넘어가기 위해 프래그먼트 내부에서 NavController 객체의 navigate(액션 아이디, bundle)을 호출해준다. 이때 type-safe navigation을 보장하는 Safe Args라고 불리는 Gradle 플러그인을 사용하여 data를 전달해도 되지만 나의 경우 Bundle을 이용해 직접 데이터를 전달하는 식으로 구현하였다.

여기까지 정리하면 하나의 메인 액티비티와 다수의 프래그먼트, 이 한 묶음에 대해 NavHostFragment, NavController, navigation_graph를 생성할 수 있고  이러한 navigation 로직과 configuration을 ToolBar에 적용시켜 업버튼 이벤트를 손쉽게 처리할 수 있었다. 

## Add a navigation drawer
로그인에 성공하면 시작되는 HomeActivity는 Navigation Drawer를 포함하고 있다. Navigation Drawer는 앱바 왼쪽에 있는 drawer 아이콘을 클릭하거나 왼쪽에서 오른쪽으로 스와이프할 때 볼 수 있다. 이는 메뉴를 포함하고 있는데, 어플리케이션에서 별도의 컨테이너 없이 메뉴만 생성하는 경우, 액티비티나 프래그먼트의 onCreateOptionsMenu(Menu menu)와 onOptionsItemSelected(MenuItem item) 메소드를 오버라이딩해서 구현하면 UI 컴포넌트가 시작할 때 onCreateOptionsMenu가 호출되고 메뉴 아이템이 클릭될 때마다 onOptionsItemSelected가 호출되는 것으로 알려져있다. 만일 onOptionsItemSelected 메소드 내부에서 NavigationUI.onNavDestinationsSelected(item, navController)를 통해 메뉴 아이템의 아이디와 navController와 연결된 nav_graph의 어떤 destination의 id가 같다면 해당 메뉴 아이템 클릭시 그 destination으로 이동할 수 있다. 결국 navigation 로직을 각각의 메뉴 아이템에도 적용시킬 수 있다는 의미이다.

그러나 여기서는 Navigation drawer가 fragments가 아닌 activities 간의 이동을 위한 메뉴로 구성이 되어있기 때문에 이를 위한 navigation component을 이용한 로직을 연결해주지 않았고 대신에 메뉴 아이템 클릭시 Intent를 통해 다른 액티비티로 넘어가도록 해주었다. 

**main_menu.xml**
```xml
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android">

    <item
        android:id="@+id/myPageActivity"
        android:title="@string/my_page" />

    <item
        android:id="@+id/surveyActivity"
        android:title="@string/bug_survey" />

    <item
        android:id="@+id/companyActivity"
        android:title="@string/company_connection" />

    <item
        android:id="@+id/productActivity"
        android:title="@string/self_kill_explanation" />

    <item
        android:id="@+id/bugActivity"
        android:title="@string/bug_explanation" />
</menu>
```

**main_menu_header.xml**
```java
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:fitsSystemWindows="true"
    android:orientation="vertical">

    <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="180dp">

        <ImageView
            android:id="@+id/profile_image_icon"
            android:layout_width="90dp"
            android:layout_height="90dp"
            android:layout_marginStart="24dp"
            android:background="@drawable/oval_r80"
            android:backgroundTint="@color/marigold"
            android:padding="20dp"
            android:src="@drawable/ic_person"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toTopOf="parent"
            app:tint="@color/white" />

        <TextView
            android:id="@+id/tv_hello"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginStart="20dp"
            android:text="안녕하세요"
            android:textColor="@color/brown_grey"
            android:textSize="14sp"
            app:layout_constraintBottom_toTopOf="@id/tv_profile_name"
            app:layout_constraintStart_toEndOf="@id/profile_image_icon"
            app:layout_constraintTop_toTopOf="@id/profile_image_icon" />

        <TextView
            android:id="@+id/tv_profile_name"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="김눈송 님"
            android:textColor="@color/brown_grey"
            android:textSize="22sp"
            android:textStyle="bold"
            app:layout_constraintBottom_toTopOf="@id/tv_num_of_acccumulated_usages"
            app:layout_constraintStart_toStartOf="@id/tv_hello"
            app:layout_constraintTop_toBottomOf="@id/tv_hello" />

        <TextView
            android:id="@+id/tv_num_of_acccumulated_usages"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="누적 업체 이용 2건"
            android:textColor="@color/brown_grey"
            android:textSize="16sp"
            android:textStyle="bold"
            app:layout_constraintBottom_toBottomOf="@id/profile_image_icon"
            app:layout_constraintStart_toStartOf="@id/tv_hello"
            app:layout_constraintTop_toBottomOf="@id/tv_profile_name" />


    </androidx.constraintlayout.widget.ConstraintLayout>

    <View
        android:layout_width="match_parent"
        android:layout_height="4dp"
        android:background="@color/white_grey" />

</LinearLayout>
```

**activity_home.xml**
```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.drawerlayout.widget.DrawerLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/drawer_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

        <androidx.appcompat.widget.Toolbar
            android:id="@+id/tool_bar"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize">

            <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_gravity="center"
                android:text="@string/main_title"
                android:textColor="@color/brown_grey"
                android:textSize="16sp"
                android:textStyle="bold" />

            <LinearLayout
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_gravity="end"
                android:layout_marginEnd="5dp"
                android:orientation="horizontal">

                <ImageView
                    android:id="@+id/my_page_icon"
                    android:layout_width="24dp"
                    android:layout_height="24dp"
                    android:src="@drawable/ic_user" />

                <ImageView
                    android:id="@+id/notification_icon"
                    android:layout_width="24dp"
                    android:layout_height="24dp"
                    android:layout_marginHorizontal="10dp"
                    android:src="@drawable/ic_notification" />
            </LinearLayout>

        </androidx.appcompat.widget.Toolbar>

        <View
            android:layout_width="match_parent"
            android:layout_height="1dp"
            android:background="@color/white_grey" />

        <androidx.fragment.app.FragmentContainerView
            android:id="@+id/nav_host_fragment_container"
            android:name="androidx.navigation.fragment.NavHostFragment"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            app:defaultNavHost="true"
            app:navGraph="@navigation/navigation_home" />

    </LinearLayout>

    <com.google.android.material.navigation.NavigationView
        android:id="@+id/nav_view"
        android:layout_width="300dp"
        android:layout_height="match_parent"
        android:layout_gravity="start"
        app:itemTextAppearance="@style/menu_text_style"
        app:itemTextColor="@color/brown_grey"
        app:menu="@menu/main_menu">

        <androidx.constraintlayout.widget.ConstraintLayout
            android:id="@+id/option_container"
            android:layout_width="match_parent"
            android:layout_height="match_parent">

            <LinearLayout
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:orientation="horizontal"
                app:layout_constraintBottom_toBottomOf="@id/option_container"
                app:layout_constraintEnd_toEndOf="@id/option_container">

                <TextView
                    android:id="@+id/tv_log_out"
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:layout_marginBottom="16dp"
                    android:drawablePadding="6dp"
                    android:text="@string/log_out"
                    android:textColor="@color/turquoise_green"
                    android:textSize="16sp"
                    android:textStyle="bold"
                    app:drawableStartCompat="@drawable/ic_log_out"
                    app:drawableTint="@color/turquoise_green" />

                <TextView
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:layout_marginHorizontal="6dp"
                    android:layout_marginBottom="16dp"
                    android:text="@string/divider"
                    android:textColor="@color/pinkish_grey"
                    android:textSize="16sp" />

                <TextView
                    android:id="@+id/tv_sign_out"
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:layout_marginEnd="16dp"
                    android:layout_marginBottom="16dp"
                    android:text="@string/sign_out"
                    android:textColor="@color/turquoise_green"
                    android:textSize="16sp"
                    android:textStyle="bold" />

            </LinearLayout>
        </androidx.constraintlayout.widget.ConstraintLayout>

    </com.google.android.material.navigation.NavigationView>

</androidx.drawerlayout.widget.DrawerLayout>
```
Navigation Drawer이 잘 동작하려면 우선 액티비티의 최상단 루트뷰가 DrawerLayout 이어야한다. 이는 메뉴가 들어가는 NavigationView를 포함하고 있다.
또한 메뉴의 하단에 보여질 로그아웃과 회원탈퇴를 할 수 있는 텍스트 뷰를 배치해주었다. 


**HomeActivity.java**
```java
    private void setNavController() {
        Toolbar toolBar = binding.toolBar;
        NavHostFragment navHostFragment = (NavHostFragment) getSupportFragmentManager().findFragmentById(R.id.nav_host_fragment_container);
        NavController navController = navHostFragment.getNavController();
        AppBarConfiguration configuration = new AppBarConfiguration.Builder(navController.getGraph())
                .setOpenableLayout(drawerLayout)
                .build();
        NavigationUI.setupWithNavController(toolBar, navController, configuration);
    }

    private void setNavigationDrawer() {
        NavigationView navView = binding.navView;
        View header = navView.inflateHeaderView(R.layout.main_menu_header);
        ((TextView) header.findViewById(R.id.tv_profile_name)).setText("김다솔 님");

        LinearLayout.LayoutParams params = new LinearLayout.LayoutParams(navView.getLayoutParams().width, ViewGroup.LayoutParams.WRAP_CONTENT);
        binding.navView.getHeaderView(0).setLayoutParams(params);
    }

    private void bindViews() {
        binding.navView.setNavigationItemSelectedListener(this::handleMenuItemClick);
        // ...
    }

    private boolean handleMenuItemClick(MenuItem item) {
        switch (item.getItemId()) {
            case (R.id.bugActivity):
                startActivity(new Intent(this, BugActivity.class));
                finish();
                return true;
            case (R.id.companyActivity):
                startActivity(new Intent(this, CompanyActivity.class));
                finish();
                return true;
            case (R.id.productActivity):
                startActivity(new Intent(this, ProductActivity.class));
                finish();
                return true;
            case (R.id.myPageActivity):
                startActivity(new Intent(this, MyPageActivity.class));
                return true;
            default:
                return false;
        }
    }
```
HomeActivity에서 위에서 생성한 커스텀 헤더 레이아웃을 inflate해주었고 메뉴 아이템의 클릭 이벤트 리스너를 바인딩해주었다. Navigation Drawer에 종속된 메뉴에 대한 아이템 클릭 리스너는 이와 같이 setNavigationItemSelectedListener 메소드를 통해 설정해주고 마찬가지로 리턴 타입은 boolean이어야 한다. 또한 툴바의 configuration과 drawerLayout을 연결해주어야 HomeActivity의 툴바에서 drawer icon을 확인할 수 있다. 

지금까지 Navigation Component들을 이용하여 네비게이션 로직을 작성하고 이를 Toolbar에도 적용시켜보았다. navigation graph의 옵션을 활용하여 액션 발생 시 백스택을 원하는 로직대로 관리해 Toolbar의 업버튼 이벤트 결과에 영향을 줄 수 있었다. Navigation Drawer 내부의 메뉴 역시 이러한 navigation component 로직의 적용을 받을 수 있지만 그렇게 해주지 않고 NavigationView의 setter를 통해 별도로 클릭 이벤트 리스너를 추가해주었다. 뿐만 아니라 Navigation Drawer에 커스텀 헤더뷰를 inflate 함으로써 현재 접속한 유저의 정보를 간단하게 보여주는 로직까지 작성해보았다. 
