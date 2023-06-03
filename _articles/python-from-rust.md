---
id: 1
title: "Calling Python from Rust"
subtitle: "A quick and dirty way to call your Python modules from Rust"
date: "2023.06.03"
tags: "rust, python"
---

# Python Code
Assume we have the following Python code that we would like to call from Rust:  
```python
def join(s1, s2):
    # Joins the strings s1 and s2
    return s1 + ' ' + s2
```

# Python Setup
We need to ensure the module is visible to the Python interpreter. One way this can be done is by modifying the `PYTHONPATH` [environment variable](https://bic-berkeley.github.io/psych-214-fall-2016/using_pythonpath.html) to include the path to the module:  
```bash
export PYTHONPATH="${PYTHONPATH}:/path/to/your/module/"
```

# Rust Setup
We can utilize the [pyo3 crate](https://docs.rs/pyo3/latest/pyo3/) to take care of the Rust bindings to the Python interpreter:
```rust
[dependencies]
pyo3 = "0.11"
```

# Call Python code from Rust
Now we can use `pyo3` to import and call our Python module from Rust:  
```rust
use pyo3::{prelude::*, types::{PyTuple, PyString}};

fn call_python_function() -> PyResult<String> {
    // Acquire the Python Globabl Interpreter Lock
    let gil = Python::acquire_gil();
    let py = gil.python();

    // Import my Python module
    let module = py.import("my_module")?;

    // Create an array of Python Strings
    let args = [
        PyString::new(py, "Hello"),
        PyString::new(py, "World"),
    ];

    // Call the Python function "join" with some arguments
    let result = module.call1(
        "join", 
        PyTuple::new(py, args),
    )?;

    let result: String = result.extract()?;

    Ok(result)
}

fn main() {
    match call_python_function() {
        Ok(result) => println!("Python function returned: {}", result),
        Err(err) => eprintln!("Error calling Python function: {:?}", err.ptraceback),
    }
}
```

Upon running we see the output: `Python function returned: Hello World`