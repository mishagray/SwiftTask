SwiftTask
=========

A native Swift port of BFTask from https://github.com/BoltsFramework/Bolts-iOS

What is this?

I'm a big fan of BFTask from https://github.com/BoltsFramework/Bolts-iOS

But as we moved to Swift I realized that inside of Swift we could make an even Better version!

Instead of doing a straight PORT, I did a simulataneous "port" and "rewrite".

With a few goals:

- 100% Swift code.
- Some interoperability with BFTask.  
- Import some lessons learned by Facebook when they create FBTask (https://github.com/facebook/facebook-ios-sdk/blob/master/src/FBTask.h)


With that, while SwiftTask LOOKS a lot like BFTask, it's really a more swift friendly implementation.

Some background:
=========

Facebook ported a lighter version of BFTask into the latest Facebook SDK. (Called FBTask)  And they made some changes that I kind of agreed with.  Like renaming _"continueWithBlock"_ to _"dependentTaskWithBlocK"_.  After trying to indoctrinate a few co-workers into Bolts, it seemed that people preferred the FBTask nameing.  Since Bolts is "owned" by Facebook now, I took this as the leading nomenclature and adopted it as the perferred method in my port.

I dropped the suffix "Block" from my methods.  The preferred calls are now "dependentTaskWith" and "completionTaskWith".  It just looks cleaner in Swift.   

Facebook also killed the BFExector.  Instead you just pass a dispatch_queue_t.   When I looked at my OWN BFExecutors, I realized that they all just created dispatch_queues, so why have a wrapper?   So I adopted the same strategy.  No more Executors.  

Facebook also made one big change, which is that by default, EACH BLOCK is seperately dispatched to it's queue everytime.  There is no "immediate" execution now, like the default BFExecutor.   I found this nice leading comment in the code : 
        _"Always dispatching callbacks async consumes less stack space, and seems to be a little faster, but loses stacktrace information. "_
        
I haven't VALIDATED that it's faster.  But those guys have Facebook have more time and money to test stuff than I do.  So i did the same thing, and I'm trusting they are right. Of course, XCode 6 now debugs dispatch "stack traces" just fine. 

So I also killed "immediate execution".  The default queue is always _dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)_, so be careful about task blocks that you want to run in the Main Queue. If you want your task on the main queue just use _"dependentTaskOnMainQueueWith"_

But Facebook didn't port "taskForCompletionOfAllTasks"!    Which is crazy useful, if you want things to execute in parallel.   So I ported that from Bolts.   Cause maybe I over use it.  


CHANGES
=========
No more "Optional" Tasks.
If you import Bolts into your swift code, than all the BFTask methods end up returning optionals.  
That just polutes your code with "!" everywhere.  Bleh.

```swift
class SwiftTask: NSObject {
        func dependentTaskWith(block: (task : SwiftTask!) -> AnyObject?) -> SwiftTask
}
```

Added the enumeration SwiftTaskCompletionValue.  Note how each case can have optional data! 

```swift
enum SwiftTaskCompletionValue {
    case NotDone
    case Result(AnyObject?)
    case Error(NSError)
    case Exception(NSException)
    case Cancelled
  }
```
Tasks now just have a single property that indicate what their current status or result is.

```swift
class SwiftTask: NSObject {
  var completionValue : SwiftTaskCompletionValue
}
```
Since the enum doesn't get exposed Objective-C, I added back all the old BFTask properties (result, error, exception).  In case you want to use SwiftTask's in your Objective C code.

But this also means you can use a switch "Switch" statement to check your previous Tasks results.

Example:
```swift
functionThatReturnsAtTask.dependentTaskWith { (task : SwiftTask!) -> AnyObject? in
                switch task.completionValue {
                case let .NotDone:
                    assertionFailure("this shouldn't happen!")
                case .Cancelled:
                    NSLog("cancelled")
                case let .Exception(ex):
                    NSLog("exception \(ex)")
                case let .Error(e):
                    NSLog("error \(e.localizedDescription)")
                case let .Result(r):
                    NSLog("result = \(r)")
                }
              }
```


Or simplify:
```swift
functionThatReturnsAtTask.dependentTaskWith { (task : SwiftTask!) -> AnyObject? in
                switch task.completionValue {
                case let .Result(r):
                    NSLog("result = \(r)")
                default:
                    NSLog("ug!")
                }
              }
```


Or without a switch:
```swift
functionThatReturnsAtTask.dependentTaskWith { (task : SwiftTask!) -> AnyObject? in
                if task.completed {
                    NSLog("result = \(task.result)")
                    }
                else {
                    NSLog("ug!")
                }
              }
```
 

What about all my Bolts Code!?:
=========

If you have a lot of code that uses BFTask, great.  You can keep it.  
SwiftTask will also work in your Objective C code, but since BFTask has been adopted by other libraries (like AWS http://mobile.awsblog.com/post/Tx2B17V9NSVLP3I/The-AWS-Mobile-SDK-for-iOS-How-to-use-BFTask), I made some extensions to make your life easy.

There is an extension on SwiftTask to return a BFTask when you need it.

```swift
extension SwiftTask {
    
    func bftask() -> BFTask
}
```

So you can also generate a SwiftTask from a BFTask, but I also added some SwiftTask methods directly to BFTask.
Including the enumeration _completionValue_

```swift
extension BFTask  {
    var completionValue : SwiftTaskCompletionValue
    // ...
    func swiftTask() -> SwiftTask
    // ..
    func dependentTaskWith(block: (task : SwiftTask!) -> AnyObject?, queue: dispatch_queue_t? = nil) -> SwiftTask 
}
```
So you can convert a BFTask to a SwiftTask and visa-versa.  

It's also "legal" to return a BFTask as a result in your task block via "dependentTaskWith" of instead of a SwiftTask. Everything works fine.   The change in nomenclature HELPS keep track of the change.  But we don't expect people to give up using BFTask for a long while.




Didn't swift kill Exceptions?
=========
Yeah... But UIKit objects still throw them.   I wrote a quick Try/Catch/Finally implementation for Swift (using Objective-C of course).  So it's still possible to catch execptions thrown by your legacy code:

```objective-c
@interface SwiftExtras : NSObject

+ (void)Try:(void(^)())tryBlock catch:(void(^)(NSException *exception))exceptionBlock finally:(void(^)())finallyBlock;
+ (void)Try:(void(^)())tryBlock catch:(void(^)(NSException *exception))exceptionBlock;

@end
```

Here is a quick SwiftTask exception I made for NSFetchedResultsController, since Core Data likes to throw exceptions
```swift
extension NSFetchedResultsController {
    
    func performFetchWithTask() -> SwiftTask {
        
        var tcs = SwiftTaskCompletionSource()
        
        SwiftExtras.Try({ () -> Void in
            var error : NSError?
            let result = self.performFetch(&error)
            
            if (result == false) {
                tcs.setError(error!)
            }
            else {
                tcs.setResult(result)
            }
            }, catch: { (exception: NSException!) -> Void in
                tcs.trySetException(exception)
                return;
            })

        return tcs.task
    }
}
```


