# TransformingOperatorsInPractice

## RxSwift: Reactive Programming with Swift | raywenderlich.com
![image](https://user-images.githubusercontent.com/47273077/185172130-b3557025-c636-4a1b-8490-c900c8312b77.png)

## Transforming the response
```swift
  func fetchEvents(repo: String) {
    let response = Observable.from([repo]).map { urlString -> URL in
      return URL(string: "https://api.github.com/repos/\(urlString)/events")!
    }.map { url -> URLRequest in
      return URLRequest(url: url)
    }.flatMap { request -> Observable<(response: HTTPURLResponse, data: Data)> in
      return URLSession.shared.rx.response(request: request)
    }.share(replay: 1) // keep in a buffer the last emitted event
    
    response.filter { response, _ in
      return 200..<300 ~= response.statusCode
    }
    .compactMap { _, data -> [Event]? in
      return try? JSONDecoder().decode([Event].self, from: data)
    }
    .subscribe(onNext: { [weak self] newEvents in
      self?.processEvents(newEvents)
    })
    .disposed(by: bag)
  }
```

## Processing the response

```swift
  private let events = BehaviorRelay<[Event]>(value: [])
  
  func processEvents(_ newEvents: [Event]) {
    var updatedEvents = newEvents + events.value
    if updatedEvents.count > 50 {
      updatedEvents = [Event](updatedEvents.prefix(upTo: 50)) // this way you will show only the latest activity
    }
    
    events.accept(updatedEvents)
    DispatchQueue.main.async {
      self.tableView.reloadData()
      self.refreshControl?.endRefreshing()
    }
  }
```
<img width="490" src="https://user-images.githubusercontent.com/47273077/188258517-564cf17b-1579-4b06-b928-8ccda648ca57.gif">

## Add a last-modified header to the request

```swift
  private let lastModified = BehaviorRelay<String?>(value: nil)
  private let modifiedFileURL = cachedFileURL("modified.txt")
  
  override func viewDidLoad() {
    super.viewDidLoad()
    
    if let lastModifiedString = try? String(contentsOf: modifiedFileURL, encoding: .utf8) {
      lastModified.accept(lastModifiedString)
    }
    
  }
  
 func fetchEvents(repo: String) {
 
     let response = Observable.from([repo]).map { urlString -> URL in
      return URL(string: "https://api.github.com/repos/\(urlString)/events")!
    }.map { [weak self] url -> URLRequest in
      var request = URLRequest(url: url)
      if let modifiedHeader = self?.lastModified.value {
        request.addValue(modifiedHeader, forHTTPHeaderField: "Last-Modified")
      }
      return request
    }.flatMap { request -> Observable<(response: HTTPURLResponse, data: Data)> in
      return URLSession.shared.rx.response(request: request)
    }.share(replay: 1)
    
     response.filter { response, _ in
      return 200..<400 ~= response.statusCode
    }
    .flatMap { response, _ -> Observable<String> in
      guard let value = response.allHeaderFields["Last-Modified"] as? String else {
        return Observable.empty()
      }
      return Observable.just(value)
    }
    .subscribe(onNext: { [weak self] modifiedHeader in
      guard let self = self else { return }
      
      self.lastModified.accept(modifiedHeader)
      try? modifiedHeader.write(to: self.modifiedFileURL, atomically: true, encoding: .utf8)
    })
    .disposed(by: bag)
 }

```
