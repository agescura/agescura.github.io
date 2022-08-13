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