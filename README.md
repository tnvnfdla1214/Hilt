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


 ```Kotlin
@AndroidEntryPoint
class RestaurantDetailActivity : BaseActivity<RestaurantDetailViewModel, ActivityRestaurantDetailBinding>() {

    @Inject
    lateinit var restaurantdetailViewModelFactory: RestaurantDetailViewModel.RestaurantDetailViewModelFactory

    override val viewModel by viewModels<RestaurantDetailViewModel> {
        RestaurantDetailViewModel.provideFactory(
            restaurantdetailViewModelFactory,
            intent.getParcelableExtra<RestaurantEntity>(RestaurantListFragment.RESTAURANT_KEY) as RestaurantEntity
        )
    }
```

## 모듈
Repository등 Singleton으로 선언해주고 싶은 객체들이 있습니다. 그럴때 Module을 선언후 @Provides를 통해 주입해 줍니다.
 ```Kotlin
@Module
@InstallIn(SingletonComponent::class)
object FirebaseModule {

    @Provides
    @Singleton
    fun provideFirebasefirestore() : FirebaseFirestore {
        return Firebase.firestore
    }
    @Provides
    @Singleton
    fun provideFirebaseStorage() : FirebaseStorage {
        return FirebaseStorage.getInstance()
    }
    @Provides
    @Singleton
    fun provideFirebaseAuth() : FirebaseAuth {
        return FirebaseAuth.getInstance()
    }
}
```
이 중 주의 해 줘야 할 것은 먼저 해당 주입받을 객체가 ApplicationContext 단위일 경우 입니다. 그럴경우 아래와 같이 @ApplicationContext 선언해 주면 됩니다.

 ```Kotlin
@Module
@InstallIn(SingletonComponent::class)
object LocalDBModule {

    //ApplicationContext 넣는법
    @Singleton
    @Provides
    fun provideAppPreferenceManager(@ApplicationContext context: Context) : AppPreferenceManager
            = AppPreferenceManager(context)
    
    ...
}
```
다음은 사용이 같은 주입입니다. 예를 들어 같은 OkHttpClient 객체인 Map과 food를 받을 경우 아래와 같이 이름을 선언 후 주입하면 됩니다. 해당 맞는 객체에 어노테이션을 걸어주면 됩니다.

 ```Kotlin
@Module
@InstallIn(SingletonComponent::class)
class ProvideAPIModule {
  
    . . .
    
    /* 4. 기본 설정 및 data :
    *  지도와 음식을 가져오는 통신
    */
    @Singleton
    @Provides
    @MapAPIService
    fun provideMapRetrofit(
        okHttpClient: OkHttpClient,
        gsonConverterFactory: GsonConverterFactory,
    ): Retrofit {
        return Retrofit.Builder()
            .baseUrl(Url.TMAP_URL)
            .addConverterFactory(gsonConverterFactory)
            .client(okHttpClient)
            .build()
    }

    @Singleton
    @Provides
    @FoodAPIService
    fun provideFoodRetrofit(
        okHttpClient: OkHttpClient,
        gsonConverterFactory: GsonConverterFactory,
    ): Retrofit {
        return Retrofit.Builder()
            .baseUrl(Url.FOOD_URL)
            .addConverterFactory(gsonConverterFactory)
            .client(okHttpClient)
            .build()
    }

    @Retention(AnnotationRetention.BINARY)
    @Qualifier
    annotation class MapAPIService

    @Retention(AnnotationRetention.BINARY)
    @Qualifier
    annotation class FoodAPIService
}
```


