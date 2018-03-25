[![CircleCI](https://circleci.com/gh/Aidanvii7/Toolbox.svg?style=svg)](https://circleci.com/gh/Aidanvii7/Toolbox)
[![](https://jitpack.io/v/Aidanvii7/Toolbox.svg)](https://jitpack.io/#Aidanvii7/Toolbox)


# Toolbox
Toolbox is a collection of libraries for Android. 

Currently it provides the following functionality:
* [Observable property delegates](#observable-property-delegates)
* [Observable property delegates with databinding](#observable-property-delegates-with-databinding)
* [Architecture component viewmodel integration](#architecture-component-viewmodel-integration)

# Setup

Latest Toolbox version:
```gradle
toolbox_version = 'v0.3.1-alpha'
```

 Add the JitPack repository to your build file: 

```gradle
repositories {
  ...
	maven { url 'https://jitpack.io' }    
}
```

Pick and choose the artifacts you need:

```gradle

// The base library, currently there's only common typealiases and constants here. It's unlikely you would use this directly.
api "com.github.Aidanvii7.Toolbox:common:$toolbox_version"

// Contains an alternative base implementation to [BaseObservable] from the data binding library ([NotifiableObservable]), 
// that uses composition - so view-models can simply implement [NotifiableObserble] and use any base class (if any).
api "com.github.Aidanvii7.Toolbox:databinding:$toolbox_version"

// Adds a convenience base class that implements [NotifiableObservable] and also extends [ViewModel] from Google's
// Architecture components library.
api "com.github.Aidanvii7.Toolbox:databinding-arch-viewmodel:$toolbox_version"

// Contains a set of property delegates, serving as an enhancement over the [ObservableProperty] class 
// that is shipped with the Kotlin standard library.
// These delegates can be chained together in a functional way.
api "com.github.Aidanvii7.Toolbox:delegates-observable:$toolbox_version"

// Provides an [ObservableProperty] implementation that integrates with the android data binding library.
api "com.github.Aidanvii7.Toolbox:delegates-observable-databinding:$toolbox_version"

```

## Observable property delegates

The delegates-observable artifact provides a set of property delegates that allow functional style syntax.

Each delegate chain must begin with a [`ObservableProperty.Source`](https://github.com/Aidanvii7/Toolbox/blob/master/delegates-observable/src/main/java/com/aidanvii/toolbox/delegates/observable/ObservableProperty.kt) delegate, for example:

```kotlin
val myProperty by observable("")
```
Where `observable(..)` provides a source observable (`ObservableProperty.Source` class).

Once a source observable is created, a set of 'decorator' observables (`ObservableProperty` interface) can be chained in a functional way such as:

```kotlin
var nonNullString by observable<String>("")
        .eager() // propagate initial value downstream instead of waiting on subsequent assignments to property
        .onFirstAccess { /* lazily do something the first time this property is accessed/read */ }
        .filter { it.isNotEmpty() } // ignore empty strings
        .doOnNext { /* do something with the initial value */ }
        .skip(1) // ignore initial value
        .distinctUntilChanged() // ignore subsequent values that are the same as the previous value
        .doOnNext { /* do something with subsequent values */ }
        .map { it.length }
        .doOnNext { stringLength -> /* do something with length */ }
```

It also supports nullable types:
```kotlin
var nullableString by observable<String?>(null)
        .eager() // propagate initial value downstream instead of waiting on subsequent assignments to property
        .onFirstAccess { /* lazily do something the first time this property is accessed/read */ }
        .filterNotNullWith { it.isNotEmpty() } // ignore empty strings
        .doOnNext { /* do something with the initial value */ }
        .skip(1) // ignore initial value
        .distinctUntilChanged() // ignore subsequent values that are the same as the previous value
        .doOnNext { /* do something with subsequent values */ }
        .map { it.length }
        .doOnNext { stringLength -> /* do something with length */ }
```

Please see the source code documentation for a description of each decorator.

## Observable property delegates with databinding
The delegates-observable-databinding artifact provides a source observable implementation that integrates with data-binding.

Two types of delegates exist, `bindable(..)` and `bindableEvent(..)`.
* `bindable(..)` will only propagate values downstream and notify data binding if the value given is different from the previous value.
* `bindableEvent(..)` will propagate values downstream and notify data binding even if the given value is the same as the previous value.

Both of these delegates can only be used as member properties of [`NotifiableObservable`](https://github.com/Aidanvii7/Toolbox/blob/master/databinding/src/main/java/com/aidanvii/toolbox/databinding/NotifiableObservable.kt) implementations, such as:

```kotlin
class ViewModel : NotifiableObservable by NotifiableObservable.delegate() {

    init {
        initDelegator(this) // required
    }
    
    @get:Bindable
    var expanded by bindable(false)
            .eager()
            .doOnNext { /* Do something */ }

    @get:Bindable
    var showToast by bindableEvent(false)
            .distinctUntilChanged()
            .doOnNext { /* do something with distinct value */ }
}
```

A base class implementation exists for [`NotifiableObservable`](https://github.com/Aidanvii7/Toolbox/blob/master/databinding/src/main/java/com/aidanvii/toolbox/databinding/NotifiableObservable.kt) exists to cut some of the boilerplate called [`ObservableViewModel`](https://github.com/Aidanvii7/Toolbox/blob/master/databinding/src/main/java/com/aidanvii/toolbox/databinding/ObservableViewModel.kt):
```kotlin
class ViewModel : ObservableViewModel() {
    
    @get:Bindable
    var expanded by bindable(false)
    ...
}
```

Both of these delegates use reflection in a inexpensive way to match the property name to a property constant ID in your app's `BR.java` class that is generated by the data binding compiler. The [`PropertyMapper`](https://github.com/Aidanvii7/Toolbox/blob/master/databinding/src/main/java/com/aidanvii/toolbox/databinding/PropertyMapper.kt) object must know about your app's `BR.java` class, and should be initialised once during the lifetime of the app.

Consider doing this in a custom `Application` class such as:
```kotlin

class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        PropertyMapper.initBRClass(
                brClass = BR::class.java,
                locked = true // prevents multiple invocations at runtime
        )
    }
}

```
Please see the [documentation](https://github.com/Aidanvii7/Toolbox/blob/master/delegates-observable-databinding/src/main/java/com/aidanvii/toolbox/databinding/BindableProperty.kt) of each for a deeper explanation and usage suggestions.

# Architecture component viewmodel integration
The databinding-arch-viewmodel artifact simply provides a base class implementation similar to [`ObservableViewModel`](https://github.com/Aidanvii7/Toolbox/blob/master/databinding/src/main/java/com/aidanvii/toolbox/databinding/ObservableViewModel.kt) which extends the [`ViewModel`](https://developer.android.com/topic/libraries/architecture/viewmodel.html) class from the architecture components library, called [`ObservableArchViewModel`](https://github.com/Aidanvii7/Toolbox/blob/master/databinding-arch-viewmodel/src/main/java/com/aidanvii/toolbox/databinding/ObservableArchViewModel.kt).

# Alpha status
This library is currently in alpha, meaning that the API may change from version to version, and also that I am looking for feedback.
