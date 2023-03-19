## Coroutines under the hood

It is necessary to point out that much of the text below is taken from the article written by [Marcin Moskała](https://kt.academy/article/cc-under-the-hood) 

* Key lessons are:
* Suspending functions are like state machines, with a possible state at the beginning of the function and after each suspending function call.
* Both the number identifying the state and the local data are kept in the continuation object.
* Continuation of a function decorates a continuation of its caller function; as a result, all these continuations represent a call stack that is used when we resume or a resumed function completes.

Key word #1 - Continuation-passing style (CPS)

Continuation-passing style (CPS) is a way of writing computer programs that helps programmers control the order in which different tasks are carried out. In CPS, every step of the program is designed to say what needs to happen next, which makes it easier to build complex programs.  

We can say too that CPS is a style of programming in which control is passed explicitly in the form of a continuation*.

- Continuation: in CS, is an abstract representation of the control state of a computer program.

A function written in continuation-passing style takes an extra argument: an explicit "continuation"; i.e., a function of one argument. When the CPS function has computed its result value, it "returns" it by calling the continuation function with this value as the argument. 

Example #1

```kotlin
// define a function that adds two numbers and calls a continuation function when done
fun addNumbers(a: Int, b: Int, cont: (Int) -> Unit) {
    val result = a + b
    cont(result)
}

// define a continuation function that prints the result of the calculation
fun printResult(result: Int) {
    println("The result is: $result")
}

// use CPS to add two numbers and print the result
addNumbers(3, 5, ::printResult)
```

Rather than returning a value directly, the addNumbers function calls the continuation function with the result, which tells the program what to do next. By passing the printResult function as the continuation, we can print the result of the calculation to the console after the addition is done.

Example #2

Direct style

```kotlin
fun sumTwoSqrt(x: Double, y: Double): Double {
    return kotlin.math.sqrt(x * x + y * y)
}
```

Continuation passing style

```kotlin
fun sumTwoSqrtCPS(x: Double, y: Double, k: (Double) -> Unit) {
    val x2 = x * x
    val y2 = y * y
    val x2py2 = x2 + y2
    val result = kotlin.math.sqrt(x2py2)
    k(result)
}

// example usage
sumTwoSqrtCPS(3.0, 4.0) { result ->
    println("The result is: $result")
} 
```

Example #3 -> using coroutines

Direct style

```kotlin
import kotlinx.coroutines.*

fun main() {
	runBlocking {
		buttonHandler()
	}
}

suspend fun setField(value: Int) {
	println("Setting field to $value")
}

suspend fun lookup(parameter: Int): Int {
	println("Looking up result for parameter $parameter...")
	delay(1000)
	return parameter * 2
}

suspend fun buttonHandler() {
	val parameter = 5 
	val result = withContext(Dispatchers.IO) {
    	lookup(parameter)
	}
	setField(result)
}
```

 Continuation passing style

```kotlin
import kotlinx.coroutines.*

fun main() {
	runBlocking {
		buttonHandler { /* no-op */ }
	}
}

typealias CPS = () -> Unit

suspend fun setField(value: Int, callback: CPS) {
	println("Setting field to $value")
	callback()
}

suspend fun lookup(parameter: Int, callback: (Int) -> Unit) {
	println("Looking up result for parameter $parameter...")
	delay(1000)
	callback(parameter * 2)
}

suspend fun buttonHandler(callback: CPS) {
	val parameter = 5
	withContext(Dispatchers.IO) {
		lookup(parameter) { result ->
			launch { setField(result, callback) }
		}
	}
}
```

- The code above is passing a continuation function (here a callback) as an argument to another function. In other words, instead of returning a result, the function calls The buttonHandler function takes a continuation function callback as an argument, which will be called when the function completes its operation.
- The buttonHandler is an asynchronous function that performs two operations sequentially. 
- First, it calls the lookup function with a parameter value of 5. 
- The lookup function takes a callback function that will be called with the result of the lookup. In this case, the callback function launches a coroutine that calls the setField function with the result value and the callback function.
- The setField function sets the field to the given value and then calls the callback function, which is the continuation function passed to buttonHandler. 
- This callback function is executed once setField completes its operation.


Going a little further, if you want to talk a little bit about state machine, we can say that - implicitly -we could discretize a list of the states the coroutine goes through during the execution of the buttonHandler function:

1. Entering the buttonHandler function and setting the parameter value to 5.
2. Calling the lookup function with the parameter value.
3. Suspending the buttonHandler coroutine while the lookup function is executing and waiting for its result.
4. Receiving the result of the lookup function and launching a new coroutine to call the setField function with the result and the callback function.
5. Suspending the new coroutine while the setField function is executing.
6. Resuming the new coroutine once the setField function completes its operation and invokes the callback function.
7. Resuming the buttonHandler coroutine once the new coroutine completes its operation and the callback function is invoked.

These states can be summarized as follows:

1. Initializing
2. Performing lookup
3. Suspended while waiting for lookup to complete
4. Processing result of lookup and launching new coroutine
5. Suspended while waiting for setField to complete
6. Resumed after setField completes and invokes the callback function
7. Resumed after buttonHandler completes and the callback function is invoked

Each state corresponds to a different phase of the coroutine's execution and represents a different state of the program's logic.

* State machine for buttonHandler function:

  +--------------+
  | Initializing |
  +--------------+
         |
         | invoke lookup function with parameter=5
         V
  +-----------------+
  | Waiting for data |
  +-----------------+
         |
         | receive data from lookup function
         V
  +-----------------+
  | Calling setField |
  +-----------------+
         |
         | invoke setField function with data and callback
         V
  +----------------+
  | Waiting for set |
  +----------------+
         |
         | receive callback from setField function
         V
  +------------+
  | Calling cb  |
  +------------+
         |
         | invoke callback
         V
  +-------------+
  | Exiting      |
  +-------------+

——————————————————————————————————————————————————————————————————————————————————————————
************************************************************************************************************************************************************************
——————————————————————————————————————————————————————————————————————————————————————————

> More deeply…

suspend fun getUser(): User?
suspend fun setUser(user: User)
suspend fun checkAvailability(flight: Flight): Boolean

// under the hood is
fun getUser(continuation: Continuation<*>): Any?
fun setUser(user: User, continuation: Continuation<*>): Any
fun checkAvailability( flight: Flight,  continuation: Continuation<*>): Any

- The result type under the hood is different from the originally declared one.  The reason is that a suspending function might be suspended, and so it might not return a declared type.
- It returns a special COROUTINE_SUSPENDED marker
- Notice that since getUser might return User? or COROUTINE_SUSPENDED (which is of type Any), its result type must be the closest supertype of User? and Any, so it is Any?.

Kotlin could introduce union types, in which case we will have User? | COROUTINE_SUSPENDED instead.

Another very simple function

```kotlin
suspend fun myFunction() {
  println("Before")
  delay(1000) // suspending
  println("After")
}
```

myFunction function signature will look under the hood —> fun myFunction(continuation: Continuation<*>): Any

- The next thing is that this function needs its own continuation in order to remember its state. 
- Let's name it MyFunctionContinuation 
- The actual continuation is an object expression and has no name
- myFunction will wrap the continuation (the parameter) with its own continuation (MyFunctionContinuation): val continuation = MyFunctionContinuation(continuation) 
- This should be done only if the continuation isn't wrapped already: if it is, this is part of the resume process, and we should keep the continuation unchanged:

val continuation =  if (continuation is MyFunctionContinuation) continuation else MyFunctionContinuation(continuation)

This condition can be simplified to: val continuation = continuation as? MyFunctionContinuation  ?: MyFunctionContinuation(continuation)

Analyzing the body of our myFunction

The function could be started from two places:
- Either from the beginning (in the case of a first call) 
- Or from the point after suspension (in the case of resuming from continuation).

To identify the current state, we use a field called label.
At the start, it is 0, therefore the function will start from the beginning. However, it is set to the next state before each suspension point so that we start from just after the suspension point after a resume.

```kotlin
// A simplified picture of how myFunction looks under the hood
fun myFunction(continuation: Continuation<Unit>): Any {
    val continuation = continuation as? MyFunctionContinuation
        ?: MyFunctionContinuation(continuation)

    if (continuation.label == 0) {
        println("Before")
        continuation.label = 1
        if (delay(1000, continuation) == COROUTINE_SUSPENDED){
            return COROUTINE_SUSPENDED
        }
    }
    if (continuation.label == 1) {
        println("After")
        return Unit
    }
    error("Impossible")
}
```

* Point of attention 01: The last important piece is also presented in the snippet above. When delay is suspended, it returns COROUTINE_SUSPENDED, then myFunction returns COROUTINE_SUSPENDED; the same is done by the function that called it, and the function that called this function, and all other functions until the top of the call stack. This is how a suspension ends all these functions and leaves the thread available for other runnables (including coroutines) to be used.

Analyzing the above code 

- What would happen if this delay call didn’t return COROUTINE_SUSPENDED?
- What if it just returned Unit instead (it is impossible for this to happen, but let's hypothesize)?

*** If the delay just returned Unit, we would just move to the next state, and the function would behave like any other. ***

We will now discuss the continuation, which is implemented as an anonymous class. In simplified terms, it can be represented as follows:

```kotlin
cont = object : ContinuationImpl(continuation) {
    var result: Any? = null
    var label = 0

    override fun invokeSuspend(`$result$: Any?): Any? {
        this.result = $result`;
        return myFunction(this);
    }
};
```

- The code defines a new instance of an anonymous class that extends the ContinuationImpl class, which is used to represent the continuation of a coroutine. The continuation parameter passed to the constructor is the parent continuation, to which the new continuation is attached.
- The anonymous class contains two fields: result and label. result is an Any? variable that will store the result of the coroutine when it is completed. label is an Int variable that will be used to keep track of the current execution state of the coroutine.
- The anonymous class also overrides the invokeSuspend method, which is the method that will be called when the coroutine is suspended.
- The $result$ parameter is the value that the coroutine was suspended with, and it is assigned to the result field of the anonymous class.
- The method then calls the myFunction function, passing itself as a parameter, and returns the result of the function.

Summarizing: the purpose of this code is to create a new continuation that can be used to resume the coroutine at a later time, by passing it as an argument to a function that expects a continuation. The invokeSuspend method is the method that will be called when the coroutine is resumed, and it will pass the result of the coroutine back to the caller. The result field is used to store the result of the coroutine when it is completed, so that it can be returned to the caller later. The label field is used to keep track of the current execution state of the coroutine, so that it can resume from where it left off.

> In JVM, type arguments are erased during compilation; so, for instance, both Continuation<Unit> or Continuation<String> become just Continuation. Since everything we present here is Kotlin representation of JVM bytecode, you should not worry about these type arguments at all.

The above sentence is PARTIALLY true.

In the JVM, type arguments are indeed erased during compilation through a process called type erasure. This means that at runtime, the JVM does not have information about the specific type arguments used in a generic class or function. This applies to the Continuation interface as well, which is a generic interface that can be used to represent the continuation of a coroutine.

However, the Kotlin compiler provides a feature called reified type parameters, which allows the compiler to retain type information for certain type parameters at runtime. This means that when using the Continuation interface in Kotlin, you can specify the type parameter, and it will be retained at runtime through the use of reified type parameters. This allows the Kotlin coroutine runtime to differentiate between continuations of different types.

So, while it is true that type arguments are erased during compilation in the JVM, it is not entirely accurate to say that you should not worry about these type arguments when working with the Continuation interface in Kotlin. 

In Kotlin, you can and should specify the type parameter for the Continuation interface, as it allows the coroutine runtime to perform type-safe operations on the continuation.

>> That being said, the code below presents a complete simplification of such function under the hood:

```kotlin
import java.util.concurrent.Executors
import java.util.concurrent.TimeUnit
import kotlin.coroutines.Continuation
import kotlin.coroutines.CoroutineContext
import kotlin.coroutines.EmptyCoroutineContext
import kotlin.coroutines.resume

fun myFunction(continuation: Continuation<Unit>): Any {
    val continuation = continuation as? MyFunctionContinuation
    ?: MyFunctionContinuation(continuation)

    if (continuation.label == 0) {
        println("Before")
        continuation.label = 1
        if (delay(1000, continuation) == COROUTINE_SUSPENDED){
            return COROUTINE_SUSPENDED
        }
    }
    if (continuation.label == 1) {
        println("After")
        return Unit
    }
    error("Impossible")
}

class MyFunctionContinuation(
    val completion: Continuation<Unit>
        ) : Continuation<Unit> {
    override val context: CoroutineContext
    get() = completion.context

    var label = 0
    var result: Result<Any>? = null

    override fun resumeWith(result: Result<Unit>) {
        this.result = result
        val res = try {
            val r = myFunction(this)
            if (r == COROUTINE_SUSPENDED) return
            Result.success(r as Unit)
        } catch (e: Throwable) {
            Result.failure(e)
        }
        completion.resumeWith(res)
    }
}

private val executor = Executors
    .newSingleThreadScheduledExecutor {
        Thread(it, "scheduler").apply { isDaemon = true }
    }

fun delay(timeMillis: Long, continuation: Continuation<Unit>): Any {
    executor.schedule({
        continuation.resume(Unit)
    }, timeMillis, TimeUnit.MILLISECONDS)
    return COROUTINE_SUSPENDED
}

fun main() {
    val EMPTY_CONTINUATION = object : Continuation<Unit> {
        override val context: CoroutineContext =
            EmptyCoroutineContext

        override fun resumeWith(result: Result<Unit>) {
            // This is root coroutine, we don't need anything in this example
        }
    }
    myFunction(EMPTY_CONTINUATION)
    Thread.sleep(2000)
    // Needed to don't let the main finish immediately.
}

val COROUTINE_SUSPENDED = Any()
```

>> Again: the code above is a custom implementation of coroutines without using Kotlin's built-in coroutine support. It includes a function called myFunction that simulates a non-blocking delay using a custom continuation and an executor for scheduling.

* Main components:

1. MyFunctionContinuation class:
This class is a custom implementation of Continuation<Unit> that keeps track of the current coroutine execution state. It has a label property to manage the current execution state, a result property to store the result, and a completion property representing the completion continuation.

2. myFunction function:
This function accepts a Continuation<Unit> as an argument and performs a simulated delay using the delay function. The function has two states:
a. When label is 0, it prints "Before", sets the label to 1, and calls the delay function.
b. When label is 1, it prints "After" and returns Unit.

3. delay function:
This function takes a time in milliseconds and a continuation as arguments. It schedules the resumption of the coroutine using a single-threaded executor after the specified delay. It returns COROUTINE_SUSPENDED to signal that the coroutine is suspended and waiting to be resumed.

4. main function:
This function creates an instance of EMPTY_CONTINUATION and calls myFunction with it. It then sleeps the main thread for 2 seconds to prevent the main function from finishing immediately.

5. COROUTINE_SUSPENDED:
A global value representing the suspended state of a coroutine. It's used as a signal to indicate that a coroutine is suspended and waiting to be resumed.

>> In IntelliJ IDEA, you can inspect the underlying suspended functions by opening the “Tools > Kotlin > Show Kotlin bytecode”, and selecting the "Decompile" option. This will display the decompiled Java code, providing you with a clearer understanding of how the code would appear if it were written in Java. See the result below!

```java
// MyFunctionContinuation.java
import kotlin.Metadata;
import kotlin.Result;
import kotlin.ResultKt;
import kotlin.Unit;
import kotlin.coroutines.Continuation;
import kotlin.coroutines.CoroutineContext;
import kotlin.jvm.internal.Intrinsics;
import org.jetbrains.annotations.NotNull;
import org.jetbrains.annotations.Nullable;

@Metadata(
   mv = {1, 7, 0},
   k = 1,
   d1 = {"\u0000,\n\u0002\u0018\u0002\n\u0002\u0018\u0002\n\u0002\u0010\u0002\n\u0002\b\u0005\n\u0002\u0018\u0002\n\u0002\b\u0003\n\u0002\u0010\b\n\u0002\b\u0005\n\u0002\u0018\u0002\n\u0002\u0010\u0000\n\u0002\b\u0007\u0018\u00002\b\u0012\u0004\u0012\u00020\u00020\u0001B\u0013\u0012\f\u0010\u0003\u001a\b\u0012\u0004\u0012\u00020\u00020\u0001¢\u0006\u0002\u0010\u0004J\u001e\u0010\u0018\u001a\u00020\u00022\f\u0010\u0011\u001a\b\u0012\u0004\u0012\u00020\u00020\u0012H\u0016ø\u0001\u0000¢\u0006\u0002\u0010\u0019R\u0017\u0010\u0003\u001a\b\u0012\u0004\u0012\u00020\u00020\u0001¢\u0006\b\n\u0000\u001a\u0004\b\u0005\u0010\u0006R\u0014\u0010\u0007\u001a\u00020\b8VX\u0096\u0004¢\u0006\u0006\u001a\u0004\b\t\u0010\nR\u001a\u0010\u000b\u001a\u00020\fX\u0086\u000e¢\u0006\u000e\n\u0000\u001a\u0004\b\r\u0010\u000e\"\u0004\b\u000f\u0010\u0010R+\u0010\u0011\u001a\n\u0012\u0004\u0012\u00020\u0013\u0018\u00010\u0012X\u0086\u000eø\u0001\u0000ø\u0001\u0001ø\u0001\u0002¢\u0006\u000e\n\u0000\u001a\u0004\b\u0014\u0010\u0015\"\u0004\b\u0016\u0010\u0017\u0082\u0002\u000f\n\u0002\b\u0019\n\u0005\b¡\u001e0\u0001\n\u0002\b!¨\u0006\u001a"},
   d2 = {"LMyFunctionContinuation;", "Lkotlin/coroutines/Continuation;", "", "completion", "(Lkotlin/coroutines/Continuation;)V", "getCompletion", "()Lkotlin/coroutines/Continuation;", "context", "Lkotlin/coroutines/CoroutineContext;", "getContext", "()Lkotlin/coroutines/CoroutineContext;", "label", "", "getLabel", "()I", "setLabel", "(I)V", "result", "Lkotlin/Result;", "", "getResult-xLWZpok", "()Lkotlin/Result;", "setResult", "(Lkotlin/Result;)V", "resumeWith", "(Ljava/lang/Object;)V", "rewards_debug"}
)
public final class MyFunctionContinuation implements Continuation {
   private int label;
   @Nullable
   private Result result;
   @NotNull
   private final Continuation completion;

   @NotNull
   public CoroutineContext getContext() {
      return this.completion.getContext();
   }

   public final int getLabel() {
      return this.label;
   }

   public final void setLabel(int var1) {
      this.label = var1;
   }

   @Nullable
   public final Result getResult_xLWZpok/* $FF was: getResult-xLWZpok*/() {
      return this.result;
   }

   public final void setResult(@Nullable Result var1) {
      this.result = var1;
   }

   public void resumeWith(@NotNull Object result) {
      this.result = Result.box-impl(result);

      Object r;
      try {
         r = CoroutinesUnderTheHoodKt.myFunction((Continuation)this);
         if (Intrinsics.areEqual(r, CoroutinesUnderTheHoodKt.getCOROUTINE_SUSPENDED())) {
            return;
         }

         Result.Companion var4 = Result.Companion;
         if (r == null) {
            throw new NullPointerException("null cannot be cast to non-null type kotlin.Unit");
         }

         Unit var7 = (Unit)r;
         r = Result.constructor-impl(var7);
      } catch (Throwable var6) {
         Result.Companion var5 = Result.Companion;
         r = Result.constructor-impl(ResultKt.createFailure(var6));
      }

      this.completion.resumeWith(r);
   }

   @NotNull
   public final Continuation getCompletion() {
      return this.completion;
   }

   public MyFunctionContinuation(@NotNull Continuation completion) {
      Intrinsics.checkNotNullParameter(completion, "completion");
      super();
      this.completion = completion;
   }
}
// CoroutinesUnderTheHoodKt.java
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.TimeUnit;
import kotlin.Metadata;
import kotlin.Result;
import kotlin.Unit;
import kotlin.coroutines.Continuation;
import kotlin.coroutines.CoroutineContext;
import kotlin.coroutines.EmptyCoroutineContext;
import kotlin.jvm.internal.Intrinsics;
import org.jetbrains.annotations.NotNull;

@Metadata(
   mv = {1, 7, 0},
   k = 2,
   d1 = {"\u0000$\n\u0000\n\u0002\u0010\u0000\n\u0002\b\u0003\n\u0002\u0018\u0002\n\u0002\b\u0003\n\u0002\u0010\t\n\u0000\n\u0002\u0018\u0002\n\u0002\u0010\u0002\n\u0002\b\u0003\u001a\u001c\u0010\u0007\u001a\u00020\u00012\u0006\u0010\b\u001a\u00020\t2\f\u0010\n\u001a\b\u0012\u0004\u0012\u00020\f0\u000b\u001a\u0006\u0010\r\u001a\u00020\f\u001a\u0014\u0010\u000e\u001a\u00020\u00012\f\u0010\n\u001a\b\u0012\u0004\u0012\u00020\f0\u000b\"\u0011\u0010\u0000\u001a\u00020\u0001¢\u0006\b\n\u0000\u001a\u0004\b\u0002\u0010\u0003\"\u0016\u0010\u0004\u001a\n \u0006*\u0004\u0018\u00010\u00050\u0005X\u0082\u0004¢\u0006\u0002\n\u0000¨\u0006\u000f"},
   d2 = {"COROUTINE_SUSPENDED", "", "getCOROUTINE_SUSPENDED", "()Ljava/lang/Object;", "executor", "Ljava/util/concurrent/ScheduledExecutorService;", "kotlin.jvm.PlatformType", "delay", "timeMillis", "", "continuation", "Lkotlin/coroutines/Continuation;", "", "main", "myFunction", "rewards_debug"}
)
public final class CoroutinesUnderTheHoodKt {
   private static final ScheduledExecutorService executor;
   @NotNull
   private static final Object COROUTINE_SUSPENDED;

   @NotNull
   public static final Object myFunction(@NotNull Continuation continuation) {
      Intrinsics.checkNotNullParameter(continuation, "continuation");
      Continuation var10000 = continuation;
      if (!(continuation instanceof MyFunctionContinuation)) {
         var10000 = null;
      }

      MyFunctionContinuation var3 = (MyFunctionContinuation)var10000;
      if (var3 == null) {
         var3 = new MyFunctionContinuation(continuation);
      }

      MyFunctionContinuation continuation = var3;
      String var2;
      if (continuation.getLabel() == 0) {
         var2 = "Before";
         System.out.println(var2);
         continuation.setLabel(1);
         if (Intrinsics.areEqual(delay(1000L, (Continuation)continuation), COROUTINE_SUSPENDED)) {
            return COROUTINE_SUSPENDED;
         }
      }

      if (continuation.getLabel() == 1) {
         var2 = "After";
         System.out.println(var2);
         return Unit.INSTANCE;
      } else {
         var2 = "Impossible";
         throw new IllegalStateException(var2.toString());
      }
   }

   @NotNull
   public static final Object delay(long timeMillis, @NotNull final Continuation continuation) {
      Intrinsics.checkNotNullParameter(continuation, "continuation");
      executor.schedule((Runnable)(new Runnable() {
         public final void run() {
            Continuation var1 = continuation;
            Unit var2 = Unit.INSTANCE;
            Result.Companion var10001 = Result.Companion;
            var1.resumeWith(Result.constructor-impl(var2));
         }
      }), timeMillis, TimeUnit.MILLISECONDS);
      return COROUTINE_SUSPENDED;
   }

   public static final void main() {
      <undefinedtype> EMPTY_CONTINUATION = new Continuation() {
         @NotNull
         private final CoroutineContext context;

         @NotNull
         public CoroutineContext getContext() {
            return this.context;
         }

         public void resumeWith(@NotNull Object result) {
         }

         {
            this.context = (CoroutineContext)EmptyCoroutineContext.INSTANCE;
         }
      };
      myFunction((Continuation)EMPTY_CONTINUATION);
      Thread.sleep(2000L);
   }

   // $FF: synthetic method
   public static void main(String[] var0) {
      main();
   }

   @NotNull
   public static final Object getCOROUTINE_SUSPENDED() {
      return COROUTINE_SUSPENDED;
   }

   static {
      executor = Executors.newSingleThreadScheduledExecutor((ThreadFactory)null.INSTANCE);
      COROUTINE_SUSPENDED = new Object();
   }
}
```
	      
## References

- https://kt.academy/article/cc-under-the-hood
- https://en.wikipedia.org/wiki/Continuation
- https://en.wikipedia.org/wiki/Continuation-passing_style

