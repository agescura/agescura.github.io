---
layout: default
---

# SwiftUI, Navegación y The Composable Architecture. Segunda parte, sheets.

En el <a href="https://agescura.github.io/composable-architecture-alerts.html">artículo anterior</a>., se habló de alerts y confirmationDialogs (los antiguos ActionSheets). En el, se tomó la decisión de utilizar la sobrecarga de las alertas y de los confirmation dialog, de la siguiente forma.

La signatura de una alerta es la siguiente:

```swift
.alert(
    route: <#T##Enum?#>,
    case: <#T##CasePath<Enum, Case>#>,
    send: <#T##(Action) -> Void#>,
    onDismiss: <#T##(() -> Void)?#>
)
```

La signatura de un confirmation dialog es la siguiente:

```swift
.confirmationDialog(
    route: <#T##Enum?#>,
    case: <#T##CasePath<Enum, Case>#>,
    titleVisibility: <#T##Visibility#>,
    send: <#T##(Action) -> Void#>,
    onDismiss: <#T##(() -> Void)?#>
)
```

Y, en este artículo, lo que buscamos es una signatura para un sheet (y, también para su hermano gemelo, el popover para los iPads).

```swift
.sheet(
    route: <#T##Enum?#>,
    case: <#T##CasePath<Enum, Case>#>,
    onDismiss: <#T##(() -> Void)?#>,
    content: <#T##(Case) -> Content#>
)
```

## Un sheet a la antigua

Empezamos nuestra investigación siguiendo los ejemplos en los casos de estudio que nos ofrece Composable Architecture.

Para ello, iremos rápido, crearemos un componente nuevo AddItem.

```swift
struct AddItemState: Equatable, Identifiable {
    var item: Item
    
    var id: UUID {
        self.item.id
    }
}

enum AddItemAction: Equatable {
    case change(style: Style)
    case incrementButtonTapped
    case decrementButtonTapped
    case generateRandomValueButtonTapped
    case saveButtonTapped
}

struct AddItemEnvironment {}

let addItemReducer = Reducer<
AddItemState,
AddItemAction,
AddItemEnvironment
> { state, action, _ in
    switch action {
    case let .change(style: style):
        state.item.style = style
        return .none
        
    case .incrementButtonTapped:
        state.item.value += 1
        return .none
        
    case .decrementButtonTapped:
        state.item.value -= 1
        return .none
        
    case .generateRandomValueButtonTapped:
        state.item.value = Int.random(in: 0...100)
        return .none
        
    case .saveButtonTapped:
        return .none
    }
}

struct AddItemView: View {
    let store: Store<AddItemState, AddItemAction>
    
    var body: some View {
        WithViewStore(self.store) { viewStore in
            ZStack {
                viewStore.item.style.color.edgesIgnoringSafeArea(.all)
                
                VStack(spacing: 16) {
                    HStack(spacing: 16) {
                        ForEach(Style.allCases, id: \.self) { style in
                            Button(
                                action: { viewStore.send(.change(style: style)) },
                                label: {
                                    Text(.init("\(style)"))
                                        .minimumScaleFactor(0.1)
                                        .lineLimit(1)
                                        .whiteBox()
                                }
                            )
                        }
                    }
                    HStack {
                        Button(
                            action: { viewStore.send(.decrementButtonTapped) },
                            label: {
                                Text("-")
                                    .whiteBox()
                            }
                        )
                        Text("\(viewStore.item.value)")
                            .whiteBox()
                        Button(
                            action: { viewStore.send(.incrementButtonTapped) },
                            label: {
                                Text("+")
                                    .whiteBox()
                            }
                        )
                    }
                    Button(
                        action: { viewStore.send(.generateRandomValueButtonTapped) },
                        label: {
                            Text("Generate Random Value")
                                .whiteBox()
                        }
                    )
                    
                    Spacer()
                    
                    Button(
                        action: { viewStore.send(.saveButtonTapped) },
                        label: {
                            Text("Add Item")
                                .whiteBox()
                        }
                    )
                }
                .padding()
            }
        }
    }
}

struct WhiteBox: ViewModifier {
    func body(content: Content) -> some View {
        content
            .foregroundColor(.black)
            .frame(maxWidth: .infinity)
            .padding()
            .background(Color.white)
            .cornerRadius(8)
    }
}

extension View {
    func whiteBox() -> some View {
        modifier(WhiteBox())
    }
}

struct AddItemView_Previews: PreviewProvider {
    static var previews: some View {
        AddItemView(
            store: .init(
                initialState: .init(
                    item: .init()
                ),
                reducer: addItemReducer,
                environment: .init()
            )
        )
    }
}
```

* En el AddItemState definimos un objeto de tipo Item
* Crearemos diferentes acciones que nos van a permitir crear un objeto Item como queramos. Estas acciones van a ser locales, es decir, afectan a la pantalla. Salvo guardar el objeto. Esta acción será usada en el appReducer.
* La navegación no se va a definir dentro de este componente.

Tenemos un error en la definición del objeto Item, lo actualizaremos.

```swift
struct Item: Identifiable, Equatable {
    let id = UUID()
    var value: Int = 0
    var style: Style = .none
}
```

En el AppState, añadimos una referencia nula del estado de AddItemState.

```swift
var addItemState: AddItemState?
```

Haremos lo mismo en el AppAction y en e AppReducer. Además, necesitaremos una acción más, la encargada de realizar la acción de presentar el AddItem.

```swift
case addItem(AddItemAction)
case addItemSheet(isPresented: Bool)
```

```swift
case .addItem:
    return .none

case let .addItemSheet(isPresented: isPresented):
    state.addItemState = isPresented ? .init(item: .init()) : nil
    return .none 
```



Finalmente, añadiremos un sheet en la vista, de la siguiente forma:

```swift
.sheet(
    isPresented: viewStore.binding(
        get: \.addItemIsPresented,
        send: AppAction.addItemSheet(isPresented:)
    )
) {
    ????
} 
```

Pero necesitamos derivar la información del dominio global AppView hacia el dominio AddItemView. Para ello, necesitaremos utilizar dos funciones que son el núcleo de TCA. La función scope, permite mapear un dominio global a uno local. Y, la función pullback, permite actualizar un estado local hacia un estado global.

Empezamos por la función pullback, en el appReducer. Tenemos que combinar el appReducer con el addItemReducer, de forma que cuando se actualiza el addItemState o se ejecuta una addItemAction, éstas se sincronicen con el appReducer. La forma es la siguiente:

```swift
let appReducer: Reducer<
    AppState,
    AppAction,
    AppEnvironment
> = .combine(
    addItemReducer
        .optional()
        .pullback(
            state: \.addItemState,
            action: /AppAction.addItem,
            environment: { (_: AppEnvironment) in AddItemEnvironment() }
        ),
    
    .init { state, action, _ in
        switch action {
        
        case let .removeAlertButtonTapped(item):
        ...
        ...
        ...
    }
)
```

* la función combine es una higher order reducer que nos permite combinar multiples reducers.

* como hemos definido el addItemState como optional, necesitamos decirle al addItemReducer que es optional. Esto es especialmente importante, porque cuando implementemos el Route, no necesitaremos un .optional() sino "algo" que verifique el case sea el adecuado y no solamente hacer un unwrap a un estado opcional.

Para hacer scope, vamos a necesitar un objeto especial llamado IfLetStore, donde su comportamiento es similar a la del pullback.

```swift
.sheet(
    isPresented: viewStore.binding(
        get: \.addItemIsPresented,
        send: AppAction.addItemSheet(isPresented:)
    )
) {
    NavigationView {
        IfLetStore(
            self.store.scope(
                state: \.addItemState,
                action: AppAction.addItem
            )
        ) {
            AddItemView(store: $0)
                .navigationBarTitle("Add Item")
                .toolbar {
                    ToolbarItem.init(placement: .navigationBarLeading) {
                        Text("Hay \(viewStore.items.count) items")
                    }
                    ToolbarItem.init(placement: .navigationBarTrailing) {
                        Button(
                            action: { viewStore.send(.addItemSheet(isPresented: false)) },
                            label: {
                                Image(systemName: "xmark")
                                    .foregroundColor(.black)
                            }
                        )
                    }
                }
        }
    }
}
```

En el enum Style, para que se vea un poquito mejor, pongamos en el case .none un gris casi transparente.

```swift
case .none:
    return .gray.opacity(0.1)
```

El botón de guardar del AddItemView no hace nada. Vamos a poner esta funcionalidad en el AppView, de la siguiente forma:

```swift
case .addItem(.saveButtonTapped):
    guard let item = state.addItemState?.item else { return .none }
    
    state.items.append(item)
    state.addItemState = nil
    return .none

case .addItem:
    return .none
```

> **_NOTA_** En vez de asignar a nil el addItemState podríamos lanzar un side effect hacia la action que se responsabiliza de la navegación.

```swift
case .addItem(.saveButtonTapped):
    guard let item = state.addItemState?.item else { return .none }
    
    state.items.append(item)
    return Effect(value: .addItemSheet(isPresented: false))
```

Pero, en realidad, el problema de fondo lo tenemos, con el modelo de datos.

```swift
var route: Route? = nil
var addItemState: AddItemState?
var addItemIsPresented: Bool { self.addItemState != nil }
```

Estamos usando un Route para las alertas y los confirmation dialogs, pero, sin embargo, necesitamos, hasta dos variables para gestionar un sheet.

En el Route, añadiremos un nuevo case.

```swift
enum Route: Equatable {
    case removeItemAlert(ActionViewModel<AppAction>)
    case changeStyleConfirmationDialog(ActionViewModel<AppAction>)
    case addItemSheet(AddItemState)
    
    static func == (lhs: Self, rhs: Self) -> Bool {
        switch (lhs, rhs) {
        case let (.removeItemAlert(lhsAlert), .removeItemAlert(rhsAlert)):
            return lhsAlert == rhsAlert
        case let (.changeStyleConfirmationDialog(lhsAlert), .changeStyleConfirmationDialog(rhsAlert)):
            return lhsAlert == rhsAlert
        case let (.addItemSheet(lhsViewModel), .addItemSheet(rhsViewModel)):
            return lhsViewModel == rhsViewModel
        default:
            return false
        }
    }
}
```

Pero, para poder derivar, el dominio, con TCA solo tenemos .optional(). Entonces, necesitamos actualizar la property addItemState.

```swift
var addItemState: AddItemState? {
    get {
        guard case let .addItemSheet(viewModel) = self.route else { return nil }
        return viewModel
    }
    set {
        guard let newValue = newValue else { return }
        self.route = .addItemSheet(newValue)
    }
}
```

Esta vez, addItemState se genera gracias al route. Ahora, actualiza la acción addItemSheet(isPresented: Bool), por esta:

```swift
extension View {
    func sheet<Enum, Case, Content>(
        route optionalValue: Enum?,
        case casePath: CasePath<Enum, Case>,
        onDismiss: (() -> Void)? = nil,
        @ViewBuilder content: @escaping (Case) -> Content
    ) -> some View where Case: Identifiable, Content: View {
        let pattern = Binding.constant(optionalValue).case(casePath)
        
        return self.sheet(
            isPresented: .init(
                get: { pattern.wrappedValue != nil },
                set: { _ in }
            ),
            onDismiss: onDismiss
        ) {
            if let value = pattern.wrappedValue {
                content(value)
            }
        }
    }
}

extension View {
    func popover<Enum, Case, Content>(
        route optionalValue: Enum?,
        case casePath: CasePath<Enum, Case>,
        onDismiss: (() -> Void)? = nil,
        @ViewBuilder content: @escaping (Case) -> Content
    ) -> some View where Case: Identifiable, Content: View {
        let pattern = Binding.constant(optionalValue).case(casePath)
        
        return self.popover(
            isPresented: .init(
                get: { pattern.wrappedValue != nil },
                set: { isPresented in
                    onDismiss?()
                }
            )
        ) {
            if let value = pattern.wrappedValue {
                content(value)
            }
        }
    }
}
```

En la vista, tendremos esto.

```swift
.sheet(
    route: viewStore.route,
    case: /AppState.Route.addItemSheet,
    onDismiss: { viewStore.send(.dismissButtonTapped) },
    content: { addItemState in
        NavigationView {
            AddItemView(
                store: self.store.scope(
                    state: { _ in addItemState },
                    action: AppAction.addItem
                )
            )
            .navigationBarTitle("Add Item")
            .toolbar {
                ToolbarItem.init(placement: .navigationBarLeading) {
                    Text("Hay \(viewStore.items.count) items")
                }
                ToolbarItem.init(placement: .navigationBarTrailing) {
                    Button(
                        action: { viewStore.send(.dismissButtonTapped) },
                        label: {
                            Image(systemName: "xmark")
                                .foregroundColor(.black)
                        }
                    )
                }
            }
        }
    }
)
```

Pues ya lo tendríamos, pero aquí tenemos varios casos de investigación pendientes.

## OptionalPaths

Para la construcción de un local domain para AddItemState, necesitamos crear una computed property.

```swift
var addItemState: AddItemState? {
    get {
        guard case let .addItemSheet(viewModel) = self.route else { return nil }
        return viewModel
    }
    set {
        guard let newValue = newValue else { return }
        self.route = .addItemSheet(newValue)
    }
}
```

Para cada case, esto sería, realmente, verboso y poco ergonómico. Una posible solución sería utilizar OptionalPaths. Se encuentran <a href="https://github.com/pointfreeco/isowords/blob/37102baec1d49574c596595e3c82985672762418/Sources/TcaHelpers/OptionalPaths.swift">aquí</a>, en la aplicación Isowords.

Esto no forma parte, aún de TCA, por lo que su uso, o algo similar, aparecerá en breve en TCA.

```swift
addItemReducer
    ._pullback(
        state: (\AppState.route).appending(path: /AppState.Route.addItemSheet),
        action: /AppAction.addItem,
        environment: { (_: AppEnvironment) in AddItemEnvironment() }
    ),
```

Ahora ya podemos borrar del AppState, la computed property addItemState.

Tendremos un error en el action saveButtonTapped.

```swift
case .addItem(.saveButtonTapped):
    guard case let .addItemSheet(addItemState) = state.route else { return .none }
    
    state.items.append(addItemState.item)
    state.route = nil
    return .none
```

La aplicación funciona exactamente igual que antes. Aquí entramos en la discusión de si es mejor utilizar OptionalPaths o crear una computed property por cada local domain que creemos.

## ¿Por que no usamos sheet(item:content:)?

Pues hecho.

```swift
extension View {
    func sheet<Enum, Case, Content>(
        route optionalValue: Enum?,
        case casePath: CasePath<Enum, Case>,
        onDismiss: (() -> Void)? = nil,
        @ViewBuilder content: @escaping (Case) -> Content
    ) -> some View where Case: Identifiable, Content: View {
        let pattern = Binding.constant(optionalValue).case(casePath)
        
        return self.sheet(
            item: .init(
                get: {
                    guard let item = pattern.wrappedValue else { return nil }
                    return item
                },
                set: { isPresented in
                    if isPresented == nil {
                        onDismiss?()
                    }
                }
            ),
            content: content
        )
    }
}

extension View {
    func popover<Enum, Case, Content>(
        route optionalValue: Enum?,
        case casePath: CasePath<Enum, Case>,
        onDismiss: (() -> Void)? = nil,
        attachmentAnchor: PopoverAttachmentAnchor = .rect(.bounds),
        arrowEdge: Edge = .top,
        @ViewBuilder content: @escaping (Case) -> Content
    ) -> some View where Case: Identifiable, Content: View {
        let pattern = Binding.constant(optionalValue).case(casePath)
        
        return self.popover(
            item: .init(
                get: {
                    guard let item = pattern.wrappedValue else { return nil }
                    return item
                },
                set: { isPresented in
                    if isPresented == nil {
                        onDismiss?()
                    }
                }
            ),
            attachmentAnchor: attachmentAnchor,
            arrowEdge: arrowEdge,
            content: content
        )
    }
}
```

## ¿Qué ocurre si queremos actualizar el GlobalState a través del LocalState?

Este es uno de los problemas. Quien se actualiza es el route. Una posible solución sería esta.

```swift
var route: Route? = nil {
    didSet {
        if case let .addItemSheet(addItemState) = self.route {
            // Update State
        }
    }
}
```

Quizás deberíamos valorar si hoy en dia interesa utilizar el OptionalPath o bien seguir usando computed properties para generar local domains con optional states a través del reducer. Seguro que en breve, nos ofrecen la clave de este problema.

Próximo parte, navigation links y navigation stacks.