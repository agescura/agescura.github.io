---
layout: default
---

# SwiftUI, CasePaths y DataSources.

Utilizar CasePaths tiene utilidad únicamente en The Composable Architecture o en el Route de la navegación. Puede utilizarse en multiples escenarios. Uno de ellos puede ser para representar la información que mostraremos en una pantalla en concreto. Veamos un ejemplo práctico.

Empezamos por un objeto Todo.

```swift
struct Todo: Identifiable {
    let id: UUID
    let name: String
    let description: String
    let price: Double
    
    init(
        id: UUID = UUID(),
        name: String,
        description: String,
        price: Double
    ) {
        self.id = id
        self.name = name
        self.description = description
        self.price = price
    }
}
```

Vamos a necesitar representar el precio así que nos crearemos una extension.

```swift
extension Todo {
    var currency: String {
        let formatter = NumberFormatter()
        formatter.locale = Locale.current
        formatter.numberStyle = .currency
        if let currency = formatter.string(from: self.price as NSNumber) {
            return currency
        }
        return "XX.XX"
    }
}
```

Vamos a suponer que tenemos un servicio y nos devolverá un listado de todos.

```swift
struct ApiClient {
    var todos: () async throws -> [Todo]
}

extension ApiClient {
    static var live: Self {
        fatalError()
    }
}
```

Crearemos una instancia mock, que devolverá unos valores y tendrá un delay de unos segundos.

```swift
extension ApiClient {
    static var mock = Self(
        todos: {
            try await Task.sleep(nanoseconds: NSEC_PER_SEC * 2)
            return [
                .init(name: "name 1", description: "description 1", price: 1),
                .init(name: "name 2", description: "description 2", price: 1),
                .init(name: "name 3", description: "description 3", price: 1)
            ]
        }
    )
}
```

Como usaremos MVVM, necesitaremos un viewModel:

```swift
class AppViewModel: ObservableObject {
    @Published var todos: [Todo] = []
    
    let apiClient: ApiClient = .mock
    
    func onAppear() {
        Task { @MainActor in
            self.todos = try await self.apiClient.todos()
        }
    }
}
```

Y, por último, actualizaremos la vista y la vista para cada todo.

```swift
struct ContentView: View {
    @ObservedObject var viewModel = AppViewModel()
    
    var body: some View {
        List {
            ForEach(self.viewModel.todos) { todo in
                TodoView(todo: todo)
            }
        }
        .onAppear(perform: self.viewModel.onAppear)
    }
}
```

```swift
struct TodoView: View {
    let todo: Todo
    
    var body: some View {
        HStack(alignment: .top, spacing: 16) {
            Text(self.todo.currency)
            VStack(alignment: .leading, spacing: 8) {
                Text(self.todo.name)
                Text(self.todo.description)
            }
        }
    }
}
```

El código anterior, funciona. Pero, el funcionamiento, es un tanto aburrido. La pantalla permanecerá en blanco, hasta que la llamada termine y devuelva el resultado en forma de listado de todos.

Podríamos estar tentados de agregar más estados a la pantalla en forma de un isLoading mezclado con un ProgressView, por ejemplo. Luego, podríamos agregar un hasError y, finalmente, pondriamos un isEmpty.

Al final estamos, incorporando casuísticas a una pantalla añadiendo más y más estados como variables y incorporando estados imposibles, que no los vamos a necesitar.

Desde el punto de vista del rendimiento, incorporar a una pantalla un número elevado de Published es peligroso ya que son elementos que están constantemente escuchando cualquier cambio para reejecutar el body de la view.

Y, aquí, de nuevo, aparecen los Enums al rescate. La solución pasa por gestionar todos los estados a través de una variable que, de momento, me parece interesante, DataSource, dado que la fuente de los datos, no necesariamente va a venir de una única fuente. El nombre me recuerda al DataSource de una UITableView, así que el nombre me parece adecuado.

Entonces, vayamos ahí.

## DataSource, redacted

Lo primero, borra el Published que tenemos en el AppViewModel y añade lo siguiente.

```swift
@Published var dataSource: DataSource = .redacted
    
enum DataSource {
    case redacted
    case list([Todo])
}
```

Vamos a tratar dos estados. Cuando la pantalla está cargando y cuando ya tiene los elementos recibidos desde la llamada a la api.

Actualiza la función onAppear:

```swift
func onAppear() {
    self.dataSource = .redacted
    
    Task { @MainActor in
        self.dataSource = .list(try await self.apiClient.todos())
    }
}
```

Para mostrar la pantalla en modo redacted, vamos a necesitar un todo especial.

```swift
extension Todo {
    static var redacted = Self(
        name: "name name",
        description: "description description",
        price: 11.11
    )
}
```

Y, luego, necesitaremos una animación interesante. Está sacada de <a href="https://github.com/markiv/SwiftUI-Shimmer">aquí</a>

```swift
public struct Shimmer: ViewModifier {
    @State private var phase: CGFloat = 0
    var duration = 1.5
    var bounce = false

    public func body(content: Content) -> some View {
        content
            .modifier(AnimatedMask(phase: phase).animation(
                Animation.linear(duration: duration)
                    .repeatForever(autoreverses: bounce)
            ))
            .onAppear { phase = 0.8 }
    }

    struct AnimatedMask: AnimatableModifier {
        var phase: CGFloat = 0

        var animatableData: CGFloat {
            get { phase }
            set { phase = newValue }
        }

        func body(content: Content) -> some View {
            content
                .mask(
                    GradientMask(phase: phase)
                        .scaleEffect(3)
                )
        }
    }

    struct GradientMask: View {
        let phase: CGFloat
        let centerColor = Color.black
        let edgeColor = Color.black.opacity(0.3)
        
        var body: some View {
            LinearGradient(
                gradient: .init(
                    stops: [
                        .init(color: edgeColor, location: phase),
                        .init(color: centerColor, location: phase + 0.1),
                        .init(color: edgeColor, location: phase + 0.2)
                    ]
                ),
                startPoint: .topLeading,
                endPoint: .bottomTrailing
            )
        }
    }
}

public extension View {
    @ViewBuilder func shimmering(
        duration: Double = 1.5,
        bounce: Bool = false
    ) -> some View {
        modifier(
            Shimmer(
                duration: duration,
                bounce: bounce
            )
        )
    }
}
```

Por último, iremos a la vista y haremos pattern matching sobre el datasource.

```swift
struct ContentView: View {
    @ObservedObject var viewModel = AppViewModel()
    
    var body: some View {
        Group {
            if case .redacted = self.viewModel.dataSource {
                List {
                    ForEach(0..<10) { _ in
                        TodoView(todo: Todo.redacted)
                            .redacted(reason: .placeholder)
                            .shimmering()
                    }
                }
            }
            if case let .list(todos) = self.viewModel.dataSource {
                List {
                    ForEach(todos) { todo in
                        TodoView(todo: todo)
                    }
                }
            }
        }
        .onAppear(perform: self.viewModel.onAppear)
    }
}
```

## CasePaths al rescate

Si no quisieramos utilizar CasePaths, pues listo. Ya habríamos terminado, pero aquí estamos viendo que merece la pena utilizarlos ya que, si estamos usando la navegación con CasePaths, pues es una pena no homogeneizarlo todo.

```swift
IfCaseLet(
    dataSource: <#T##_#>,
    case: <#T##CasePath<_, _>#>,
    content: <#T##(_) -> _#>
)
```

Nuestra idea sería utilizar esta forma. Le pasamos el enum, le pasamos el case, sacamos el pattern matching y el contenido construye la vista junto con el pattern matching. El IfCaseLet original viene de <a href="https://www.pointfree.co/collections/swiftui/navigation">la serie de navegación de SwiftUI</a>.

```swift
struct IfCaseLet<Enum, Case, Content>: View where Content: View {
    let dataSource: Enum
    let casePath: CasePath<Enum, Case>
    let content: (Case) -> Content
    
    init(
        dataSource: Enum,
        case casePath: CasePath<Enum, Case>,
        @ViewBuilder content: @escaping (Case) -> Content
    ) {
        self.dataSource = dataSource
        self.casePath = casePath
        self.content = content
    }
    
    var body: some View {
        if let `case` = self.casePath.extract(from: self.dataSource) {
            self.content(`case`)
        }
    }
}
```

Ya podemos actualizar la vista.

```swift
IfCaseLet(
    dataSource: self.viewModel.dataSource,
    case: /AppViewModel.DataSource.redacted
) {
    List {
        ForEach(0..<10) { _ in
            TodoView(todo: Todo.redacted)
                .redacted(reason: .placeholder)
                .shimmering()
        }
    }
}
IfCaseLet(
    dataSource: self.viewModel.dataSource,
    case: /AppViewModel.DataSource.list
) { todos in
    List {
        ForEach(todos) { todo in
            TodoView(todo: todo)
        }
    }
}
```

Si quisieramos modularizar un poco, el resultado anterior, podríamos hacer:

```swift
Group {
    IfCaseLet(
        dataSource: self.viewModel.dataSource,
        case: /AppViewModel.DataSource.redacted,
        content: RedactedTodoView.init
    )
    IfCaseLet(
        dataSource: self.viewModel.dataSource,
        case: /AppViewModel.DataSource.list,
        content: ListTodoView.init(todos:)
    )
}
```

## Metiendo más estados

Vamos a meter el resto de los estados.

```swift
enum DataSource {
    case redacted
    case list([Todo])
    case error(Error)
    case empty
}
```

Para simular los estados, necesitamos dos mocks más.

```swift
enum DomainError: Error {
    case unknown
}

extension ApiClient {
    static var empty = Self(
        todos: {
            try await Task.sleep(nanoseconds: NSEC_PER_SEC * 2)
            return []
        }
    )
    
    static var error = Self(
        todos: {
            try await Task.sleep(nanoseconds: NSEC_PER_SEC * 2)
            throw DomainError.unknown
        }
    )
}
```

Actualizaremos la función onAppear del AppViewModel. Ahora, tenemos que tratar el error, haremos un do catch. Y si el resultado es vacío, hay que asignar el case adecuado.

```swift
func onAppear() {
    self.dataSource = .redacted
    
    Task { @MainActor in
        do {
            let todos = try await self.apiClient.todos()
            self.dataSource = todos.isEmpty ? .empty : .list(todos)
        } catch let error {
            self.dataSource = .error(error)
        }
    }
}
```

Además necesitaremos dos vistas más.

```swift
struct ErrorTodoView: View {
    var action: () -> Void
    
    var body: some View {
        VStack {
            Text("Hubo un error")
            Button(
                action: self.action,
                label: { Text("Reintentar") }
            )
        }
    }
}

struct EmptyTodoView: View {
    var body: some View {
        Text("No hay todos")
    }
}
```

Finalmente, actualizaremos la vista, agregando los dos estados. Como, en el caso del error, hay un evento, necesitamos implementar el content para conectar la acción del botón con el viewModel y así mantener el Unidirectional Flow.

```swift
IfCaseLet(
    dataSource: self.viewModel.dataSource,
    case: /AppViewModel.DataSource.error
) { _ in
    ErrorTodoView(action: self.viewModel.onAppear)
}
IfCaseLet(
    dataSource: self.viewModel.dataSource,
    case: /AppViewModel.DataSource.empty,
    content: EmptyTodoView.init
)
```

![yay](/assets/img/datasource.gif)

## Resultado y reflexión

El resultado es más que interesante. Hemos podido controlar 4 estados de una forma más o menos escalable. Además, conseguimos controlar la complejidad accidental no agregando casos erroneos o imposibles.

Tenemos un único Published para n estados posibles. Esto simplifica mucho el testing, tanto que es hasta trivial.

Sin embargo, para cada n estados, vamos a necesitar codificar n vistas en la View, lo que hace un poco engorroso. Quizás, habría que estudiar la forma de encapsular el case con el IfCaseLet y no perder la unidireccionalidad en el caso de que un estado tenga acciones, por ejemplo, pensemos en un formulario.

Seguiremos investigando.