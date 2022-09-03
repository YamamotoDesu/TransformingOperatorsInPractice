# TransformingOperatorsInPractice

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
