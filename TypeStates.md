# Typestates

Typestates allow Data to define mutually exclusive compile-time states. Methods can be constrained to specific states and can transition the instance between states.

## Declaring States

```EBNF
StateList ::= Identifier ("," Identifier)*
```

* **Default State**: A Data type with states MUST explicitly resolve to a specific state upon instantiation or when used as a type (e.g., `File::Opened`).
* **Namespace Collision**: Because states are accessed via the `::` operator (e.g., `File::Opened`), the Data MUST NOT contain any nested `Data` declarations whose name exactly matches a state identifier. This is a strict compile-time error.

```C#
// Invalid: Namespace collision
File :: Opened, Closed 
{
    Opened { } // ERROR: Cannot declare nested Data with the same name as a state
}
```

## State-Bound Receivers and Transitions

Method receivers strictly dictate which states the method accepts and how the state transitions after the call.

1. **Universal Borrow (`@` or `mut @`)**: If no state is specified, the method MAY be called regardless of the instance's current state. The state remains unchanged.
2. **Restricted Borrow (`@State`)**: The method can ONLY be called if the instance is currently in `State`. The state remains unchanged after the call.
3. **State Transition (`@Current -> Next`)**: The method requires the instance to be in `Current`. Upon successful return, the compiler strictly mutates the static type of the caller's variable to `Next`.
4. **Transition Mutability Requirement**: Because a transition alters the semantic state of the caller's binding, the variable on the caller's side MUST be declared as `mut` to allow the transition.

```C#
File :: Opened, Closed
{
    _handle: int;

    // Factory method strictly returning the Opened state.
    // '-> @Opened' is shorthand for '-> File::Opened'
    New(path: string) -> @Opened 
    {
        return File { _handle: Syscall.OpenFile(path) };
    }

    // Universal method: callable in ANY state
    PrintCool(@) 
    {
        Console.WriteLine("Cool!");
    }

    // Restricted method: requires Opened, does not transition
    Read(@Opened) -> byte[] 
    {
        return Syscall.ReadFile(@._handle);
    }

    // Transition method: requires Opened, transitions to Closed
    Close(@Opened -> Closed) 
    {
        Syscall.CloseFile(@._handle);
    }
}
```

## Call Site and Aliasing Safety

The compiler performs linear tracking of the variable's state across the execution path.

```C#
mut file := File.New("log.txt"); // tracked as File::Opened

file.PrintCool();     // Valid: Universal method
data := file.Read();  // Valid: 'file' is Opened
file.Close();         // Valid: Transitions 'file' to File::Closed

// file.Read();       // ERROR: 'Read' requires Opened, but 'file' is Closed
// file.Close();      // ERROR: 'Close' requires Opened, but 'file' is Closed
file.PrintCool();     // Valid: Universal method callable on Closed
```

* **Transition Block**: A method that transitions the state MUST NOT be called if there are ANY active View Pointers (`&` or `mut&`) referencing the instance.
* The Last Use Place (LUP) of all active View Pointers MUST strictly precede the state transition. This eliminates aliasing bugs at compile time.

## Divergent States

If an instance undergoes a state transition inside a conditional branch (`if` or `match`), the compiler MUST unify the state at the end of the branches.

* **Divergent States Rule**: If a variable transitions to `StateA` in an `if` branch and `StateB` in an `else` branch (or remains unchanged), its state becomes **divergent**.
* Calling any state-specific method on a divergent variable is a strict compilation error. The developer MUST manually restructure the logic to ensure a deterministic state, or fully consume the instance before the branches merge.

```C#
mut file := File.New("log.txt");

if condition 
{
    file.Close(); // State is Closed here
} 
else 
{
    // State is Opened here
}

// ERROR: State of 'file' is divergent (Opened | Closed). 
// The compiler prevents calling file.Close() or file.Read() here.
```
