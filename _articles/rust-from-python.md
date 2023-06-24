---
id: 3
title: "Creating Python Extensions with Rust"
subtitle: "Utilizing PyO3 and maturin to create Python Extensions"
date: "2023.06.24"
tags: "rust, python"
---

# Intro to Python Extensions
Python extensions, are modules written in C or C++ that can be dynamically loaded into the Python interpreter. These extensions provide a way to extend the functionality of Python by integrating with existing C/C++ code or accessing low-level system resources. They allow developers to optimize performance-critical sections of their Python code or interact with libraries and APIs written in other languages. Python extensions provide a bridge between Python's high-level dynamic nature and the low-level power and efficiency of compiled languages, enabling seamless integration and enhancing the capabilities of Python applications.

For this example, we'll write our Python extension in Rust, and use [maturin](https://github.com/PyO3/maturin) to generate the C [FFI](https://en.wikipedia.org/wiki/Foreign_function_interface).

# Setup
First we'll need [maturin](https://github.com/PyO3/maturin), which can be installed through pip:
`pip install maturin`

Next, we'll create a simple Rust project called `rust-py` that contains everything we need to build the extension with:
`maturin new rust-py`
and select `pyo3` for the binding.

# Rust Code
Now we should see a directory named `rust-py` with the following structure:
```
.
├── Cargo.toml
├── pyproject.toml
└── src
    └── lib.rs
```

Examining the `src/lib.rs`:
```rust
use pyo3::prelude::*;

/// Formats the sum of two numbers as string.
#[pyfunction]
fn sum_as_string(a: usize, b: usize) -> PyResult<String> {
    Ok((a + b).to_string())
}

/// A Python module implemented in Rust.
#[pymodule]
fn rust_py(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(sum_as_string, m)?)?;
    Ok(())
}
```

# Build the module
To build and install our Rust code as a Python module, we'll need to first activate a [Python virtual environment](https://docs.python.org/3/library/venv.html), and run `maturin develop` from inside the project:
```
$ source extension/bin/activate
$ cd rust-py
$ maturin develop
...
📦 Built wheel for CPython 3.8 to /tmp/.tmp16Fe0E/rust-py-0.1.0-cp38-cp38-linux_x86_64.whl
🛠 Installed rust-py-0.1.0
```

# Calling Rust from Python
Now that the Python module is built and installed within our virtual environment, we can interact with it like any other Python module:
```python
import rust_py
print(rust_py.sum_as_string(1, 2))
```

For building more interesting examples, I would recommend reading through the [official PyO3 user guide](https://pyo3.rs/v0.11.0/module).

[Full code on GitHub](https://github.com/CadenMG/py-extensions)