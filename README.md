# Pantrix Android SDK — Distribution

Auth-free Maven repository for the [Pantrix Android SDK](https://github.com/developersancho/pantrix-sdk-android).

CI on the source repo pushes a Maven directory layout onto this repo's
`maven-repo` branch every time a `v*` tag lands. Consumers resolve over
`raw.githubusercontent.com` with no credentials.

## Install

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven {
            url = uri("https://raw.githubusercontent.com/developersancho/pantrix-sdk-android-aar/maven-repo/")
        }
    }
}

// app/build.gradle.kts
dependencies {
    val pantrixVersion = "0.0.1-alpha.10"
    implementation("com.pantrix.analytics:pantrix-sdk:$pantrixVersion")
    debugImplementation("com.pantrix.analytics:pantrix-inspector:$pantrixVersion")
    releaseImplementation("com.pantrix.analytics:pantrix-inspector-noop:$pantrixVersion")
}
