# Hilt

이 포스팅은 [ToDo](https://github.com/tnvnfdla1214/ToDo) 프로젝트, [Delivery](https://github.com/tnvnfdla1214/DelevryProject)프로젝트, [Covid19_Map](https://github.com/tnvnfdla1214/Covid19_Map) 프로젝트 에서 발최한 것입니다.

## 목차
- Gradle 추가
- 기본(Application, Activity(Fragment), ViewModel)
- 모듈

## Gradle 추가

 ```Kotlin
buildscript {
    ...
    dependencies {
        ...
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:1.5.21" //hilt에서 1.5.20의 kapt버그이므로 올려주자
        classpath 'com.google.dagger:hilt-android-gradle-plugin:2.28-alpha'
    }
}
```
 ```Kotlin
...
apply plugin: 'kotlin-kapt'
apply plugin: 'dagger.hilt.android.plugin'

android {
    ...
}

dependencies {
    ...
    //hilt
    implementation "com.google.dagger:hilt-android:2.31.2-alpha"
    kapt "com.google.dagger:hilt-android-compiler:2.31.2-alpha"
    //Hilt로 ViewModel 객체 삽입
    kapt 'androidx.hilt:hilt-compiler:1.0.0-alpha01' // alpha02는 hilt 충돌 발생
}
```

 ```Kotlin
android {
  ...
  compileOptions {
    sourceCompatibility JavaVersion.VERSION_1_8
    targetCompatibility JavaVersion.VERSION_1_8
  }
}
```

## 기본(Application, Activity(Fragment), ViewModel)

### Application

 ```Kotlin
@HiltAndroidApp
class ExampleApplication : Application() { ... }
```

아래와 같이 App단위의 Context이 있다면 이런식으로 써내 사용 가능합니다.
 
<img src = "https://user-images.githubusercontent.com/48902047/152526469-ffda5bb5-4e8d-482b-a7ac-b917a87d0ce7.png">

### Acticity(Fragment)

 ```Kotlin
@AndroidEntryPoint
class ExampleActivity : AppCompatActivity() {

  @Inject lateinit var analytics: AnalyticsAdapter
  ...
}
```

Activity 혹은 Fragment 에 @AndroidEntryPoint를 통하여 주입합니다. 대해  또한 위와 같이 필요한 종속학목을 가져오려면 다음과 같이 @InJect를 사용합니다.

### ViewModel

 ```Kotlin
@HiltViewModel
class HomeViewModel
@Inject constructor(
    private val homeRepository: HomeRepository,
    private val mapRepository: MapRepository,
    private val userRepository: UserRepository,
) : BaseViewModel() {
```
위에 같이 해주었던 것을 아래와 같이 버젼업을 하면서 아래와 같이 사용됩니다.
 ```Kotlin
class ExampleViewModel @ViewModelInject constructor(
  private val repository: ExampleRepository,
  @Assisted private val savedStateHandle: SavedStateHandle
) : ViewModel() {
  ...
}
```

혹은 동적으로 주입 받고 싶은 순간이 있을 겁니다.(백타.....이거 할려고 진짜 노력 많이 했음)
본 예시는 레스토랑의 엔티티를 동적으로 받기 위해 아래와 같이 @AssistedInject로 선언하고 @AssistedFactory를 통해 원하는 주입을 선언해 줍니다. 그리고 companion object를 통해 Acitivty에서 주입시켜줄 함수를 정의해줍니다.

 ```Kotlin
class RestaurantDetailViewModel @AssistedInject constructor(
    @Assisted private val restaurantEntity: RestaurantEntity,
    private val restaurantFoodRepository: RestaurantFoodRepository,
    private val userRepository: UserRepository
) : BaseViewModel() {

    @AssistedFactory
    interface RestaurantDetailViewModelFactory {
        fun create(restaurantEntity: RestaurantEntity): RestaurantDetailViewModel
    }

    companion object {
        fun provideFactory(
            assistedFactory: RestaurantDetailViewModelFactory,
            restaurantEntity: RestaurantEntity
        ): ViewModelProvider.Factory = object : ViewModelProvider.Factory {
            override fun <T : ViewModel?> create(modelClass: Class<T>): T {
                return assistedFactory.create(restaurantEntity) as T
            }
        }
    }
```

아래는 Activity에서 @Inject를 통해 Factory를 선언 한 후 ViewModel을 선언할 때 레스토랑의 엔티티를 주입시켜줍니다.
## 모듈
아래 사진은 최소한의 모듈을 도식화 한 것 입니다. Retrofit과 Room을 연결시켜 Retrofit을 모듈로 연결 시킵니다.
<p align = center>
    <img src="https://user-images.githubusercontent.com/48902047/163676885-b555d316-d451-4ff6-bc9c-5878a00dd953.JPG"/>
</p>

### 1. APIMoudule

 ```Kotlin
@Module
@InstallIn(SingletonComponent::class)
object ApiModule {

    @Provides
    fun provideBaseUrl() = Constants.BASE_URL

    @Singleton
    @Provides
    fun provideOkHttpClient() = if (BuildConfig.DEBUG) {
        val loggingInterceptor = HttpLoggingInterceptor()
        loggingInterceptor.setLevel(HttpLoggingInterceptor.Level.BODY)
        OkHttpClient.Builder()
            .addInterceptor(loggingInterceptor)
            .build()
    } else {
        OkHttpClient.Builder().build()
    }

    @Singleton
    @APIService
    @Provides
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .client(okHttpClient)
            .baseUrl(provideBaseUrl())
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }

    @Provides
    @Singleton
    fun provideApiService(retrofit: Retrofit): ApiService {
        return retrofit.create(ApiService::class.java)
    }
    
    @Retention(AnnotationRetention.BINARY)
    @Qualifier
    annotation class APIService
}
    
```

### 2. DispatcherModule

 ```Kotlin
@Module
@InstallIn(SingletonComponent::class)
object DispatcherModule {
    @Provides
    @IoDispatcher
    @Singleton
    fun provideIODispatcher(): CoroutineDispatcher {
        return Dispatchers.IO
    }
    @Provides
    @MainDispatcher
    @Singleton
    fun provideMainDispatcher(): CoroutineDispatcher {
        return Dispatchers.Main
    }

    @Retention(AnnotationRetention.BINARY)
    @Qualifier
    annotation class IoDispatcher

    @Retention(AnnotationRetention.BINARY)
    @Qualifier
    annotation class MainDispatcher
}
```

### 3. LocalDBModule

 ```Kotlin
@Module
@InstallIn(SingletonComponent::class)
object LocalDBModule {

    //ApplicationContext 넣는법
    @Singleton
    @Provides
    fun provideAppPreferenceManager(@ApplicationContext context: Context) : AppPreferenceManager
            = AppPreferenceManager(context)

    @Provides
    @Singleton
    fun providesDao(applicationDatabase: ApplicationDatabase): Dao = applicationDatabase.Dao()

    @Provides
    @Singleton
    fun providesApplicationDatabase(@ApplicationContext context: Context): ApplicationDatabase
            = Room.databaseBuilder(context, ApplicationDatabase::class.java,"Database").build()

}
```
### 4. RepositoryModule

 ```Kotlin
@Module
@InstallIn(SingletonComponent::class)
object RepositoryModule {

    @Singleton
    @Provides
    fun provideRepository(
        ApiService: ApiService,
        dao: Dao,
        @DispatcherModule.IoDispatcher ioDispatcher: CoroutineDispatcher
    ): RestaurantRepository
            = RepositoryImpl(ApiService,dao,ioDispatcher)

}
```
