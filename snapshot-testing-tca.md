---
layout: default
---

# Snapshot Testing y TCA.

Una de los puntos fuertes en The Composable Architecture es el testing. La clase TestStore nos permite realizar un testing al 100% de cobertura del sistema que estamos construyendo. Sin embargo, SwiftUI no permite realizar un test para comprobar la buena disposición visual de cada uno de los objetos.

Una posibilidad es realizar un test a modo de captura de imágen, en otras palabras, hacer una foto del estado actual de la aplicación allí donde queramos. Esto se soluciona con Snapshot Testing, otro de los repositorios más usados por PointFree.

Por ejemplo, si tuvieramos una feature, por ejemplo, el típico contador, podríamos hacer un snapshot de la siguiente forma.

```swift
func testSnapshot() {
    let view = CounterView(
        store: .init(
            initialState: .init(),
            reducer: counterReducer,
            environment: .init()
        )
    )
    
    let vc = UIHostingController(rootView: view)
    vc.view.frame = UIScreen.main.bounds
    assertSnapshot(matching: vc, as: .image)
}
```

Pero ahora tenemos un problema porque al instanciar la vista necesitamos acceder a su viewStore para realizar un incrementButtonTapped o un decrementButtonTapped. Existe un truco que es construir un viewStore derivando su propio store hacia uno vacío.

```swift
func testSnapshot() {
    let store = Store(
        initialState: .init(),
        reducer: counterReducer,
        environment: .init()
    )
    let view = CounterView(store: store)
    
    lazy var viewStore = ViewStore(
        store.scope(state: { _ in () }),
        removeDuplicates: ==
    )
    
    let vc = UIHostingController(rootView: view)
    vc.view.frame = UIScreen.main.bounds
    assertSnapshot(matching: vc, as: .image)
    
    viewStore.send(.incrementButtonTapped)
    assertSnapshot(matching: vc, as: .image)
    
    viewStore.send(.decrementButtonTapped)
    assertSnapshot(matching: vc, as: .image)
}
```

De esta forma, podemos crear 3 snapshots y así consiguiendo una cobertura buena para los elementos visuales en SwiftUI.

## TestStore

Nos quedaría ahora validar que los estados internos se testeen. Esto es, utilizar la clase TestStore para asegurarnos que la feature funciona.

```swift
func testHappyPath() {
    let store = TestStore(
        initialState: .init(),
        reducer: counterReducer,
        environment: .init()
    )
    
    store.send(.incrementButtonTapped) {
        $0.count = 1
    }
    
    store.send(.decrementButtonTapped) {
        $0.count = 0
    }
}
```

Si observamos ambos tests. ¿No estamos haciendo lo mismo? Entonces, ¿podríamos intentar juntar ambos tests para no repetir la misma funcionalidad y realizar un solo test que nos compruebe los cambios de estado y luego la parte visual de esos cambios?

La respuesta es, si se puede.

## Snapshot Testing dentro de TCA

Tenemos que extender los metodos send y receive de la clase TestStore.

```swift
extension TestStore {
    func receive() {
    }
    
    func send() {
    }
}
```

La signatura del método send es el siguiente

```swift
@MainActor
@discardableResult
public func send(
    _ action: ScopedAction,
    _ updateExpectingResult: ((inout ScopedState) throws -> Void)? = nil,
    file: StaticString = #file,
    line: UInt = #line
) async -> TestStoreTask
```

Vamos a usar exactamente la misma signatura.

```swift
extension TestStore where Action: Equatable, ScopedState: Equatable {
    @MainActor
    @discardableResult
    func send(
        _ action: ScopedAction,
        _ updateExpectingResult: ((inout ScopedState) throws -> Void)? = nil,
        file: StaticString = #file,
        testName: String = #function,
        line: UInt = #line
    ) async -> TestStoreTask {
    }
}
```

Ahora viene lo más importante. Para cada acción enviada o recibida, queremos levantar la vista a la que pertenece ese TestStore y así poder hacer una foto. Necesitaremos construir la vista a partir de la store, es decir, una función que reciba un Store y devuelva la vista que se construye a partir de esta Store.

```swift
(Store) -> V
```

```swift
extension TestStore where Action: Equatable, ScopedState: Equatable {
    @MainActor
    @discardableResult
    func send<V: View>(
        _ action: ScopedAction,
        _ updateExpectingResult: ((inout ScopedState) throws -> Void)? = nil,
        view: @escaping (Store<ScopedState, ScopedAction>) -> V,
        file: StaticString = #file,
        testName: String = #function,
        line: UInt = #line
    ) async -> TestStoreTask {
    }
}
```

Ahora ya teniendo todos los parámetros definidos, ya podemos implementar la función send.

```swift
extension TestStore where Action: Equatable, ScopedState: Equatable {
    @MainActor
    @discardableResult
    func send<V: View>(
        _ action: ScopedAction,
        view: @escaping (Store<ScopedState, ScopedAction>) -> V,
        _ updateExpectingResult: ((inout ScopedState) throws -> Void)? = nil,
        file: StaticString = #file,
        testName: String = #function,
        line: UInt = #line
    ) async -> TestStoreTask {
        await send(action) { state in
            do {
                try updateExpectingResult?(&state)
            } catch {
                XCTFail("Threw error: \(error)", file: file, line: line)
            }
            let store = Store<ScopedState, ScopedAction>(initialState: state, reducer: .empty, environment: ())
            let view = view(store)
            assertSnapshot(matching: view, as: .image(layout: .device(config: .iPhone13)))
        }
    }
}
```

Ahora, podríamos actualizar el test happyPath anterior.

```swift
func testHappyPath() {
    let store = TestStore(
        initialState: .init(),
        reducer: counterReducer,
        environment: .init()
    )
    
    store.send(.incrementButtonTapped, view: CounterView.init(store:)) {
        $0.count = 1
    }
    
    store.send(.decrementButtonTapped, view: CounterView.init(store:)) {
        $0.count = 0
    }
}
```

El test se ejecuta perfectamente, comprueba el cambio de estado y se lanza un snapshot. Ahora bien, los errores aparecen en los métodos creados anteriormente. Podemos derivar los mensajes de error hacia nuestro test.

```swift
extension TestStore where Action: Equatable, ScopedState: Equatable {
    @MainActor
    @discardableResult
    func send<V: View>(
        _ action: ScopedAction,
        view: @escaping (Store<ScopedState, ScopedAction>) -> V,
        _ updateExpectingResult: ((inout ScopedState) throws -> Void)? = nil,
        file: StaticString = #file,
        testName: String = #function,
        line: UInt = #line
    ) async -> TestStoreTask {
        await send(action) { state in
            do {
                try updateExpectingResult?(&state)
            } catch {
                XCTFail("Threw error: \(error)", file: file, line: line)
            }
            let store = Store<ScopedState, ScopedAction>(initialState: state, reducer: .empty, environment: ())
            let view = view(store)
            assertSnapshot(matching: view, as: .image(layout: .device(config: .iPhone13)), file: file, testName: testName, line: line)
        }
    }
}
```

Por último, podría ser interesante pasar la estrategia del snapshot fuera.

```swift
@MainActor
@discardableResult
func send<V: View, Format>(
    _ action: ScopedAction,
    view: @escaping (Store<ScopedState, ScopedAction>) -> V,
    as snapshotting: Snapshotting<V, Format>,
    _ updateExpectingResult: ((inout ScopedState) throws -> Void)? = nil,
    file: StaticString = #file,
    testName: String = #function,
    line: UInt = #line
) async -> TestStoreTask {
    await send(action) { state in
        do {
            try updateExpectingResult?(&state)
        } catch {
            XCTFail("Threw error: \(error)", file: file, line: line)
        }
        let store = Store<ScopedState, ScopedAction>(initialState: state, reducer: .empty, environment: ())
        let view = view(store)
        assertSnapshot(matching: view, as: snapshotting, file: file, testName: testName, line: line)
    }
}
```

Ya lo tendríamos.

```swift
func testHappyPath() {
    let store = TestStore(
        initialState: .init(),
        reducer: counterReducer,
        environment: .init()
    )
    
    await store.send(.incrementButtonTapped, view: CounterView.init(store:), as: .image(layout: .device(config: .iPhone13))) {
            $0.count = 1
        }
        
    await store.send(.decrementButtonTapped, view: CounterView.init(store:), as: .image(layout: .device(config: .iPhone13))) {
        $0.count = 0
    }
}
```

Por último, pondremos la extensión para el método receive. Así quedaría nuestra extensión terminada.

```swift
extension TestStore where Action: Equatable, ScopedState: Equatable {
    @MainActor
    func receive<V: View, Format>(
        _ action: Action,
        view: @escaping (Store<ScopedState, ScopedAction>) -> V,
        as snapshotting: Snapshotting<V, Format>,
        _ updateExpectingResult: ((inout ScopedState) throws -> Void)? = nil,
        file: StaticString = #file,
        testName: String = #function,
        line: UInt = #line
    ) async {
        await receive(action) { state in
            do {
                try updateExpectingResult?(&state)
            } catch {
                XCTFail("Threw error: \(error)", file: file, line: line)
            }
            let store = Store<ScopedState, ScopedAction>(initialState: state, reducer: .empty, environment: ())
            let view = view(store)
            assertSnapshot(matching: view, as: snapshotting, file: file, testName: testName, line: line)
        }
    }
    
    @MainActor
    @discardableResult
    func send<V: View, Format>(
        _ action: ScopedAction,
        view: @escaping (Store<ScopedState, ScopedAction>) -> V,
        as snapshotting: Snapshotting<V, Format>,
        _ updateExpectingResult: ((inout ScopedState) throws -> Void)? = nil,
        file: StaticString = #file,
        testName: String = #function,
        line: UInt = #line
    ) async -> TestStoreTask {
        await send(action) { state in
            do {
                try updateExpectingResult?(&state)
            } catch {
                XCTFail("Threw error: \(error)", file: file, line: line)
            }
            let store = Store<ScopedState, ScopedAction>(initialState: state, reducer: .empty, environment: ())
            let view = view(store)
            assertSnapshot(matching: view, as: snapshotting, file: file, testName: testName, line: line)
        }
    }
}
```