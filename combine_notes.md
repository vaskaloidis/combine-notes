# GitHub Combine

## Resources

- HeckJ: https://heckj.github.io/swiftui-notes/#introduction
- HeckJ Repo: https://github.com/heckj/swiftui-notes/tree/master/UIKit-Combine
- https://www.hackingwithswift.com/quick-start/swiftui/how-to-use-environmentobject-to-share-data-between-views
- https://cocoacasts.com/collections/combine-essentials

## Repositories

- https://github.com/V8tr/InfiniteListSwiftUI/tree/master/InfiniteListSwiftUI
- https://github.com/Swift-Compiled/pokelist/tree/master/PokeList


### Books

 - Combine-Aync Book: https://www.raywenderlich.com/books/combine-asynchronous-programming-with-swift/v1.0/chapters/3-transforming-operators#toc-chapter-006-anchor-001
 - Book Repo: https://github.com/raywenderlich/comb-materials/tree/editions/2.0

```
Username: vaskaloidis
Password: Flyp2021!!!
```

### Misc

 - https://www.swiftbysundell.com/articles/building-custom-combine-publishers-in-swift/
 - https://github.com/aidevjoe/SweetKit/blob/master/SweetKit/SweetKit/Extensions/UIKit/UIScrollView%2BExtension.swift

## Code Notes

```swift
  // Combine MVVM https://github.com/V8tr/ModernMVVM  
  // https://github.com/V8tr/ModernMVVM/tree/master/ModernMVVM/Features/MovieDetails

  import Foundation
  import Combine

  final class MovieDetailViewModel: ObservableObject {
      @Published private(set) var state: State

      private var bag = Set<AnyCancellable>()

      private let input = PassthroughSubject<Event, Never>()

      init(movieID: Int) {
          state = .idle(movieID)

          Publishers.system(
              initial: state,
              reduce: Self.reduce,
              scheduler: RunLoop.main,
              feedbacks: [
                  Self.whenLoading(),
                  Self.userInput(input: input.eraseToAnyPublisher())
              ]
          )
          .assign(to: \.state, on: self)
          .store(in: &bag)
      }

      func send(event: Event) {
          input.send(event)
      }
  }

  // MARK: - Inner Types
  extension MovieDetailViewModel {
      enum State {
          case idle(Int)
          case loading(Int)
          case loaded(MovieDetail)
          case error(Error)
      }

      enum Event {
          case onAppear
          case onLoaded(MovieDetail)
          case onFailedToLoad(Error)
      }

      struct MovieDetail {
          let id: Int
          let title: String
          let overview: String?
          let poster: URL?
          let rating: Double?
          let duration: String
          let genres: [String]
          let releasedAt: String
          let language: String

          init(movie: MovieDetailDTO) {
              id = movie.id
              title = movie.title
              overview = movie.overview
              poster = movie.poster
              rating = movie.vote_average

              let formatter = DateComponentsFormatter()
              formatter.unitsStyle = .abbreviated
              formatter.allowedUnits = [.minute, .hour]
              duration = movie.runtime.flatMap { formatter.string(from: TimeInterval($0 * 60)) } ?? "N/A"

              genres = movie.genres.map(\.name)

              releasedAt = movie.release_date ?? "N/A"

              language = movie.spoken_languages.first?.name ?? "N/A"
          }
      }
  }

  // MARK: - State Machine
  extension MovieDetailViewModel {
      static func reduce(_ state: State, _ event: Event) -> State {
          switch state {
          case .idle(let id):
              switch event {
              case .onAppear:
                  return .loading(id)
              default:
                  return state
              }
          case .loading:
              switch event {
              case .onFailedToLoad(let error):
                  return .error(error)
              case .onLoaded(let movie):
                  return .loaded(movie)
              default:
                  return state
              }
          case .loaded:
              return state
          case .error:
              return state
          }
      }

      static func whenLoading() -> Feedback<State, Event> {
          Feedback { (state: State) -> AnyPublisher<Event, Never> in
              guard case .loading(let id) = state else { return Empty().eraseToAnyPublisher() }
              return MoviesAPI.movieDetail(id: id)
                  .map(MovieDetail.init)
                  .map(Event.onLoaded)
                  .catch { Just(Event.onFailedToLoad($0)) }
                  .eraseToAnyPublisher()
          }
      }

      static func userInput(input: AnyPublisher<Event, Never>) -> Feedback<State, Event> {
          Feedback(run: { _ in
              return input
          })
      }
  }
```

```swift
  // filter() vs. compactMap()

  // https://cocoacasts.com/collections/combine-essentials

  private func setupBindings() {
      reachablePublisher
          .filter { $0 }
          .sink { [weak self] _ in
              self?.fetchEpisodes()
          }.store(in: &subscriptions)
  }

  private func setupBindings() {
      reachablePublisher
          .compactMap { $0 ? $0 : nil }
          .sink { [weak self] _ in
              self?.fetchEpisodes()
          }.store(in: &subscriptions)
  }
```

```swift
  // AnyCancellable

  var mySubscriber: AnyCancellable?

  let mySinkSubscriber = remotePublisher
      .sink { data in
          print("received ", data)
      }
  mySubscriber = AnyCancellable(mySinkSubscriber)

  // Alternatively

  private var cancellableSet: Set<AnyCancellable> = []

  let mySinkSubscriber = remotePublisher
      .sink { data in
          print("received ", data)
      }
      .store(in: &cancellableSet)
```

```swift
// SwiftUI Published Binding

struct ReactiveForm: View {

    @ObservedObject var model: ReactiveFormModel
    // $model is a ObservedObject<ExampleModel>.Wrapper
    // and $model.objectWillChange is a Binding<ObservableObjectPublisher>
    @State private var buttonIsDisabled = true
    // $buttonIsDisabled is a Binding<Bool>
    var body: some View {
        VStack {
            Text("Reactive Form")
                .font(.headline)

            Form {
                TextField("first entry", text: $model.firstEntry)
                    .textFieldStyle(RoundedBorderTextFieldStyle())
                    .lineLimit(1)
                    .multilineTextAlignment(.center)
                    .padding()

                TextField("second entry", text: $model.secondEntry)
                    .textFieldStyle(RoundedBorderTextFieldStyle())
                    .multilineTextAlignment(.center)
                    .padding()

                VStack {
                    ForEach(model.validationMessages, id: \.self) { msg in
                        Text(msg)
                            .foregroundColor(.red)
                            .font(.callout)
                    }
                }
            }

            Button(action: {}) {
                Text("Submit")
            }.disabled(buttonIsDisabled)
                .onReceive(model.submitAllowed) { submitAllowed in
                    self.buttonIsDisabled = !submitAllowed
            }
            .padding()
            .background(RoundedRectangle(cornerRadius: 10)      .stroke(Color.blue, lineWidth: 1)
            )


            Spacer()
        }
    }
}

struct HeadingView: View {
    @ObservedObject var locationModel: LocationProxy
    @State var lastHeading: CLHeading?
    @State var lastLocation: CLLocation?

    var body: some View {
        VStack {
            HStack {
                Text("authorization status:")
                Text(locationModel.authorizationStatusString())
            }
            if (locationModel.authorizationStatus == .notDetermined) {
                Button(action: {
                    self.locationModel.requestAuthorization()
                }) {
                    Image(systemName: "lock.shield")
                    Text("Request location authorization")
                }
                .padding()
                .background(RoundedRectangle(cornerRadius: 10)      .stroke(Color.blue, lineWidth: 1)
                )
            }
            if (self.lastHeading != nil) {
                Text("Heading: ")+Text(String(self.lastHeading!.description))
            }
            if (self.lastLocation != nil) {
                Text("Location: ")+Text(lastLocation!.description)
                ZStack {
                    Circle()
                        .stroke(Color.blue, lineWidth: 1)

                    GeometryReader { geometry in
                        Path { path in
                            let minWidthHeight = min(geometry.size.height, geometry.size.width)

                            path.move(to: CGPoint(x: geometry.size.width/2, y: geometry.size.height/2))
                            path.addLine(to: CGPoint(x: geometry.size.width/2, y: geometry.size.height/2 - minWidthHeight/2 + 5)  )
                        }
                        .stroke()
                        .rotation(Angle(degrees: self.lastLocation!.course))
                        .animation(.linear)
                    }
                }

            }
        }
        .onReceive(self.locationModel.headingPublisher) { heading in
            self.lastHeading = heading
        }
        .onReceive(self.locationModel.locationPublisher, perform: {
            self.lastLocation = $0
        })

    }
}

struct ContentView : View {
    @ObservedObject var model: ReactiveFormModel

    var body: some View {
        TabView {
            ReactiveForm(model: model)
            .tabItem {
                Image(systemName: "1.circle")
                Text("Reactive Form")
            }

            HeadingView(locationModel: LocationProxy())
            .tabItem {
                Image(systemName: "mappin.circle")
                Text("Location")
            }
        }
    }
}
```

```swift
// https://heckj.github.io/swiftui-notes/#reference-swiftui

// SwiftUI Binding

class Contact : ObservableObject {
    @Published var name: String
    @Published var age: Int
    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
    func haveBirthday() -> Int {
        age += 1
        return age
    }
 }

 let john = Contact(name: "John Appleseed", age: 24)
 let cancellable = john.objectWillChange.sink { _ in
    expectation.fulfill()
    print("will change")
    // Prints "will change"
    // Prints "25"
}
print(john.haveBirthday())
```
