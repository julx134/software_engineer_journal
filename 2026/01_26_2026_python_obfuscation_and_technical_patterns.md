# Journal Entry: January 26, 2026

**Project ID:** f59a6424-be1f-49c7-82df-01efe73099e8 <br>
**Series:** 001 <br>
**TAGS:** #pyarmor, #django #architecture, #obfuscation #security, #reverse_engineering_security, #design_pattern

### Description:

> This project was to research and implement a way to obfuscate certain modules of the application which contains intellectual property. This was necessasry due to the application code being directly deployed unto the client's machine. <br>

## Context

The application runs Django ASGI using Uvicorn. We have a engine directory/module that is imported in Django Views to orchestrate and execute business logic which needed to be protected/obfuscated. <br><br>
I took over the task from my manager who was working already working on it midway (he had to drop the task for other priorities). He had caught me up on using a previous version of pyarmor (v7.7) which worked, however, the newer version (v9.2) did not work due to having to use **RFT** mode (Rename Function/Type). <br>

## Part 1: Obfuscation Research and Challenges

### 1. Pyarmor Standard Obfuscation vs. RFT

| Feature       | Standard Obfuscation (`pyarmor gen`)                                                | RFT (`pyarmor gen --enable-rft`)                                        |
| :------------ | :---------------------------------------------------------------------------------- | :---------------------------------------------------------------------- |
| **Mechanism** | Encrypts Python bytecode. Preserves symbol names.                                   | Rewrites source code to rename symbols (e.g., `get_user` $\to$ `f_x9`). |
| **Use Case**  | Framework-heavy apps (Django/FastAPI) or large codebases.                           | Algorithmic IP where logic must be hidden.                              |
| **Weakness**  | "Security by Obscurity." Vulnerable to memory dumping & decompilers (`uncompyle6`). | Breaks "Duck Typing" & external imports if not configured correctly.    |

### 2. Challenges

#### Runtime Problem

Because RFT renames functions, it is extremely difficult to obfuscate standard DRF capabilities like views, routing, models, ...etc., without the application breaking. This is because DRF expects these function names to be standard in the application. For example, `path('user/<id>/', views.get_user)` fails because RFT compiles the code from `def get_user(id):` into `def py_fn_1(py_arg_1):`, causing Django to first fail finding `views.get_user (AttributeError)`, and if fixed, fail passing the parameter `id=5 (TypeError)`.

#### Module Boundary Problem

The solution to the above is to leave the DRF runtime unobfuscated since it contains no significant IP that needs to be protected. However, the same import depedency problem appears again due to the fact that you now have a boundary of unobfuscated (not renamed) and obfuscated (renamed) code -- causing **import dependency hell**.
![alt text](./images/pyarmor_boundary_error.png)

**Key Takeaway:** RFT is

---

## Part 2: Software Architecture & Design Patterns

### 1. The Facade Pattern (Python Implementation)

- **Definition:** A structural pattern that provides a simplified interface to a complex subsystem.
- **Python Implementation:** The `__init__.py` file acts as the Facade.
- **Mechanism:**
  - Internal files (`_calculations.py`, `_utils.py`) contain the logic.
  - `__init__.py` imports specific classes and adds them to `__all__`.
  - External modules import _only_ from the package: `from engine_v2 import MyClass`.
- **Obfuscation Benefit:** `__init__.py` becomes the single source of truth for the RFT Exclude list. Everything listed in `__all__` is public (excluded from renaming); everything else is private (safe to scramble).

### 2. Facade vs. Interface

- **Interface (Contract):** Defines _behavior_ (e.g., `abc.ABC`). Ensures Class A and Class B both have a `run()` method.
- **Facade (Access):** Defines _entry points_. Hides the complexity of _where_ the code lives.

---

## Part 3: The "Barrel File" Epiphany (JS to Python Transfer)

### The Challenge

I faced a critical conflict:

1.  **Goal:** Obfuscate the backend engine (`engine_v2`) using **RFT** for maximum security.
2.  **Problem:** The backend was "deeply coupled" with the engine, importing internal classes directly.
3.  **Failure:** RFT scrambled internal names, causing the backend to crash.

### The Solution: "Wait, isn't this just a Barrel File?"

I realized this is a solved problem in JavaScript/TypeScript (React/Angular) using **Barrel Files** (`index.js`).

- **The Discovery:** Python's `__init__.py` serves the same role as `index.js`.
- **The Implementation:** I used `__init__.py` to explicitly define the package's "Public Interface" by importing key classes and adding them to `__all__`.
- **The Config:** I mapped the `__all__` list directly to Pyarmor's exclusion list.

---

## Part 4: Architecture Enforcement (The "Fail-Safe")

The most powerful outcome is that obfuscation turns **Architecture Conventions** into **Hard Constraints**.

1.  **The Convention:** Developers _should_ only import from the barrel file (`from engine_v2 import Pipeline`).
2.  **The Enforcement:** Because I configured RFT to **scramble everything not in `__init__.py`**, internal names literally cease to exist in the production build.
3.  **The Result:** If a developer bypasses the barrel file, their code will crash in the obfuscated build. The obfuscator acts as a strict architectural linter.
