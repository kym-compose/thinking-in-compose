# Thinking in Compose
- `Jetpack Compose` is a modern declarative UI Toolkit for Android.
- **Compose makes it easier to write and maintain app UI by providing a declarative API.**

### Android View hierarchy
- An Android View hierarchy has been representable as a tree of UI widgets.
  - XML layout.
  - Each widget maintains its own internal state.
  - Each widget exposes setter or getter functions.
- A change in state means that the View hierarchy needs to be updated.
- This requires the following steps:
  - Use the `findViewById()` method to find to the View.
  - Or Use the created `binding` object to refer to the View.
  - Change the internal state by using the `method` of the view.
    - **button.setText(String)**
    - **container.addChild(View)**
    - **img.setImageBitmap(Bitmap)**
- The above process has the following problems:
  - If a data is rendered in multiple places, it's easy to forget to update one of them.
  - Multiple updates may cause unexpected crashes.
  - The software `maintenance complexity` grows with `the number of Views that require updating`.

### The declarative programming paradigm
- Over the last several years, the entire industry has started shifting to a declarative UI model.
- The technique works by conceptually `regenerating` **the entire screen** and `applying` only **the necessary changes.**
- This approach `avoids the complexity` of manually updating a **stateful** View hierarchy.

### The declarative paradigm shift
- The imperative object-oriented UI toolkit differs from the declarative approach.
  - Each widget is relatively stateless.
  - Each widget do not exposes setter or getter functions.
  - Widgets can be thought of as **composable functions** rather than objects.
- `UI is updated by calling the same composable function with different arguments.`
  - When data is changed, UI will also change automatically.
  - This makes it easy to apply [architectural patterns.](https://developer.android.com/topic/architecture)

---

### Composable function
```kotlin
@Composable
fun Greeting(name: String) {
    Text("Hello $name")
}
```

- Must have `@Composable` annotation.
  - **The Compose compiler** can know by this annotation that it is a composable function.
- Must accept `data` as parameters.
  - The data is used to describe the UI.
- Emit `UI hierarchy` by calling other composable functions.
  - Composable functions can only be called from other composable functions.
- `Doesn't return anything.`
  - Instead describe the desired screen state.
- `Fast`, `idempotent`, and `free of side-effects.`
  - Composable functions behave **the same way** when called with **the same argument.**
  - Composable functions describes the UI without any side-effects.
    - Modifying properties or global variables.

### Dynamic content
- Composable functions that written in `Kotlin` can be quite `dynamic.`
  - Can use `loops.`
  ```kotlin
  @Composable
  fun Greeting(names: List<String>) {
      Column {
          for (name in names) {
              Text("Hello $name")
          }
      }
  }
  ```
  - Can use `if` statements.
  ```kotlin
  @Composable
  fun Greeting(names: List<String>) {
      Column {
          if (names.size == 0) {
              Text("No Users")
          } else {
              for (name in names) {
                  Text("Hello $name")
              }
          }
      }
  }
  ```
  - Can call helper functions.

---

### Recomposition
```kotlin
@Composable
fun Title(title: String) {
    Text(text = title)
}

@Composable
fun ClickCounter(clicks: Int, onClick: () -> Unit) {
    Button(onClick = onClick) {
        Text("I've been clicked $clicks times")
    }
}
```

- The button changes the `clicks` value when clicked.
- Every time updates the value of clicks, Compose `calls ClickCounter() again` to show the `new value.`
  - `Title()` is not called.
- This process is called `recomposition.`
  - Compose does not recompose the entire UI tree.
  - `Recompose only the UI that need to be changed.`
    - **functions or lambdas with changed parameter.**
  - So it's very efficient.
- However, that's not to say there aren't any caveats.

### Composable functions can execute in any order
- `The code does not necessarily run in the order shown.`
- If a composable function calls many composable functions, they can be executed in **any order.**
- Compose has the option of recognizing that some UI elements are **higher priority** that others.
- This means that each of those functions needs to be **independent.**
  - Shared use of global variables, etc.

```kotlin
@Composable
fun ButtonRow() {
	MyFancyNavigation {
		StartScreen()
		MiddleScreen()
		EndScreen()
	}
}
```

### Composable functions can run in parallel
- `Compose can optimize recomposition by running composable functions in parallel.`
  - Compose take advantage of **multiple cores.**
  - Compose run composable functions **not on the screen at a lower priority.**

- Composable functions can be called from multiple threads at the same time.
  - Write code that is **thread-safe.**
  - Avoid code that modifies variables in a composable lambda.

```kotlin
@Composable
@Deprecated("Example with bug")
fun ListWithBug(myList: List<String>) {
    var items = 0

    Row(horizontalArrangement = Arrangement.SpaceBetween) {
        Column {
            for (item in myList) {
                Text("Item: $item")
                items++ // Avoid! Side-effect of the column recomposing.
            }
        }
        Text("Count: $items")
    }
}
```

### Recomposition skips as much as possible
```kotlin
/**
 * Display a list of names the user can click with a header
 */
@Composable
fun NamePicker(
    header: String,
    names: List<String>,
    onNameClicked: (String) -> Unit
) {
    Column {
        // this will recompose when [header] changes, but not when [names] changes
        Text(header, style = MaterialTheme.typography.h5)
        Divider()

        // LazyColumn is the Compose version of a RecyclerView.
        // The lambda passed to items() is similar to a RecyclerView.ViewHolder.
        LazyColumn {
            items(names) { name ->
                // When an item's [name] updates, the adapter for that item
                // will recompose. This will not recompose when [header] changes
                NamePickerItem(name, onNameClicked)
            }
        }
    }
}

/**
 * Display a single name the user can click.
 */
@Composable
private fun NamePickerItem(name: String, onClicked: (String) -> Unit) {
    Text(name, Modifier.clickable(onClick = { onClicked(name) }))
}
```

### Recomposition is optimistic
- `Recomposition is optimistc.`
- It means Compose expects to finish recomposition before the parameters change again.
- If a parameter does change before recomposition finishes, Compose **might cancel the recomposition.**
  - And **restart** it with the new parameter.
- If there are side-effects that depend on the being displayed UI, they will be applied.
  - Even if the recomposition is canceled.

- For optimistic reconstructions functions and lambdas need to be:
  - **Idempotent** and have **no side-effects.**

### Composable functions might run quite frequently
- `UI jank can occur when composable executes expensive operations.`
  - Reading and writing to local database.
  - Connecting to a network and retrieving data.
- Let's do the expensive work in a **different thread** outside.
  - And pass data to Compose using `mutableStateOf` or `LiveData.`
