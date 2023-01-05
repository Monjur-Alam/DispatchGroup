# How to Handle Multiple Async Operations in Swift

![image](https://user-images.githubusercontent.com/40492085/210708266-031945e7-1f7d-404f-a9b1-0b62b1764677.png)

We’ve had cases where we need to wait for a few APIs to finish and then process further logic or update the UI accordingly. We’ll see what I (and probably many others) used to do and then the best approach to use with Swift.

## Problem
As you can see below, we’re calling two APIs, and then we want to update the UI. We’ll use Alamofire to keep things simple with API requests. We’re also keeping track of how much time it took for both of the APIs to complete.

Note: If you’re trying this out on your own, then don’t forget to set App Transport Security Settings ➡ Allow Arbitrary Loads to `YES` in your `Info.plist` file.

## Code
```
import UIKit
import Alamofire

final class ViewController: UIViewController {

  // 1
  private struct Endpoint {
    static let posts = "http://jsonplaceholder.typicode.com/posts"
    static let comments = "http://jsonplaceholder.typicode.com/comments"
  }

  // 2
  @IBOutlet weak var label: UILabel!
  @IBOutlet weak var activityIndicatorView: UIActivityIndicatorView!

  // 3
  private var startTimeInterval: TimeInterval?

  // 4
  private var isWaiting = false {
    didSet {
      self.updateUI()
    }
  }

  // 5
  override func viewDidLoad() {
    super.viewDidLoad()

    self.isWaiting = true

    // Step1: Call posts API
    AF.request(Endpoint.posts).response { response in

      // Step2: Call comments API
      AF.request(Endpoint.comments).response { response in

        // Step3: Update the UI
        self.isWaiting = false
      }
    }
  }

  // 6
  private func updateUI() {
    if self.isWaiting {
      self.startTimeInterval = Date().timeIntervalSinceReferenceDate
      self.label.text = "Waiting for APIs"
      self.activityIndicatorView.startAnimating()
    } else {
      let delta = Date().timeIntervalSinceReferenceDate - self.startTimeInterval! // Don't use force casting in production
      self.label.text = "Responses received in \(delta)"
      self.activityIndicatorView.stopAnimating()
    }
  }
}
```
## Code explanation
- The endpoint is the structure holding together URLs to the APIs we want to call later
- An `IBOutlet` is needed to access UI objects in order to update them when the API response states changes
- `startTimeInterval` is the reference to when we started our first API call
- `isWaiting` is tracking if there’s an API request for which we’re awaiting a response. And upon state changes, it updates the UI.
- In `viewDidLoad()`, we’re calling two APIs using Alamofire. Please keep in mind that we’re nesting them. So those API requests will execute one after another, not simultaneously.
- `updateUI()` is really all about updating the label and `activityIndicatorView` with the appropriate text and animation, respectively
## Storyboard
If you’re wondering how our storyboard looks like, it’s very much left to a bare minimum. It’s just a label and an activity indicator.
![image](https://user-images.githubusercontent.com/40492085/210711826-1b55029c-af3a-4b81-a673-753d524286ca.png)
## Solution: ‘DispatchGroup’
Now enters (pun intended!) the [DispatchGroup](https://developer.apple.com/documentation/dispatch/dispatchgroup). It’s specifically designed for cases like this, grouping together multiple tasks and waiting for them to finish. When all the tasks finish, it can notify via a Swift closure so we can proceed further.
## Code
```
// 5
override func viewDidLoad() {
  super.viewDidLoad()

  self.isWaiting = true

  // Step0: Creating and entering dispatch groups
  let group = DispatchGroup()
  group.enter()
  group.enter()

  // Step1: Call posts API
  AF.request(Endpoint.posts).response { response in
    group.leave()
  }

  // Step2: Call comments API
  AF.request(Endpoint.comments).response { response in
    group.leave()
  }

  group.notify(queue: .main, execute: {
    // Step3: Update the UI
    self.isWaiting = false
  })
}
```
## Code explanation
- We’ve added a `DispatchGroup` instance
- We use the `enter()` method on the group. This is to tell the `DispatchGroup` we’re waiting for something. Think of the `DispatchGroup` like it’s keeping a count of how many times `enter()` was called. So the counter will increase by one every time `enter()` is called.
- We’ve removed the nested API calls and how we’re calling both APIs at the same time. Overall, this makes the API waiting time shorter. Here, note that we’re using the `leave()` method to tell `DisaptchGroup` the task is complete, meaning the API response has been received. By calling the `leave()` method, we’ll decrease our imaginary counter by one. As soon as our imaginary counter reaches zero, `DispatchGroup` will call the `notify()` method.

![image](https://user-images.githubusercontent.com/40492085/210712646-08637417-1ff1-4a48-a2d2-504b11e0a0d0.png)
![image](https://user-images.githubusercontent.com/40492085/210712691-9dfcdf9f-b8f4-49e2-816d-22cc699f9cc7.png)

Note: Every `enter()` call should be accompanied by a `leave()` call also. Otherwise, the `notify()` method won’t get executed at all.
You may be tempted to only call `enter()` right before the API call, like below:
```
// Step1: Call posts API
group.enter()
AF.request(Endpoint.posts).response { response in
  group.leave()
}
```
But I’d advise you to avoid that, as very very rarely, if the API response is too fast, the group is entered and left almost immediately, and if there are no other scheduled tasks in the group, it’ll call the `notify()` method and not wait for your next tasks.

So it’s my personal opinion that we should group `enter()` calls together if we know how many tasks we’re going to start beforehand.

Conclusion
Great! Today we learned how to use `DispatchGroup` to wait for async operations. There are different ways to use `DispatchGroup`, but that’s for another day.

Thank you for reading.
