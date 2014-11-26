SwiftTask
=========

A native Swift port of BFTask from https://github.com/BoltsFramework/Bolts-iOS

What is this?

I'm a big fan of BFTask from https://github.com/BoltsFramework/Bolts-iOS

But as we moved to Swift I realized that inside of Swift we could make an even Better version!

Instead of doing a straight PORT, I did a simulataneous "port" and "rewrite".

With a few goals:

1) 100% Swift code.
2) Some interoperability with BFTask.  
3) Import some lessons learned by Facebook when they create FBTask (https://github.com/facebook/facebook-ios-sdk/blob/master/src/FBTask.h)


With that, while SwiftTask LOOKS a lot like BFTask, it's really a more swift friendly implementation.

Some background:
=========

Facebook ported a lighter version of BFTask into the latest Facebook SDK. (Called FBTask)  And they made some changes that I kind of agreed with.  Like renaming "continueWithBlock" to "dependentTaskWithBlocK".  After trying to indoctrinate a few co-workers into Bolts, it seemed that people preferred the FBTask nameing.  Since Bolts is "owned" by Facebook, I took this as the leading nomenclature and adopted it as the perferred method in my port.

I dropped the suffix "Block" from my methods.  The preferred calls are now "dependentTaskWith" and "completionTaskWith".  It just looks cleaner in Swift.   

Facebook also killed the BFExector.  Instead you just pass a dispatch_queue_t.   When I looked at my OWN BFExecutors, I realized that they all just created dispatch_queues, so why have a wrapper?   So I adopted the same strategy.  No more Executors.  

Facebook also made one big change, which is that by default, EACH BLOCK is seperately dispatched to it's queue everytime.  There is no "immediate" execution now, like the default BFExecutur.   I found this nice leading comment in the code : 
        "Always dispatching callbacks async consumes less stack space, and seems to be a little faster, but loses stacktrace information. "
        
I haven't VALIDATED that it's faster.  But those guys have Facebook have more time and money to test stuff than I do.  So i did the same thing, and I'm trusting they are right. Of course, XCode 6 now debugs asynch dispatch "stack trace" just fine. 

So I also killed the idea of "immediate exeuction".  The default queue is always dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), so be careful about task blocks that you want to run in the Main Queue. (I added a "dependentTaskOnMainQueueWith") 

But Facebook didn't port "taskForCompletionOfAllTasks"!    Which is crazy useful, if you want things to execute in parallel.   So I ported that from Bolts.   Cause maybe I over use it.  


CHANGES
=========
Added the enumeration TaskCompletionValue

```swift
enum SwiftTaskCompletionValue {
    case NotDone
    case Result(AnyObject?)
    case Error(NSError)
    case Exception(NSException)
    case Cancelled
  }
```
Tasks now just have a single property 

```swift
class SwiftTask: NSObject {
  var completionValue : TaskCompletionValue
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

 






