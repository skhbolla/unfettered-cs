When we talk about the Python Virtual Machine (PVM) “running” the bytecode, it’s easy to imagine that the PVM converts this bytecode into native machine code and executes it. But that’s not actually what happens.

In reality, the PVM does **not** generate any native machine code. The execution of a Python program occurs as a **side effect** of the PVM repeatedly invoking the relevant C functions that implement each bytecode instruction.

Some alternative implementations, like **PyPy**, include a **JIT compiler** that caches the native machine code for frequently executed (“hot”) sections of bytecode — so those C functions don’t need to be called again and again.

Continue reading for a in-depth guide.

Important :   When a compiled program runs (eg ./a.out) it itself becomes a process with own heap and stack and other segments ; but when a interepreted code is run (eg python3 [main.py](http://main.py)) , the main.py does not get its own process , the python3 executable becomes a process with its stack and heap and other segments , the main.py file is then loaded into the heap of the python3 process. What ensues is the conversion of the python string data(raw code) to Python Byte Code (compiled by the compiler inside the python executable) , this bytecode is stored as “PyByteCode” objects (again on heap of the python executable) , the PVM section of the python executable (which is just a giant loop with switch cases for each bytecode opcode) then picks up the produced byte code and executes the relevant C functions.

As explained above , The PVM takes python bytecode and calls the relevant C function. Since the executable is pre-compiled (of course , what else can it be LOL) , that means the C function is already in binary format (machine code). i.e when the PVM looks at a bytecode op , it calls the relevant machine code (that came pre compiled with the Python executable). Now where does ***JIT*** come in here??

Since the bytecode can have many repetitions of the same opcode , we can optimize the calling of the relevant machine code by caching the commonly used opcode-machinecode combos. This is a layer that sits between the CPU and the PVM , ie the PVM tries to execute relevant machine code , but JIT takes over.

In standard Python execution, the PVM acts like a manager reading a to-do list (bytecode). For every item, it pauses to look up and call a generic C function (pre-compiled machine code) to do the work. My intuition was that JIT acts as a "cache" to make this lookup faster. While logical, this was incorrect because it still traps the execution in the slow "interpreter loop," where the CPU wastes significant time jumping back and forth between the manager and the workers for every single instruction.

A true JIT compiler eliminates the manager entirely for frequently used code. Instead of calling those generic C functions one by one, the JIT analyzes the bytecode and translates the entire sequence into a brand-new, streamlined block of machine code. This allows the CPU to execute the logic directly and continuously in hardware without ever checking back in with the PVM, resulting in massive speed gains.

**The JIT Workflow:**

1. **Monitor:** The PVM runs normally (interpreting). It notices a specific function is being run 10,000 times ("Hot Code").
2. **Compile:** The JIT pauses, reads that Python Bytecode, and translates it into a new block of raw Machine Code.
    - *Crucial Step:* It often **inlines** logic. Instead of calling the generic "Add two Python Objects" C function, it might see that you are only adding integers. It will generate the machine code for a simple hardware integer add (`ADD EAX, EBX`), which is 100x faster.
3. **Patch:** The PVM swaps the pointer. Next time code calls that function, the PVM says, "Don't interpret this. Jump directly to this new block of machine code I just wrote."

### 

# 🧠 The Hidden Life of Your Python Code — From Source to CPU (and Where JIT Fits In)

If you’ve ever heard that *“Python is an interpreted language”*, you might have pictured it as some magical engine reading your `.py` files line by line.

But that’s not really what happens.

In fact, there’s a fascinating chain of events between the moment you hit **Run** and the moment your CPU executes your code. And understanding this chain explains why Python can feel slow — and how **JIT (Just-In-Time) compilation** changes everything.

Let’s peel back the layers, one by one.

---

## 🧩 Step 1 — Python Isn’t “Just Interpreted”

When you run a Python file:

```bash
python hello.py

```

You’re not directly “interpreting” the source code.

What actually happens is:

1. **Your source code (`hello.py`)** is first **compiled into bytecode** — a lower-level, platform-independent representation.
2. This **bytecode** is then executed by an **interpreter** — the part we call the “Python runtime”.

That bytecode lives inside `.pyc` files — the stuff you see in the `__pycache__` folder.

So already, Python has a compiler — it’s just compiling to **bytecode**, not native CPU instructions.

---

## ⚙️ Step 2 — The Interpreter’s Real Job

Now we’re inside the **interpreter**, which takes bytecode as input.

In **CPython** (the default implementation of Python), this interpreter is written in C.

Here’s the big picture:

- Each Python instruction — like `LOAD_CONST`, `BINARY_ADD`, or `CALL_FUNCTION` — is just a *bytecode opcode*.
- The CPython interpreter reads these opcodes one by one in a **loop** called the *eval loop*.
- For every opcode, it calls a corresponding **C function**.

For example:

| Bytecode | What CPython Does |
| --- | --- |
| `BINARY_ADD` | Calls `PyNumber_Add()` (a C function that handles addition) |
| `LOAD_CONST` | Pushes a constant value to the stack |
| `CALL_FUNCTION` | Calls another Python function, managing frames, arguments, etc. |

So, when you write:

```python
a = 2 + 3

```

What actually happens under the hood is roughly:

```
LOAD_CONST 2
LOAD_CONST 3
BINARY_ADD   →  calls PyNumber_Add()
STORE_NAME a

```

The **interpreter executes this by calling C functions** for each step.

That’s why Python feels slower — you’re not executing native instructions directly, but constantly jumping between layers of logic, type checks, and function calls.

---

## ⚡ Step 3 — What Does “Interpreting” Actually Produce?

At the hardware level, the CPU doesn’t understand Python, or bytecode, or even C.

It only understands **machine code** — binary instructions like:

```
MOV RAX, 5
ADD RAX, RBX

```

The Python interpreter’s job, therefore, is to:

- Take your **bytecode**,
- Look at each opcode,
- Run some **C code** that simulates what that instruction means,
- And let the CPU execute that C code.

So effectively:

```
Python bytecode
   ↓
C interpreter functions
   ↓
CPU executes machine code produced by those C functions

```

But the interpreter never stores or reuses that machine code.

Each time you run that loop again, it goes through the same cycle — decode bytecode → call C function → execute → repeat.

That’s why interpreted code runs slower: there’s no caching or reuse of CPU-level instructions.

---

## 🔥 Step 4 — Enter the JIT Compiler

This is where the **JIT (Just-In-Time compiler)** comes in — it’s the clever middle layer between the interpreter and the CPU.

A JIT compiler is like saying:

> “Hey, we’ve been interpreting this same loop a million times. Let’s stop translating every line and just compile this part into real CPU code.”
> 

In a JIT-enabled runtime like **PyPy**, the system starts exactly like CPython — interpreting bytecode. But it also **watches what’s happening**.

Whenever it notices a piece of code (say, a loop or function) being executed over and over, it flags that as **hot**.

Then it:

1. **Compiles** that section of bytecode into **native machine code** on the fly.
2. **Caches** that machine code in memory.
3. Next time, **skips the interpreter entirely** and jumps straight to that native version.

This process is called *runtime specialization* — the code adapts itself to how your program behaves.

---

## 🧮 What’s Actually Different?

Let’s compare the two:

| Concept | CPython (No JIT) | PyPy (With JIT) |
| --- | --- | --- |
| Execution | Interprets bytecode repeatedly | Interprets first, then compiles hot sections |
| Speed | Slower (C function overhead) | Faster (native CPU instructions) |
| Type handling | Checks types every time | Specializes compiled code for current types |
| Memory | No native cache | Keeps a JIT cache of machine code |
| Example | `PyNumber_Add()` every time | Inline `ADD rax, rbx` directly |

So when a JIT compiler “sits in the middle,” it’s literally short-circuiting the slow path:

```
CPython: Bytecode → C functions → Machine code
PyPy:    Bytecode → Machine code (cached and reused)

```

---

## 🚀 Step 5 — Why This Feels Magical

Let’s make it concrete.

In CPython:

```python
for i in range(10_000_000):
    total += i

```

Every single iteration goes through the interpreter’s eval loop, calling `PyNumber_Add` millions of times.

In PyPy (JIT):

- The first few iterations run normally.
- PyPy detects that this loop is hot.
- It compiles it into machine code once.
- From that point, your CPU executes **pure native instructions** directly — as if you had written it in C.

That’s why PyPy often runs **4× to 10× faster** than CPython — sometimes even more for math-heavy code.

---

## 🧠 Step 6 — The Beautiful Tradeoff

But there’s no free lunch.

JIT compilation itself takes time — analyzing bytecode, profiling, and generating machine code.

So, for **short-lived scripts**, CPython may still be faster.

But for **long-running programs** (like web servers, simulations, or ML workloads), JIT wins big.

---

## 🧩 Step 7 — A Mental Model to Remember

Here’s the big picture of the Python execution pipeline:

```
Your Code (.py)
   ↓
Bytecode (.pyc)
   ↓
┌──────────────────────────┐
│  Interpreter (CPython)   │
│  Reads bytecode, calls C │
└────────────┬─────────────┘
             │
             ▼
      (In PyPy)
      ┌────────────────────┐
      │    JIT Compiler    │
      │ Detects + caches   │
      │ native code        │
      └─────────┬──────────┘
                │
                ▼
           CPU executes
         optimized machine code

```

---

## 🏁 Step 8 — So, Is Python Interpreted or Compiled?

Both.

That’s the twist.

- It’s **compiled** from source → to bytecode.
- It’s **interpreted** from bytecode → by the runtime.
- It can be **JIT-compiled** at runtime → to native machine code (in some implementations).

Python isn’t slow because it’s “interpreted.”

It’s slow because the interpreter has to keep doing work the JIT can avoid — converting the same instructions over and over instead of caching them.

---

## 💬 Final Thought

Once you see Python this way — as a system of *translation layers* — it changes how you think about performance.

The next time you hear “Python is interpreted,” you can smile and think:

> “Sure… until the JIT gets involved. Then it’s just Python — but turbocharged.”
> 

---

# 🧠 How the Python Virtual Machine Actually Executes Bytecode

When we say *“the Python Virtual Machine runs bytecode,”* it’s easy to imagine the PVM converting that bytecode into native machine code and handing it off to the CPU. But that’s not what actually happens — at least not in standard CPython.

In reality, the **PVM doesn’t generate any native machine code at all**. Instead, it behaves like a sophisticated *dispatcher* written in C.

Here’s what really happens:

1. **The bytecode interpreter loop** (implemented in C inside `ceval.c`) reads one bytecode instruction at a time — for example, `LOAD_CONST`, `CALL_FUNCTION`, or `BINARY_ADD`.
2. For each opcode, the PVM looks up the corresponding **C function implementation** that knows how to perform that operation.
3. It then **calls that C function** directly.
    - For example, when your Python code does `x + y`, the PVM calls a C function (like `PyNumber_Add`) that knows how to add two Python objects, handle type checks, and manage memory safely.
4. These C functions execute at the native level — meaning, *they* are compiled into real machine code as part of the CPython binary itself.
5. The output of those C functions is usually another Python object (e.g., the result of `x + y`), which is then pushed back onto the interpreter’s internal stack.

So, **execution in Python happens as a side effect** of the PVM repeatedly calling precompiled C functions — not by directly turning bytecode into CPU instructions.

That’s why we say Python is *interpreted* rather than compiled.

---

### ⚡ Where JIT Fits In

Now, JIT-enabled implementations like **PyPy** add an optimization layer here.

They still start by interpreting bytecode, but they **observe** which sections of code (called “hot loops”) are executed repeatedly. For these hot spots, the JIT compiler dynamically **generates native machine code** — effectively caching it so that next time, the interpreter doesn’t need to call those C functions repeatedly.

This is what makes JIT so powerful: it *learns* which parts of your program matter most and optimizes them while it runs.

---

Would you like me to show you how this section fits smoothly into the **entire blog post** (so you can see the context and flow from start to finish)?
