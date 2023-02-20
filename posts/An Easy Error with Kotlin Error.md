A well-designed API provides interfaces that are difficult to misuse. In general, the Kotlin APIs are delightful, but I recently debugged a crash with a stack trace that contained something similar to the following.

```kotlin
java.lang.IllegalStateException: Function0<java.lang.String>
	at SomeClass.andFunction(...)(SomeClass.kt)
    ...
```

An IllegalStateException with a Function message

The stack trace itself did not seem so puzzling. The application encountered an illegal state, but the `Function0<java.lang.String>` section of the stack trace surprised me because, typically, this section would contain a message that describes what caused the illegal state, such as `Unsupported data type`. A review of the crash site showed a piece of logic similar to the following playful example.

```kotlin
enum class Fruit {
    APPLE, GRAPES, WATERMELON, CANTALOUPE
}

class PicnicBasket {
    private val fruits = mutableListOf<Fruit>()

    fun pack(fruit: Fruit) {
        when (fruit) {
            Fruit.APPLE, Fruit.GRAPES -> fruits.add(fruit)
            Fruit.WATERMELON, Fruit.CANTALOUPE -> error { "A $fruit is too large to pack in a picnic basket" }
        }
    }
}
```

An Example Showing Error with Lambda

Specifically, take note of the `error` usage shown in the `when` block where the argument provided is a function that returns a `String`. At first glance, the call seems fine and compiles and throws an `IllegalStateException` as expected—just not with the intended `A $fruit is...` message. A closer look at the [Kotlin error function](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/error.html) shows a potential issue: the error function allows a developer to pass one argument of type `Any` as the message. This includes a function that returns a `String` like the provided example. Additionally, the `error` implementation will invoke `toString()` on the given message and then throw an `IllegalStateException` with the value. As a result, the app effectively performs the following when an error is called with a lambda.

This snippet shows the original puzzling `Function0<java.lang.String>` as the `IllegalStateException` message.

---

The resolution here is to simply replace the usage of `error { "An error occurred" }` with `error("An error occurred")` and then the stack trace will contain the expected error message rather than a lambda's `toString()` value, but taking a step back, it is important to understand how this programming error can occur in a project. While probably well intended, the flexibility of `error` allowing a message of type `Any` does mean that developers could make this mistake another way by perhaps passing an object that does not implement `toString`. Additionally, at first glance, `error { ... }` did not strike me as a mistake because of how much it resembles the Kotlin [check](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/check.html) API where developers are allowed to pass an optional lazy message as a trailing lambda as shown in the example below.

Making the same mistake using the check API would require a call site similar to `check(false) {{ "An error occurred" }}` which a programmer is highly unlikely to make compared to error with a trailing lambda.

---

To prevent this issue, I added a lint rule that checks for usages of error with a trailing lambda and warns developers that their stack trace may be missing valuable data. This may not be necessary for your project, but it can save a fellow engineer a headache trying to understand the nature of a crash. In short, double check your error call sites and continue to [fail-fast](https://en.wikipedia.org/wiki/Fail-fast) but with your intended messages. 💥