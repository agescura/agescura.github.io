---
layout: default
---

# SwiftUI, Navegación y The Composable Architecture. Primera parte, alertas.

Antes de empezar a leer este artículo, recomiendo ver la serie de navegación de SwiftUI. Además, no está familiarizado con The Composable Architecture, mejor acceder al repositorio y visionar los Case Studies que hay; son muy buenos, yo aprendí mucho con ellos!

Por resumir un poco, la solución que ofrece Apple para la navegación en SwiftUI ofrece muchas lagunas. Qué ocurre cuando en una pantalla hay muchas opciones para navegar? Imaginemos que tenemos una pantalla con una alerta para cuando no tenemos conexión de internet, otra alerta para realizar una acción de peligro, un sheet para cuando hay un error en la llamada de internet, una navegación hacia un detalle, etc, etc.

La representación de estos estados en SwiftUI, se haría de una forma similar a esta.

```swift
@Published var alertNoInternet = false
@Published var alertRemoveItem = false
@Published var sheetRequestError = false
@Published var navigateToDetail = false
```

Pero, haciendo un mapeado de este estilo, estamos incrementando la complejidad accidental de nuestro proyecto añadiendo estados imposibles. ¿Qué ocurriría si tuvieramos un estado así?

```swift
@Published var alertNoInternet = true
@Published var alertRemoveItem = true
@Published var sheetRequestError = true
@Published var navigateToDetail = true
```

Nota: En nuevas versiones en SwiftUI, se prefiere utilizar un Binding de un objeto, de forma que si el objeto existe, se lanza la navegación y en caso contrario, se dismisea la pantalla. Esta forma, aunque se simplifica un poco el código porque ya no hace falta controlar la navegación con dos variables, continua siendo engorrosa para el caso de que una pantalla tenga muchas posibilidades de navegación.

Seguramente el comportamiento de la aplicación no sería el esperado. Pues bien, la solución que nos ofrece Pointfree en la serie de navegación con SwiftUI es mapear la pantalla justamente con los estados suficientes.

```swift
@Published var route: Route? = nil

enum Route {
    case alertNoInternet
    case alertRemoveItem
    case sheetRequestError(RequestErrorViewModel)
    case navigateToDetail(DetailViewModel)
}
```

Este planteamiento es sencillo y elegante. Además, permite la construcción de un sistema de deeplink de forma sencilla y, más importante aún, modularizable.

Sin embargo, no existe, aún, una solución similar en The Composable Architecture. Pero, mientras estamos esperando, con muchas ganas, que aparezcan ya la integración de la navegación con el TCA, escribo este artículo ofreciendo una solución a este problema.

## Todo empieza con un proyecto de ejemplo.

Vamos a empezar a diseñar un proyecto sencillo. 

* Tendremos un listado de números que inicialmente estará vacío.
* Tendremos la posibilidad de añadir un número dándole al botón + de la esquina superior derecha.
* Podemos hacer swipe derecha para eliminar un elemento del listado. Se mostrará una alerta avisando de que estamos a punto de eliminar el elemento.
* Podemos hacer swipe izquiera para poder cambiar el background a uno de los colores disponibles. Se mostrará un action sheet o confirmation dialog (como se llama en SwiftUI).

## The Composable Architecture: Basics

Empezaremos creando un proyecto, yo lo he llamado TCAlertsInvestigation y le añadiremos The Composable Architecture como dependencia.

Como es un primer proyecto de investigación, no voy a modularizar nada, ni a crear ninguna librería. Es una prueba nada más.

Empezaremos por construir el listado de items. En TCA, empezamos por el estado de la aplicación.

```swift
struct AppState: Equatable {
    var items: IdentifiedArrayOf<Item> = []

    struct Item: Identifiable, Equatable {
        let value: Int
    
        var id: Int {
            self.value
        }
    }
}
```

Nota: IdentifiedArrayOf es un array con un identificar UUID, que nos permite realizar búsquedas más eficientemente que con Array.

El siguiente objeto son las acciones que tendrá nuestra aplicación. De momento, solo queremos añadir un número al listado.

```swift
enum AppAction: Equatable {
    case addItemButtonTapped
}
```

El tercer objeto es el Environment. En este proyecto, de momento, no tenemos necesidad de hacer uso de el.

```swift
struct AppEnvironment {}
```

El cuarto objeto es el Reducer. Es el encargado de ejecutar las acciones.

```swift
let appReducer = Reducer<
AppState,
AppAction,
AppEnvironment
> { state, action, _ in
    switch action {
        
    case .addItemButtonTapped:
        state.items.append(.init(value: Int.random(in: 0...100)))
        return .none
    }
}
```

Finalmente, crearemos el Store y la vista AppView.

```swift
struct AppView: View {
    let store: Store<AppState, AppAction>
    
    var body: some View {
        WithViewStore(self.store) { viewStore in
            NavigationView {
                List {
                    ForEach(viewStore.items) { item in
                        Text("\(item.value)")
                    }
                }
                .navigationTitle("Items")
                .toolbar {
                    ToolbarItem(placement: .navigationBarTrailing) {
                        Button(
                            action: { viewStore.send(.addItemButtonTapped) },
                            label: { Image(systemName: "plus") }
                        )
                    }
                }
            }
        }
    }
}
```

Ahora, actualizamos tanto el preview como el programa principal.


```swift
AppView(
    store: .init(
        initialState: .init(),
        reducer: appReducer,
        environment: .init()
    )
)
```

## Route

Para añadir una alerta, primero tenemos que crear un Route en el AppState.

```swift
var route: Route? = nil

enum Route: Equatable {
    case removeItemAlert
}
```

Lo que nos gustaría es poder instanciar cualquier alerta desde un Reducer, con lo cual con un case sin pattern matching no sería suficiente. Necesitamos un objeto asociado al case de la alerta. Vamos a construirlo.

Una alerta tiene un titulo y un mensaje. El título es obligado pero el mensaje es opcional.

```swift
struct ActionViewModel: Equatable {
    let title: String
    let message: String?
}
```

De momento, nos quedamos con estos objetos. Más adelante, podremos generalizar el ActionViewModel con más información. Podemos actualizar el route.

```swift
case removeItemAlert(ActionViewModel)
```

Tenemos que añadir más acciones y actualizar el reducer.

```swift
case removeAlertButtonTapped(AppState.Item)
case removeButtonTapped(AppState.Item)
case dismissButtonTapped
```

```swift
case let .removeAlertButtonTapped(item):
    guard let item = state.items[id: item.id] else { return .none }
        
    state.route = .removeItemAlert(
        .init(
            title: "Cambiar estilo",
            message: "¿Estás a punto de cambiar el estilo \(item.value)?"
        )
    )
    return .none
        
    case let .removeButtonTapped(item):
    state.items.remove(id: item.id)
    return .none
        
    case .dismissButtonTapped:
    state.route = nil
    return .none
```

Nota: Este artículo se centra en la navegación pero, para hacer bien esta parte, quizás deberíamos derivar el listado y usar un ForEachStore, de esta forma, no habría necesidad de utilizar acciones que tengan como parámetro el elemento al que queremos borrar.

## CasePaths, Bindings and Alerts

Ahora tenemos que recuperar los métodos de la serie de navigation. Necesitamos la función que une el binding del Route con su case para obtener el binding del pattern matching.

```swift
extension Binding {
    func `case`<Enum, Case>(
        _ casePath: CasePath<Enum, Case>
    ) -> Binding<Case?> where Value == Enum? {
        Binding<Case?>(
            get: {
                guard
                    let wrappedValue = self.wrappedValue,
                    let `case` = casePath.extract(from: wrappedValue)
                else { return nil }
                return `case`
            },
            set: { `case` in
                if let `case` = `case` {
                    self.wrappedValue = casePath.embed(`case`)
                } else {
                    self.wrappedValue = nil
                }
            }
        )
    }
}
```

Ahora tenemos que adaptar la versión de la alert para que sea compatible con el ActionViewModel que hemos creado antes. 

* El primer parámetro prefiero llamarlo route ya que hace referencia al route de la pantalla, en vez, de unwrap. Por otro lado, no es necesario usar un Binding, así que la entrada es un optional Enum.
* El case será el valor que tenemos del route para la alerta.
* Las acciones se quedan como estan pero devolverá un objeto de tipo ActionViewModel
* Por último, necesitamos en TCA devolver el onDismiss para que se ejecute la accion de dismisear en el reducer.

```swift
extension View {
    func alert<A: View, Enum, Case>(
        route optionalValue: Enum?,
        case casePath: CasePath<Enum, Case>,
        @ViewBuilder actions: @escaping (ActionViewModel) -> A,
        onDismiss: (() -> Void)? = nil
    ) -> some View {
        let pattern = Binding.constant(optionalValue).case(casePath)
        let presenting = pattern.wrappedValue as? ActionViewModel
        
        return self.alert(
            Text(presenting?.title ?? ""),
            isPresented: .init(
                get: { pattern.wrappedValue != nil },
                set: { isPresented in
                    if !isPresented {
                        onDismiss?()
                    }
                }
            ),
            presenting: presenting,
            actions: actions,
            message: {
                if let message = $0.message {
                    Text(message)
                }
            }
        )
    }
}
```

En la vista ahora añadiremos un swipeAction.

```swift
Text("\(item.value)")
    .swipeActions(
        edge: .trailing,
        allowsFullSwipe: true
    ) {
        Button(
            action: { viewStore.send(.removeAlertButtonTapped(item)) },
            label: { Text("Remove") }
        )
        .tint(.red)
    }
```

Pero tenemos ahora un problema. Necesitamos guardar el item, de forma temporal, para luego ejecutar la acción en la alerta. Actualizaremos el AppState.

```swift
var item: Item?
```

Y ya, por último, agregaremos la alerta, debajo del toolbar.

```swift
.alert(
    route: viewStore.route,
    case: /AppState.Route.removeItemAlert,
    actions: { _ in
        Button(
            "Borrar",
            role: .destructive,
            action: { viewStore.send(.removeButtonTapped) }
        )
    },
    onDismiss: { viewStore.send(.dismissButtonTapped) }
)
```

Ya tenemos implementado el borrado de un elemento de una lista, usando una alerta con un enum Route dentro de TCA.

Principales problemas que tenemos ahora mismo.

* En el AppState, tenemos una variable temporal que usamos para borrar un elemento.
* Si queremos usar varias alertas, en una misma pantalla, tendríamos que crear varios cases, tantos como tipos de alerta tengamos.

En general, tenemos acoplado el alert a la vista. Siguiente paso, desacoplarlo, de forma que, desde el reducer podamos crear una alerta. Esto va a ser especialmente importante con el confirmationDialog, ya que este componente, si suele ser más dinámico.

## Desacoplamiento de las actions en el ActionViewModel

Queremos añadir los botones en el ActionViewModel para poder instanciar una alerta desde el reducer. En una primera iteración, añadiremos un array de botones con un título y un rol, y nos dejaremos la acción del botón para el final.

```swift
struct ActionViewModel: Equatable {
    let title: String
    let message: String?
    let buttons: [ActionViewModel.Button]
    
    struct Button: Hashable {
        let id = UUID()
        let title: String
        let role: ButtonRole?
        
        init(
            _ title: String,
            role: ButtonRole? = nil
        ) {
            self.title = title
            self.role = role
        }
        
        func hash(into hasher: inout Hasher) {
            hasher.combine(self.id)
        }
        
        static func == (lhs: Self, rhs: Self) -> Bool {
            lhs.id == rhs.id
        }
    }
}
```

Ahora actualizaremos la actualización del route dentro del reducer

```swift
state.route = .removeItemAlert(
    .init(
        title: "Cambiar estilo",
        message: "¿Estás a punto de cambiar el estilo \(item.value)?",
        buttons: [
            .init("Delete", role: .destructive)
        ]
    )
)
```

Y, por último, la alerta, para que lea el botón del route.

```swift
.alert(
    route: viewStore.route,
    case: /AppState.Route.removeItemAlert,
    actions: { actionViewModel in
        Button(
            actionViewModel.buttons[0].title,
            role: actionViewModel.buttons[0].role,
            action: { viewStore.send(.removeButtonTapped) }
        )
    },
    onDismiss: { viewStore.send(.dismissButtonTapped) }
)
```

Sin embargo, buttons es un array, así que bien podríamos poner ahí un ForEach. Algo así,

```swift
ForEach(actionViewModel.buttons, id: \.self) { button in
    Button(
        button.title,
        role: button.role
    ) {
        viewStore.send(.removeButtonTapped)
    }
}
```

Pero seguimos teniendo, la acción hardcodeada en la vista.

## Generalizar las acciones en la vista

Las acciones del ActionViewModel van a ser las acciones del TCA, por lo tanto, vamos a necesitar un genérico. Una posible solución es esta.

```swift
struct ActionViewModel<Action>: Equatable {
    let title: String
    let message: String?
    let buttons: [ActionViewModel.Button<Action>]
    
    init(
        _ title: String,
        message: String? = nil,
        buttons: [ActionViewModel.Button<Action>]
    ) {
        self.title = title
        self.message = message
        self.buttons = buttons
    }
    
    struct Button<Action>: Hashable {
        let id = UUID()
        let title: String
        let role: ButtonRole?
        let action: Action
        
        init(
            _ title: String,
            role: ButtonRole? = nil,
            action: Action
        ) {
            self.title = title
            self.role = role
            self.action = action
        }
        
        func hash(into hasher: inout Hasher) {
            hasher.combine(self.id)
        }
        
        static func == (lhs: Self, rhs: Self) -> Bool {
            lhs.id == rhs.id
        }
    }
}
```

Ahora, tenemos que adaptar el código a este nuevo tipo. Lo primero será actualizar el tipo ActionViewModel a ActionViewModel<Value>.
Después, como estamos usando un genérico, tenemos que usar su tipo y pasarselo como parámetro.

```swift
extension View {
    func alert<A: View, Enum, Case, Action>(
        route optionalValue: Enum?,
        case casePath: CasePath<Enum, Case>,
        action: Action.Type,
        @ViewBuilder actions: @escaping (ActionViewModel<Action>) -> A,
        onDismiss: (() -> Void)? = nil
    ) -> some View {
        let pattern = Binding.constant(optionalValue).case(casePath)
        let presenting = pattern.wrappedValue as? ActionViewModel<Action>
        
        return self.alert(
            Text(presenting?.title ?? ""),
            isPresented: .init(
                get: { pattern.wrappedValue != nil },
                set: { isPresented in
                    if !isPresented {
                        onDismiss?()
                    }
                }
            ),
            presenting: presenting,
            actions: actions,
            message: {
                if let message = $0.message {
                    Text(message)
                }
            }
        )
    }
}
```

Resolvemos errores.

* En la definición del route.

```swift
case removeItemAlert(ActionViewModel<AppAction>)
```

* Cuando usamos el route para asignar la alerta.

```swift
state.route = .removeItemAlert(
    .init(
        "Cambiar estilo",
        message: "¿Estás a punto de cambiar el estilo \(item.value)?",
        buttons: [
            .init("Delete", role: .destructive, action: .removeButtonTapped)
        ]
    )
)
```

* Por último, en la alerta.

```swift
.alert(
    route: viewStore.route,
    case: /AppState.Route.removeItemAlert,
    action: AppAction.self,
    actions: { actionViewModel in
        ForEach(actionViewModel.buttons, id: \.self) { button in
            Button(
                button.title,
                role: button.role
            ) {
                viewStore.send(button.action)
            }
        }
    },
    onDismiss: { viewStore.send(.dismissButtonTapped) }
)
```

Podemos prescindir de pasar explicitamente la acción si en la clausura de las acciones hacemos explícito su tipo.

```swift
.alert(
    route: viewStore.route,
    case: /AppState.Route.removeItemAlert,
    actions: { (actionViewModel: ActionViewModel<AppAction>) in
        ForEach(actionViewModel.buttons, id: \.self) { button in
            Button(
                button.title,
                role: button.role
            ) {
                viewStore.send(button.action)
            }
        }
    },
    onDismiss: { viewStore.send(.dismissButtonTapped) }
)
```

Y eliminamos el parámetro action del alert.

```swift
action: Action.Type,
```

El código que tenemos en la vista, ya está desacoplado. Sin embargo, este codigo siempre será el mismo. Entonces, lo que vamos a hacer, será trasladar la clausura, a la implementación de la alerta.

Si copiamos la clausura de las actions a la alerta, nos estamos llevando la función send. Tendremos que añadir un parámetro de entrada send para enviar las acciones. Ya lo tendríamos.

```swift
extension View {
    func alert<Enum, Case, Action>(
        route optionalValue: Enum?,
        case casePath: CasePath<Enum, Case>,
        send: @escaping (Action) -> Void,
        onDismiss: (() -> Void)? = nil
    ) -> some View {
        let pattern = Binding.constant(optionalValue).case(casePath)
        let presenting = pattern.wrappedValue as? ActionViewModel<Action>
        
        return self.alert(
            Text(presenting?.title ?? ""),
            isPresented: .init(
                get: { pattern.wrappedValue != nil },
                set: { isPresented in
                    if !isPresented {
                        onDismiss?()
                    }
                }
            ),
            presenting: presenting,
            actions: { (alertViewModel: ActionViewModel<Action>) in
                ForEach(alertViewModel.buttons, id: \.self) { button in
                    Button(
                        button.title,
                        role: button.role
                    ) {
                        send(button.action)
                    }
                }
            },
            message: {
                if let message = $0.message {
                    Text(message)
                }
            }
        )
    }
}
```

Y su uso, en la vista, tendría esta forma.

```swift
.alert(
    route: viewStore.route,
    case: /AppState.Route.removeItemAlert,
    send: { viewStore.send($0) },
    onDismiss: { viewStore.send(.dismissButtonTapped) }
)
```

## Confirmation Dialog

El Confirmation Dialog es primo hermano del Alert. Así que iremos rápido.

```swift
extension View {
    func confirmationDialog<Enum, Case, Action>(
        route optionalValue: Enum?,
        case casePath: CasePath<Enum, Case>,
        titleVisibility: Visibility = .automatic,
        send: @escaping (Action) -> Void,
        onDismiss: (() -> Void)? = nil
    ) -> some View {
        let pattern = Binding.constant(optionalValue).case(casePath)
        let presenting = pattern.wrappedValue as? ActionViewModel<Action>
        
        return self.confirmationDialog(
            Text(presenting?.title ?? ""),
            isPresented: .init(
                get: { pattern.wrappedValue != nil },
                set: { isPresented in
                    if !isPresented {
                        onDismiss?()
                    }
                }
            ),
            titleVisibility: .visible,
            presenting: presenting,
            actions: { (alertViewModel: ActionViewModel<Action>) in
                ForEach(alertViewModel.buttons, id: \.self) { button in
                    Button(
                        button.title,
                        role: button.role
                    ) {
                        send(button.action)
                    }
                }
            },
            message: {
                if let message = $0.message {
                    Text(message)
                }
            }
        )
    }
}
```

Y ahora, actualizaremos el proyecto para hacer un cambio en el listado.

Añade la definición de estilo siguiente:

```swift
enum Style: CaseIterable {
    case none
    case yellow
    case blue
    case green
    
    var color: Color {
        switch self {
        case .none:
            return .white
        case .yellow:
            return .yellow
        case .blue:
            return .blue
        case .green:
            return .green
        }
    }
}
```

Actualiza el elemento Item:

```swift
struct Item: Identifiable, Equatable {
        let value: Int
        var style: Style = .none
        
        ....
}
```

Necesitamos un nuevo case en el Route para añadir el nuevo ConfirmationDialog.

```swift
enum Route: Equatable {
        case removeItemAlert(ActionViewModel<AppAction>)
        case changeStyleConfirmationDialog(ActionViewModel<AppAction>)
        
        static func == (lhs: Self, rhs: Self) -> Bool {
            switch (lhs, rhs) {
            case let (.removeItemAlert(lhsAlert), .removeItemAlert(rhsAlert)):
                return lhsAlert == rhsAlert
            case let (.changeStyleConfirmationDialog(lhsAlert), .changeStyleConfirmationDialog(rhsAlert)):
                return lhsAlert == rhsAlert
            default:
                return false
            }
        }
    }
```


Añadiremos dos nuevas acciones al AppAction. 

```swift
case changeStyleConfirmationDialogButtonTapped(AppState.Item)
case change(style: AppState.Item.Style, item: AppState.Item)
```

Y, implementaremos su lógica dentro del Reducer:

```swift
case let .changeStyleConfirmationDialogButtonTapped(item):
        state.route = .changeStyleConfirmationDialog(
            .init(
                "Eliminar item",
                message: "¿Estás a punto de eliminar el item \(item.value)?",
                buttons: AppState.Item.Style.allCases.map { style in
                        .init("\(style)", action: .change(style: style, item: item))
                }
            )
        )
        return .none
            
    case let .change(style: style, item: item):
        state.items[id: item.id]?.style = style
        return .none
    }
```

Ahora ya solo nos queda actualizar la vista.

```swift
List {
    ForEach(viewStore.items) { item in
        Text("\(item.value)")
            .listRowBackground(item.style.color)
            .swipeActions(
                edge: .trailing,
                allowsFullSwipe: true
            ) {
                Button(
                    action: { viewStore.send(.removeAlertButtonTapped(item)) },
                    label: { Text("Remove") }
                )
                .tint(.red)
            }
            .swipeActions(edge: .leading, allowsFullSwipe: false) {
                Button(
                    action: { viewStore.send(.changeStyleConfirmationDialogButtonTapped(item)) },
                    label: { Text("Change color") }
                )
                .tint(.orange)
            }
    }
}
```

y el uso del confirmation dialog es similar al de la alerta.

```swift
.alert(
    route: viewStore.route,
    case: /AppState.Route.removeItemAlert,
    send: { viewStore.send($0) },
    onDismiss: { viewStore.send(.dismissButtonTapped) }
)
.confirmationDialog(
    route: viewStore.route,
    case: /AppState.Route.changeStyleConfirmationDialog,
    titleVisibility: .visible,
    send: { viewStore.send($0) },
    onDismiss: { viewStore.send(.dismissButtonTapped) }
)
```

Próxima parte, hablaremos de los sheets y de los popover. Van a tener una forma parecida a los alerts:

```swift
.sheet(
    route: viewStore.route,
    case: /AppState.Route.addItemSheet,
    onDismiss: { viewStore.send(.dismissButtonTapped) },
    content: { $addItemState in
        AddItemView(
            store: self.store.scope(
                state: { _ in addItemState },
                action: AppAction.addItem
            )
        )
    }
)
```

Pero esta vez, como cualquier navegación en TCA, tendremos que ver la forma de derivar dominios a través del Route.