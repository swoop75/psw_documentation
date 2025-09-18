# PSW_ANDROID - Android Application Infrastructure

## Overview
PSW_ANDROID provides a native Android experience for portfolio management with offline capabilities, biometric authentication, and push notifications.

## Technology Stack
- **Language**: Kotlin
- **UI Framework**: Jetpack Compose
- **Architecture**: MVVM with Clean Architecture
- **Database**: Room + SQLite
- **Networking**: Retrofit + OkHttp
- **DI**: Hilt (Dagger)
- **Authentication**: Biometric + JWT

## Application Architecture

### Module Structure
```
app/
├── src/main/java/com/psw/android/
│   ├── data/
│   │   ├── local/         # Room database, DAOs
│   │   ├── remote/        # API services, DTOs
│   │   └── repository/    # Repository implementations
│   ├── domain/
│   │   ├── model/         # Domain models
│   │   ├── repository/    # Repository interfaces
│   │   └── usecase/       # Business logic
│   ├── presentation/
│   │   ├── ui/            # Compose screens
│   │   ├── viewmodel/     # ViewModels
│   │   └── navigation/    # Navigation setup
│   └── di/                # Dependency injection
└── build.gradle
```

### Build Configuration
```kotlin
// build.gradle (Module: app)
android {
    compileSdk 34

    defaultConfig {
        applicationId "com.psw.android"
        minSdk 26
        targetSdk 34
        versionCode 1
        versionName "1.0.0"
    }

    buildFeatures {
        compose true
    }

    compileOptions {
        sourceCompatibility JavaVersion.VERSION_17
        targetCompatibility JavaVersion.VERSION_17
    }
}

dependencies {
    // Compose BOM
    implementation platform('androidx.compose:compose-bom:2023.10.01')
    implementation 'androidx.compose.ui:ui'
    implementation 'androidx.compose.material3:material3'

    // Architecture
    implementation 'androidx.lifecycle:lifecycle-viewmodel-compose:2.7.0'
    implementation 'androidx.navigation:navigation-compose:2.7.4'

    // Database
    implementation 'androidx.room:room-runtime:2.6.0'
    implementation 'androidx.room:room-ktx:2.6.0'
    kapt 'androidx.room:room-compiler:2.6.0'

    // Networking
    implementation 'com.squareup.retrofit2:retrofit:2.9.0'
    implementation 'com.squareup.retrofit2:converter-gson:2.9.0'
    implementation 'com.squareup.okhttp3:logging-interceptor:4.12.0'

    // DI
    implementation 'com.google.dagger:hilt-android:2.48'
    kapt 'com.google.dagger:hilt-compiler:2.48'

    // Security
    implementation 'androidx.biometric:biometric:1.1.0'
    implementation 'androidx.security:security-crypto:1.1.0-alpha06'
}
```

## Data Layer Implementation

### Room Database
```kotlin
// data/local/PSWDatabase.kt
@Database(
    entities = [
        PortfolioEntity::class,
        StockEntity::class,
        HoldingEntity::class
    ],
    version = 1,
    exportSchema = false
)
@TypeConverters(Converters::class)
abstract class PSWDatabase : RoomDatabase() {
    abstract fun portfolioDao(): PortfolioDao
    abstract fun stockDao(): StockDao
    abstract fun holdingDao(): HoldingDao
}

// data/local/dao/PortfolioDao.kt
@Dao
interface PortfolioDao {
    @Query("SELECT * FROM portfolios WHERE userId = :userId")
    fun getPortfolios(userId: String): Flow<List<PortfolioEntity>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertPortfolio(portfolio: PortfolioEntity)

    @Query("DELETE FROM portfolios WHERE id = :portfolioId")
    suspend fun deletePortfolio(portfolioId: String)
}
```

### API Service
```kotlin
// data/remote/PSWApiService.kt
interface PSWApiService {
    @GET("auth/user")
    suspend fun getCurrentUser(): Response<UserDto>

    @GET("portfolios")
    suspend fun getPortfolios(): Response<List<PortfolioDto>>

    @GET("portfolios/{id}")
    suspend fun getPortfolio(@Path("id") id: String): Response<PortfolioDto>

    @POST("portfolios")
    suspend fun createPortfolio(@Body portfolio: CreatePortfolioDto): Response<PortfolioDto>

    @GET("stocks/search")
    suspend fun searchStocks(@Query("q") query: String): Response<List<StockDto>>
}

// data/remote/AuthInterceptor.kt
class AuthInterceptor @Inject constructor(
    private val tokenManager: TokenManager
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val token = tokenManager.getToken()

        val authenticatedRequest = if (token != null) {
            request.newBuilder()
                .header("Authorization", "Bearer $token")
                .build()
        } else {
            request
        }

        return chain.proceed(authenticatedRequest)
    }
}
```

## Security Implementation

### Biometric Authentication
```kotlin
// presentation/auth/BiometricAuthManager.kt
@Singleton
class BiometricAuthManager @Inject constructor(
    @ApplicationContext private val context: Context
) {
    fun authenticateWithBiometric(
        fragment: Fragment,
        onSuccess: () -> Unit,
        onError: (String) -> Unit
    ) {
        val biometricPrompt = BiometricPrompt(
            fragment as FragmentActivity,
            ContextCompat.getMainExecutor(context),
            object : BiometricPrompt.AuthenticationCallback() {
                override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                    onSuccess()
                }

                override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
                    onError(errString.toString())
                }
            }
        )

        val promptInfo = BiometricPrompt.PromptInfo.Builder()
            .setTitle("Authenticate to access PSW")
            .setSubtitle("Use your fingerprint or face to continue")
            .setNegativeButtonText("Cancel")
            .build()

        biometricPrompt.authenticate(promptInfo)
    }
}
```

### Secure Storage
```kotlin
// data/local/SecurePreferences.kt
@Singleton
class SecurePreferences @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()

    private val sharedPreferences = EncryptedSharedPreferences.create(
        context,
        "psw_secure_prefs",
        masterKey,
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
    )

    fun saveToken(token: String) {
        sharedPreferences.edit()
            .putString(KEY_AUTH_TOKEN, token)
            .apply()
    }

    fun getToken(): String? {
        return sharedPreferences.getString(KEY_AUTH_TOKEN, null)
    }
}
```

## UI Implementation with Compose

### Main Screen
```kotlin
// presentation/ui/dashboard/DashboardScreen.kt
@Composable
fun DashboardScreen(
    viewModel: DashboardViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsState()

    LazyColumn(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp)
    ) {
        item {
            PortfolioSummaryCard(
                totalValue = uiState.totalValue,
                dayChange = uiState.dayChange,
                modifier = Modifier.fillMaxWidth()
            )
        }

        items(uiState.portfolios) { portfolio ->
            PortfolioCard(
                portfolio = portfolio,
                onClick = { viewModel.onPortfolioClick(portfolio.id) },
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(vertical = 4.dp)
            )
        }
    }
}

@Composable
fun PortfolioCard(
    portfolio: Portfolio,
    onClick: () -> Unit,
    modifier: Modifier = Modifier
) {
    Card(
        modifier = modifier.clickable { onClick() },
        elevation = CardDefaults.cardElevation(defaultElevation = 4.dp)
    ) {
        Column(
            modifier = Modifier.padding(16.dp)
        ) {
            Text(
                text = portfolio.name,
                style = MaterialTheme.typography.headlineSmall
            )

            Text(
                text = "$${portfolio.value}",
                style = MaterialTheme.typography.bodyLarge,
                color = if (portfolio.dayChange >= 0)
                    MaterialTheme.colorScheme.primary
                else
                    MaterialTheme.colorScheme.error
            )
        }
    }
}
```

## Push Notifications

### Firebase Setup
```kotlin
// services/PSWFirebaseMessagingService.kt
class PSWFirebaseMessagingService : FirebaseMessagingService() {

    override fun onMessageReceived(remoteMessage: RemoteMessage) {
        super.onMessageReceived(remoteMessage)

        val title = remoteMessage.notification?.title ?: "PSW Update"
        val body = remoteMessage.notification?.body ?: ""

        showNotification(title, body, remoteMessage.data)
    }

    override fun onNewToken(token: String) {
        super.onNewToken(token)
        // Send token to backend
        sendTokenToServer(token)
    }

    private fun showNotification(title: String, body: String, data: Map<String, String>) {
        val intent = Intent(this, MainActivity::class.java).apply {
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
        }

        val pendingIntent = PendingIntent.getActivity(
            this, 0, intent, PendingIntent.FLAG_IMMUTABLE
        )

        val notification = NotificationCompat.Builder(this, CHANNEL_ID)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(title)
            .setContentText(body)
            .setContentIntent(pendingIntent)
            .setAutoCancel(true)
            .build()

        NotificationManagerCompat.from(this).notify(1, notification)
    }
}
```

## Build & Deployment

### Gradle Build Script
```kotlin
// build.gradle (Project level)
buildscript {
    ext {
        compose_version = '1.5.4'
        kotlin_version = '1.9.10'
    }
}

// app/build.gradle
android {
    signingConfigs {
        release {
            storeFile file('../keystore/psw-release.jks')
            storePassword System.getenv("KEYSTORE_PASSWORD")
            keyAlias System.getenv("KEY_ALIAS")
            keyPassword System.getenv("KEY_PASSWORD")
        }
    }

    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
            signingConfig signingConfigs.release

            buildConfigField "String", "API_BASE_URL", "\"https://api.psw.com/v1\""
        }

        debug {
            buildConfigField "String", "API_BASE_URL", "\"http://10.0.2.2:8000/api/v1\""
        }
    }
}
```

### ProGuard Configuration
```proguard
# proguard-rules.pro
-keep class com.psw.android.data.remote.dto.** { *; }
-keep class com.psw.android.domain.model.** { *; }

# Retrofit
-keepattributes Signature, InnerClasses, EnclosingMethod
-keepclassmembers,allowshrinking,allowobfuscation interface * {
    @retrofit2.http.* <methods>;
}

# Room
-keep class * extends androidx.room.RoomDatabase
-dontwarn androidx.room.paging.**
```

## Testing Infrastructure

### Unit Tests
```kotlin
// test/java/com/psw/android/PortfolioRepositoryTest.kt
@ExtendWith(MockitoExtension::class)
class PortfolioRepositoryTest {

    @Mock
    private lateinit var apiService: PSWApiService

    @Mock
    private lateinit var portfolioDao: PortfolioDao

    private lateinit var repository: PortfolioRepository

    @Before
    fun setup() {
        repository = PortfolioRepositoryImpl(apiService, portfolioDao)
    }

    @Test
    fun `getPortfolios should return cached data when available`() = runTest {
        // Given
        val cachedPortfolios = listOf(
            PortfolioEntity(id = "1", name = "Test Portfolio", value = 1000.0)
        )
        whenever(portfolioDao.getPortfolios("user1")).thenReturn(flowOf(cachedPortfolios))

        // When
        val result = repository.getPortfolios("user1").first()

        // Then
        assertEquals(1, result.size)
        assertEquals("Test Portfolio", result[0].name)
    }
}
```

## Performance Optimization

### Image Loading
```kotlin
// utils/ImageLoader.kt
@Composable
fun AsyncImage(
    url: String,
    contentDescription: String?,
    modifier: Modifier = Modifier
) {
    var imageState by remember { mutableStateOf<ImageState>(ImageState.Loading) }

    val imageLoader = ImageLoader.Builder(LocalContext.current)
        .memoryCache {
            MemoryCache.Builder(LocalContext.current)
                .maxSizePercent(0.25)
                .build()
        }
        .diskCache {
            DiskCache.Builder()
                .directory(LocalContext.current.cacheDir.resolve("image_cache"))
                .maxSizeBytes(50 * 1024 * 1024) // 50MB
                .build()
        }
        .build()

    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(url)
            .crossfade(true)
            .build(),
        contentDescription = contentDescription,
        imageLoader = imageLoader,
        modifier = modifier
    )
}
```

## Troubleshooting

### Common Issues

**Build Failures**
```bash
# Clean and rebuild
./gradlew clean
./gradlew build

# Clear caches
rm -rf ~/.gradle/caches/
```

**API Connection Issues**
```kotlin
// Network debugging
class NetworkInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        Log.d("Network", "Request: ${request.url}")

        try {
            val response = chain.proceed(request)
            Log.d("Network", "Response: ${response.code}")
            return response
        } catch (e: Exception) {
            Log.e("Network", "Error: ${e.message}")
            throw e
        }
    }
}
```

---

*Infrastructure documentation for PSW_ANDROID Mobile Application*