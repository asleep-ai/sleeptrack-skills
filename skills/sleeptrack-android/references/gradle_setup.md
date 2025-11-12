# Gradle Setup for Asleep SDK

## Project-level build.gradle

```gradle
buildscript {
    dependencies {
        classpath 'com.google.dagger:hilt-android-gradle-plugin:2.48'
    }
}

plugins {
    id 'com.android.application' version '8.2.2' apply false
    id 'com.android.library' version '8.2.2' apply false
    id 'org.jetbrains.kotlin.android' version '1.9.24' apply false
    id 'com.google.dagger.hilt.android' version '2.48' apply false
}
```

## App-level build.gradle

```gradle
plugins {
    id 'com.android.application'
    id 'org.jetbrains.kotlin.android'
    id 'kotlin-kapt'
    id 'dagger.hilt.android.plugin'
}

android {
    namespace 'com.example.yourapp'
    compileSdk 34

    defaultConfig {
        applicationId "com.example.yourapp"
        minSdk 24  // Minimum SDK 24 required for Asleep SDK
        targetSdk 34
        versionCode 1
        versionName "1.0"

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"

        // Store API key securely in local.properties
        Properties properties = new Properties()
        properties.load(project.rootProject.file('local.properties').newDataInputStream())
        buildConfigField "String", "ASLEEP_API_KEY", properties['asleep_api_key']
    }

    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }

    compileOptions {
        sourceCompatibility JavaVersion.VERSION_17
        targetCompatibility JavaVersion.VERSION_17
    }

    kotlinOptions {
        jvmTarget = '17'
    }

    buildFeatures {
        viewBinding true
        buildConfig = true
    }
}

dependencies {
    // Core Android dependencies
    implementation 'androidx.core:core-ktx:1.12.0'
    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'com.google.android.material:material:1.11.0'
    implementation 'androidx.constraintlayout:constraintlayout:2.1.4'

    // Hilt for dependency injection
    implementation "com.google.dagger:hilt-android:2.48"
    kapt "com.google.dagger:hilt-compiler:2.48"

    // Activity and Fragment KTX
    implementation 'androidx.activity:activity-ktx:1.8.2'
    implementation 'androidx.fragment:fragment-ktx:1.6.2'
    implementation 'androidx.lifecycle:lifecycle-service:2.7.0'

    // Required for Asleep SDK
    implementation 'com.squareup.okhttp3:okhttp:4.11.0'
    debugImplementation 'com.squareup.okhttp3:logging-interceptor:4.10.0'
    implementation 'com.google.code.gson:gson:2.10'

    // Asleep SDK (check for latest version)
    implementation 'ai.asleep:asleepsdk:3.1.4'

    // Testing
    testImplementation 'junit:junit:4.13.2'
    androidTestImplementation 'androidx.test.ext:junit:1.1.5'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.5.1'
}
```

## local.properties

```properties
# Store your Asleep API key securely (never commit this file to version control)
asleep_api_key="your_api_key_here"
```

## Key Requirements

1. **Minimum SDK**: 24 (Android 7.0)
2. **Target SDK**: 34 recommended
3. **Java Version**: 17
4. **Kotlin Version**: 1.9.24 or higher
5. **Gradle Plugin**: 8.2.2 or higher

## Dependency Notes

- **OkHttp**: Required for network operations
- **Gson**: Required for JSON parsing
- **Hilt**: Recommended for dependency injection (not strictly required)
- **ViewBinding**: Recommended for type-safe view access

## ProGuard Rules

If using ProGuard/R8, add these rules to `proguard-rules.pro`:

```proguard
# Asleep SDK
-keep class ai.asleep.asleepsdk.** { *; }
-dontwarn ai.asleep.asleepsdk.**

# Gson
-keepattributes Signature
-keepattributes *Annotation*
-dontwarn sun.misc.**
-keep class * implements com.google.gson.TypeAdapter
-keep class * implements com.google.gson.TypeAdapterFactory
-keep class * implements com.google.gson.JsonSerializer
-keep class * implements com.google.gson.JsonDeserializer

# OkHttp
-dontwarn okhttp3.**
-dontwarn okio.**
-keepnames class okhttp3.internal.publicsuffix.PublicSuffixDatabase
```
