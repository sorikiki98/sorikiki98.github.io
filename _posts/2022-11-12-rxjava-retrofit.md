---
layout: post
title: RxJava with Retrofit
subtitle: RxJava, Reactive Extensions for the JVM
comments: true
tags: [Android, RxJava, Retrofit]
---

코드를 리팩토링하기 전 서버에 http 요청을 하는 Retrofit 통신의 결과를 비동기적으로 받아와(enqueue) 성공에 대한 콜백 그리고 실패에 대한 콜백을 모두 받아와야 했다. 이렇게 콜백을 이용하는 것의 단점은 가독성이 현저히 떨어지고 별개의 http 요청에 대한 응답 결과를 결합하기 어렵다는 점이다. 예를 들면 '업체 목록'을 불러올 때 로컬 DB의 목록이 비어있다면 sequentially하게 HTTP GET 요청을 해서 원격 DB에 저장된 데이터를 불러와야 하기에 chaining 처리가 필수적이다.

나는 안드로이드 플랫폼에서 주로 사용되는 비동기 처리 라이브러리인 RxJava나 Kotlin Coroutine에 대한 api 문서를 정독해보고 이 중 RxJava를 먼저 이번 프로젝트에 적용해보았다. 자바 8부터 지원되는 CompletableFuture의 경우 chaining api를 통해 combined computations, error handling 및 blocking protection 등의 기능을 통해 Future 인터페이스의 문제점들을 해소해주었지만 안스에서 사용하기는 적절치 않았다. 왜냐하면 I/O 작업을 백그라운드 스레드에서 실행하고 메인 UI 로직과 관련된 부분을 메인 스레드에서 별도로 실행해주어야 하는데 처리가 상당히 까다롭기 때문이다. 이번 포스팅에서는 RxJava에서 제공하는 api를 어떻게 프로젝트에 적용했는 지 코드를 다시 한번 분석해보겠다.

우선 이 프로젝트에서는 다음의 의존성들을 필요로 한다. 
```java
 // retrofit
    implementation "com.squareup.retrofit2:retrofit:2.9.0"
    implementation "com.squareup.retrofit2:converter-gson:2.9.0"
    implementation "com.squareup.retrofit2:adapter-rxjava3:2.9.0"
    implementation "com.squareup.okhttp3:okhttp:4.10.0"
    implementation "com.squareup.okhttp3:logging-interceptor:4.10.0"

// rxjava3
    implementation "io.reactivex.rxjava3:rxandroid:3.0.0"
    implementation "io.reactivex.rxjava3:rxjava:3.1.5"
```
rxandroid 라이브러리는 메인스레드용 스케줄러인 AndroidSchedulers.mainThread() api를 지원해준다. Retrofit Service의 인터페이스의 반환 타입을 더 이상 Call이 아닌 Rxjava에서 지원하는 Observables(Single, Maybe, Completable, Observable, Flowable) 타입으로 mapping 해주기 위해서는 rxjava3용 retrofit 어댑터도 필요하다. 

## MVP Architecture with RxJava ⭐

![MVP Architecture](https://github.com/googlesamples/android-architecture/wiki/images/mvp.png){: .mx-auto.d-block :}
프로젝트에서 사용한 아키텍처의 큰 구조는 위와 같다. RemoteDataSource에는 Retrofit 서비스에 접근하는 로직을 작성해주고 LocalDataSource에는 Room Dao과 인터랙션하는 로직을 작성해준다. 이렇게 하면 Repository에서 RemoteDatSource 혹은 LocalDataSource에 데이터를 read/write하고 데이터 로딩 이후 어플리케이션 메모리에 캐싱해주는 작업을 진행한다. 이후 Presenter에는 Repository로부터 연쇄되어 반환된 Observable을 구독하여 메인 스레드 하에서 어떤 View를 Fragment에 display할지 로직을 작성해준다. 이러한 로직을 작성하는데 안드로이드 팀에서 제시한 [architecture-samples 깃헙 레포지토리의 todo-mvp-rxjava 브랜치의 readme](https://github.com/android/architecture-samples/tree/todo-mvp-rxjava)를 참고했기에 큰 도움이 되었다.


## Observables을 반환하는 API들의 집합 Interface 정의(local, remote)

**CompanyDao.java**
```java
@Dao
public interface CompanyDao {
    @Query("SELECT * FROM companies")
    public Flowable<List<Company>> getCompanies();

    @Query("SELECT * FROM companies WHERE id=:companyId")
    public Flowable<Optional<Company>> getCompany(int companyId);

    @Query("UPDATE companies SET isCompanyInterested=1, numOfInterestedUsers=numOfInterestedUsers+1 WHERE id=:companyId")
    public Completable addCompanyInterest(int companyId);

    @Query("UPDATE companies SET isCompanyInterested=0, numOfInterestedUsers=numOfInterestedUsers-1 WHERE id=:companyId")
    public Completable removeCompanyInterest(int companyId);

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    public void insertCompanies(List<Company> companies);

}
```
로컬 db인 SQLite의 abstract layer인 Room 라이브러리를 사용하기 위해 이에 필요한 Data Access Object 인터페이스를 정의해주었다. 

SQLite에 비동기적인 쿼리를 보내는 Observable 혹은 Flowable에 대해 구독하기 시작하면 쿼리가 참조하는 테이블의 어떤 부분이라도 변경될 때마다 쿼리를 재실행하여 업데이트된 값을 emit한다. 어플리케이션 로직상 반드시 로컬 companies 테이블의 observable을 구독하고 있는 상태이므로 찜하기 toggle을 할 때마다 로컬 db가 업데이트되므로 Flowable을 구독하는 downstream 함수들이 모두 호출되어 최종적으로 UI에도 반영될 수 있는 것이다. 이렇게 RXJava는 observable queries에 대한 처리를 용이하게 해주는 Flowable, Publisher, Observable을 지원해 스트림 형식으로 구현된 코드를 통해 변경 사항을 UI에 시시각각 반영해주는 장점이 있다. 그러나 one shot queries, 즉 db operation을 실행하고 snapshot을 가져와 단 한번 emit하는 작업(예를 들면 서버에서 어떤 작업을 완료하는 것을 기다리는 작업) 역시 스트림으로 작성해야 한다는 점은 chaining이 불필요하게 삽입되어 가독성을 떨어뜨릴 수 있다는 것을 감안해야 한다. 

참고 사이트: [Write asynchronous DAO queries](https://developer.android.com/training/data-storage/room/async-queries)

**CompanyService.java**
```java
public interface CompanyService {
    @GET("companies")
    Flowable<List<Company>> getCompanies();

    @POST("companies/interest/{company_id}")
    Completable addCompanyInterest(@Path("company_id") int companyId);

    @DELETE("companies/interest/{company_id}")
    Completable removeCompanyInterest(@Path("company_id") int companyId);

    @POST("companies/reservation/{company_id}")
    Maybe<Integer> reserveCompany(@Path("company_id") int companyId, @Body ReservationForm reservationForm);

    @GET("companies/reservation/{reservation_id}")
    Maybe<Reservation> getReservationInformation(@Path("reservation_id") int reservationId);
}
```
MySQL에 저장된 companies 테이블에 접근하는 Retrofit 통신을 위한 API를 정의한 인터페이스이다. 각 요청마다 헤더에 들어가는 Bearer 토큰 정보는 Retrofit을 빌드하는 과정에서 따로 interceptor로 처리해주었으므로 인터페이스 내부 각 API들에 따로 파라미터를 추가해줄 필요없다. 이는 다음 포스팅에서 자세히 다룰 예정이다. 사실상 서버에 요청해서 응답을 받아오는 것은 모두 one shot이라고 할 수 있기에 getCompanies 메소드의 리턴 타입을 Flowable로 사용한 것은 의미가 없는 듯하다.

## Repository 인터페이스 및 구현체

```java
public interface CompanyRepository {
    Flowable<List<Company>> getCompanies(boolean isFirstLoad);

    Flowable<Optional<Company>> getCompanyWithId(int companyId);

    Completable addCompanyInterest(int companyId);

    Completable removeCompanyInterest(int companyId);

    Maybe<Integer> reserveCompany(int companyId);

    Maybe<Reservation> getReservationInformation(int reservationId);

    ReservationForm getReservationForm();

    List<Company> filterCompanies(String keyword);

    boolean isCompanyInterested(int companyId);

    void keepReservationForm(ReservationForm reservationForm);

    void refreshCache(List<Company> companies);

    void refreshLocalDataSource(List<Company> companies);
}
```


```java
@Singleton
public class CompanyRepositoryImpl implements CompanyRepository {
    private HashMap<Integer, Company> cachedCompanies = new LinkedHashMap<>();

    private ReservationForm reservationForm;

    private boolean isCacheDirty = false;

    private final CompanyRemoteDataSource companyRemoteDataSource;

    private final CompanyLocalDataSource companyLocalDataSource;


    @Inject
    CompanyRepositoryImpl(CompanyRemoteDataSource companyRemoteDataSource, CompanyLocalDataSource companyLocalDataSource) {
        this.companyRemoteDataSource = companyRemoteDataSource;
        this.companyLocalDataSource = companyLocalDataSource;
    }


    @Override
    public Flowable<List<Company>> getCompanies(boolean isFirstLoad) {
        if (!isCacheDirty && !cachedCompanies.isEmpty() && isFirstLoad) {
            return Flowable.just(new ArrayList<>(cachedCompanies.values()));
        }

        // 새로고침
        if (!isFirstLoad) {
            return getCompaniesFromRemoteDataSource();
        }

        return companyLocalDataSource.getCompanies()
                .flatMap((List<Company> companies) -> {
                    if (companies.isEmpty()) {
                        return getCompaniesFromRemoteDataSource();
                    }
                    refreshCache(companies);
                    isCacheDirty = false;
                    return Flowable.just(companies);
                });
    }

    @Override
    public Flowable<Optional<Company>> getCompanyWithId(int companyId) {
        if (!isCacheDirty && !cachedCompanies.isEmpty()) {
            Optional<Company> company = Optional.ofNullable(cachedCompanies.get(companyId));
            return Flowable.just(company);
        }

        return companyLocalDataSource.getCompanyWithId(companyId);
    }

    @Override
    public Completable addCompanyInterest(int companyId) {
        isCacheDirty = true;
        return companyRemoteDataSource.addCompanyInterest(companyId)
                .andThen(companyLocalDataSource.addCompanyInterest(companyId));

    }

    @Override
    public Completable removeCompanyInterest(int companyId) {
        isCacheDirty = true;
        return companyRemoteDataSource.removeCompanyInterest(companyId)
                .andThen(companyLocalDataSource.removeCompanyInterest(companyId));
    }

    @Override
    public Maybe<Integer> reserveCompany(int companyId) {
        return companyRemoteDataSource.reserveCompany(companyId, reservationForm)
                .doAfterSuccess((reservationId) -> reservationForm = null);
    }

    @Override
    public Maybe<Reservation> getReservationInformation(int reservationId) {
        return companyRemoteDataSource.getReservationInformation(reservationId);
    }

    @Override
    public List<Company> filterCompanies(String keyword) {
        List<Company> companies = new ArrayList<>(cachedCompanies.values());
        List<Company> filteredCompanies = new ArrayList<>();
        for (Company company: companies) {
            if (company.name.contains(keyword)) {
                filteredCompanies.add(company);
            }
        }
        return filteredCompanies;
    }

    @Override
    public ReservationForm getReservationForm() {
        return this.reservationForm;
    }

    @Override
    public boolean isCompanyInterested(int companyId) {
        if (cachedCompanies != null && cachedCompanies.get(companyId) != null) {
            int isCompanyInterested = Objects.requireNonNull(cachedCompanies.get(companyId)).isCompanyInterested;
            return isCompanyInterested == 1;
        }
        return false;
    }

    @Override
    public void keepReservationForm(ReservationForm reservationForm) {
        this.reservationForm = reservationForm;
    }

    @Override
    public void refreshCache(List<Company> companies) {
        cachedCompanies = new LinkedHashMap<>();
        for (Company company : companies) {
            cachedCompanies.put(company.id, company);
        }
    }

    @Override
    public void refreshLocalDataSource(List<Company> companies) {
        companyLocalDataSource.insertCompanies(companies);
    }

    private Flowable<List<Company>> getCompaniesFromRemoteDataSource() {
        return companyRemoteDataSource.getCompanies()
                .flatMap((List<Company> companies) -> {
                    refreshLocalDataSource(companies);
                    refreshCache(companies);
                    isCacheDirty = false;
                    return Flowable.just(companies);
                });
    }
}
```
업체 목록 로딩(**read**) 시, 보존된 캐시 데이터가 있으면 이를 리턴한다. 어플리케이션 유저가 새로고침을 하면 http 요청을 해서 얻어온 결과를 리턴한다. remoteDataSource에서 get 해올 때에는 비동기를 로직을 처리할 수 있는 변환 연산자인 flatMap api를 이용하여 로컬 DB에 같은 데이터를 insert하고 메모리에 캐싱한 후 그대로 받아온 데이터를 리턴해준다. 위의 두 가지 상황이 아닌 경우 로컬 db에 데이터가 존재하는지 확인하고 존재한다면 localDataSource의 데이터를 불러온 후 캐싱을 해준다. 존재하지 않는다면 remoteDataSource로부터 불러온다. 

업체 찜하기(**write**) 상태를 변경해줄 때에는 localDataSource와 remoteDataSource 모두에 업데이트해준다. 이렇게 하면 로컬 데이터를 로딩한 업체 목록 화면에서 변경사항이 있을 때마다 UI의 아이콘 색상이 변경된다. 다만, 캐싱 데이터를 로딩한 경우에는 자동적으로 업데이트가 적용이 되지 않아 Presenter 단에서 이벤트가 완료되면 업체 목록을 재로딩해주는 식으로 구현해야 한다.  

## DataSource 인터페이스 정의 및 local, remote용 구현체 생성

**CompanyDataSource.java**
```java
public interface CompanyDataSource {
    public Flowable<List<Company>> getCompanies();

    public Flowable<Optional<Company>> getCompanyWithId(int companyId);

    public Completable addCompanyInterest(int companyId);

    public Completable removeCompanyInterest(int companyId);

    public Maybe<Integer> reserveCompany(int companyId, ReservationForm reservationForm);

    public Maybe<Reservation> getReservationInformation(int reservationId);

    public void insertCompanies(List<Company> companies);
}
```

**CompanyLocalDataSource**
```java
public class CompanyLocalDataSource implements CompanyDataSource {
    private final CompanyDao companyDao;
    private final Scheduler ioScheduler;

    @Inject
    public CompanyLocalDataSource(CompanyDao companyDao, SchedulersFacade schedulersFacade) {
        this.companyDao = companyDao;
        this.ioScheduler = schedulersFacade.io();
    }

    @Override
    public Flowable<List<Company>> getCompanies() {
        return companyDao.getCompanies()
                .subscribeOn(ioScheduler);
    }

    @Override
    public Flowable<Optional<Company>> getCompanyWithId(int companyId) {
        return companyDao.getCompany(companyId)
                .subscribeOn(ioScheduler);
    }

    @Override
    public Completable addCompanyInterest(int companyId) {
        return companyDao.addCompanyInterest(companyId)
                .subscribeOn(ioScheduler);
    }

    @Override
    public Completable removeCompanyInterest(int companyId) {
        return companyDao.removeCompanyInterest(companyId)
                .subscribeOn(ioScheduler);
    }

    @Override
    public Maybe<Integer> reserveCompany(int companyId, ReservationForm reservationForm) {
        return null;
    }

    @Override
    public Maybe<Reservation> getReservationInformation(int reservationId) {
        return null;
    }

    @Override
    public void insertCompanies(List<Company> companies) {
        companyDao.insertCompanies(companies);
    }
}
```

**CompanyRemoteDataSource.java**
```java
public class CompanyRemoteDataSource implements CompanyDataSource {
    private final CompanyService companyService;
    private final Scheduler ioScheduler;

    @Inject
    public CompanyRemoteDataSource(CompanyService companyService, SchedulersFacade schedulersFacade) {
        this.companyService = companyService;
        this.ioScheduler = schedulersFacade.io();
    }

    @Override
    public Flowable<List<Company>> getCompanies() {
        return companyService.getCompanies()
                .subscribeOn(ioScheduler);
    }

    @Override
    public Flowable<Optional<Company>> getCompanyWithId(int companyId) {
        return null;
    }

    @Override
    public Completable addCompanyInterest(int companyId) {
        return companyService.addCompanyInterest(companyId)
                .subscribeOn(ioScheduler);
    }

    @Override
    public Completable removeCompanyInterest(int companyId) {
        return companyService.removeCompanyInterest(companyId)
                .subscribeOn(ioScheduler);
    }

    @Override
    public Maybe<Integer> reserveCompany(int companyId, ReservationForm reservationForm) {
        return companyService.reserveCompany(companyId, reservationForm)
                .subscribeOn(ioScheduler);
    }

    @Override
    public Maybe<Reservation> getReservationInformation(int reservationId) {
        return companyService.getReservationInformation(reservationId)
                .subscribeOn(ioScheduler);
    }

    @Override
    public void insertCompanies(List<Company> companies) {

    }
}
```
Observable 데이터인 업체 목록과 업체 아이템을 불러오는 api의 리턴 타입은 Flowable, 업체 찜하기 관련 api는 Completable, 예약 성공 후 아이디를 받아오고 예약 정보를 불러오는 api는 Maybe 타입으로 구분해주었다. 

## Presenter와 View 

**CompanyListContract.java**

```java
public interface CompanyListContract {
    interface Presenter extends BasePresenter {
        void getCompanies();

        void refreshCompanies();

        void toggleCompanyInterest(int companyId);
    }

    interface View extends BaseView {
        void showCompanies(List<Company> companies);

        void showErrorMessage(String message);

        void undoRefreshLoading();
    }
}
```
위 인터페이스에 Presenter와 View 간 인터랙션을 위한 api들을 정의해두었다. 

**CompanyListPresenter.java**
```java
public class CompanyListPresenter implements CompanyListContract.Presenter {
    private final CompanyRepository companyRepository;
    private final CompanyListContract.View view;
    private final Scheduler mainScheduler;

    private Subscription subscription;

    private final CompositeDisposable compositeDisposable = new CompositeDisposable();

    @Inject
    public CompanyListPresenter(CompanyRepository companyRepository, CompanyListContract.View view, Scheduler scheduler) {
        this.companyRepository = companyRepository;
        this.view = view;
        this.mainScheduler = scheduler;
    }

    @Override
    public void subscribe() {
        getCompanies();
    }

    @Override
    public void unsubscribe() {
        subscription.cancel();
        compositeDisposable.clear();
    }

    @Override
    public void getCompanies() {
        loadCompanies(true);
    }

    @Override
    public void refreshCompanies() {
        loadCompanies(false);
    }

    @Override
    public void toggleCompanyInterest(int companyId) {
        boolean isCompanyInterested = companyRepository.isCompanyInterested(companyId);
        if (isCompanyInterested) removeCompanyInterest(companyId);
        else addCompanyInterest(companyId);
    }

    private void addCompanyInterest(int companyId) {
        companyRepository.addCompanyInterest(companyId)
                .subscribe(getCompanyInterestCompletableObserver());
    }

    private void removeCompanyInterest(int companyId) {
        companyRepository.removeCompanyInterest(companyId)
                .subscribe(getCompanyInterestCompletableObserver());
    }

    private CompletableObserver getCompanyInterestCompletableObserver() {
        return new CompletableObserver() {
            @Override
            public void onSubscribe(@NonNull Disposable d) {
                compositeDisposable.add(d);
            }

            @Override
            public void onComplete() {
                subscribe();
            }

            @Override
            public void onError(@NonNull Throwable e) {
                if (e instanceof SocketTimeoutException) {
                    view.showErrorMessage("서버와 연결하는 데 실패했습니다.(Connection failed)");
                }
                else if (e instanceof HttpException) {
                    if (((HttpException) e).code() == 404) {
                        view.showErrorMessage("찜하기 해제 실패");
                    } else {
                        view.showErrorMessage("찜하기 실패");
                    }
                }
                else view.showErrorMessage("알 수 없는 오류");
            }
        };
    }

    private void loadCompanies(boolean isFirstLoad) {
        companyRepository.getCompanies(isFirstLoad)
                .observeOn(mainScheduler)
                .subscribe(new FlowableSubscriber<List<Company>>() {
                    @Override
                    public void onSubscribe(@NonNull Subscription s) {
                        s.request(Long.MAX_VALUE);
                        subscription = s;
                    }

                    @Override
                    public void onNext(List<Company> companies) {
                        view.showCompanies(companies);
                        view.undoRefreshLoading();
                    }

                    @Override
                    public void onError(Throwable t) {
                        view.showErrorMessage("목록을 불러올 수 없습니다.");
                        view.undoRefreshLoading();
                    }

                    @Override
                    public void onComplete() {
                        view.undoRefreshLoading();
                    }
                });
    }
}
```
앞서 설명했듯 캐싱된 데이터를 로딩하는 경우는 이벤트의 결과가 반영이 안된다. 그래서 onComplete 콜백함수에서 업체 목록을 재로딩해주는 식으로 구현하였다. I/O 요청이 성공하는 경우와 실패하는 경우 모두 Fragment에서 어떤 뷰를 display해주어야 하는지에 대한 로직을 작성해주었다.  

**CompanyListFragment.java**

```java
public class CompanyListFragment extends Fragment implements CompanyListContract.View {
    @Inject
    CompanyListContract.Presenter presenter;

    private FragmentCompanyListBinding binding;

    private final CompanyListAdapter companyListAdapter = new CompanyListAdapter(
            new CompanyDiffCallback(),
            (companyId) -> {
                navigate(companyId);
                return null;
            },
            (companyId) -> {
                presenter.toggleCompanyInterest(companyId);
                return null;
            }
    );

    @Override
    public void onAttach(@NonNull Context context) {
        AndroidSupportInjection.inject(this);
        super.onAttach(context);
    }

    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        super.onCreateView(inflater, container, savedInstanceState);
        binding = FragmentCompanyListBinding.inflate(inflater, container, false);
        return binding.getRoot();
    }

    @Override
    public void onViewCreated(@NonNull View view, @Nullable Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);

        initViews();
        bindViews();
    }

    @Override
    public void onResume() {
        super.onResume();
        presenter.subscribe();
    }

    private void initViews() {
        binding.rvCompanyList.setAdapter(companyListAdapter);
        binding.rvCompanyList.setLayoutManager(new LinearLayoutManager(getContext()));
        binding.errorMessage.setVisibility(View.GONE);
        binding.progressBar.setVisibility(View.VISIBLE);
    }

    private void bindViews() {
        binding.swipeRefreshLayout.setOnRefreshListener(() -> presenter.refreshCompanies());
    }

    @Override
    public void navigate(int id) {
        NavController navController = NavHostFragment.findNavController(this);
        Bundle bundle = new Bundle();
        bundle.putInt("companyId", id);
        navController.navigate(R.id.action_companyListFragment_to_companyItemFragment, bundle);
    }

    @Override
    public void showCompanies(List<Company> companies) {
        if (!companies.isEmpty()) {
            companyListAdapter.submitList(companies);
            binding.progressBar.setVisibility(View.GONE);
        } else showErrorMessage("목록이 비어있습니다.");
    }

    @Override
    public void showErrorMessage(String message) {
        binding.errorMessage.setVisibility(View.VISIBLE);
        binding.errorMessage.setText(message);
        binding.progressBar.setVisibility(View.GONE);
    }

    @Override
    public void undoRefreshLoading() {
        binding.swipeRefreshLayout.setRefreshing(false);
    }

    @Override
    public void onStop() {
        super.onStop();
        presenter.unsubscribe();
    }

    @Override
    public void onDestroyView() {
        super.onDestroyView();
        binding = null;
    }
}
```
사용자가 swipe를 해서 새로고침을 시도하는 경우 Presenter의 refreshCompanies()의 메소드를 호출한다. 레포지토리 내부에서 refreshCompanies()는 getCompanies(false), 즉 isFirstLoad의 값이 false 인채 전달이 되기 때문에 http 요청을 하게 된다.

지금까지 살펴봤듯이 RxJava는 I/O 작업에 대한 처리를 콜백이 아닌 스트림 형식으로 작성할 수 있어 MVP 아키텍처 구조 안에서 순차적, 논리적으로 작성할 수 있고 무엇보다 스케줄러를 이용해 thread switching이 굉장히 편리하다는 것을 알 수 있다. 그러나 원샷 작업 역시 스트림 방식으로 작성해야 하기에 chaining이 불필요하게 길어져 가독성을 떨어뜨리며 더군다나 transforming, filtering, combining operator들을 별도로 숙지하는 것 역시 쉽지 않다. Coroutine을 사용하면 **순차적 코드 작성을 통한 직관적인 코드를 보장**하고 **부분적으로 Flow를 사용해 스트림 형식으로 작성**할 수 있다고 하는데 반드시 다음 프로젝트에는 코루틴을 직접 적용시켜보도록 하겠다. 

