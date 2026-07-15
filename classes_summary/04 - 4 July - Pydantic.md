# 🛡️ Class 3: Pydantic Deep Dive + AI Foundations
### 📋 Agentic AI 3.0 Specialization | Krish Naik Academy

**🎙️ Mentor:** Mayank Aggarwal
**⏱️ Duration:** ~4.5 hours | **📅 Session:** Day 3 (4 July 2026)

---

## 📰 Quick Updates Before Class

- 🎥 New video posted on **Fable 5** access being reinstated in India (temporary access window shared for the week).
- 💼 Reminder: Mayank is meeting his ex-manager (now Global AI Head at Puma) — last call for eligible learners to send CVs for the **MLOps Engineer** / **Senior AI Engineer** referrals from Class 1.
- 🧠 A custom **"MayankGPT"** doubt-solver is live — trained specifically to explain concepts with analogies, step-by-step, tailored to this course.

---

## 🤔 Why Pydantic Exists — The Deeper Case

```mermaid
flowchart LR
    A["🔤 Statically typed languages<br/>Java, C++"] --> B["Memory blocks reserved<br/>by declared type"]
    C["🐍 Python / JavaScript"] --> D["No type enforcement<br/>at all — 'loose' by design"]
    D --> E["✅ Great for learning<br/>❌ Dangerous in production"]

    style E fill:#fecaca,stroke:#ef4444
```

- Python stores a variable as a **pointer to a location** holding the value — not a fixed-size memory block like C++/Java. That's *why* it can freely reassign a variable from `int` to `string` to `list` with zero complaints.
- 🏢 **Where this bites you:** production systems, real user input, anywhere "someone might try to break your system." Good data flows fine; bad/malicious data silently corrupts things — e.g. wrong types crash string/number operations downstream (in a database, spreadsheet, etc.).
- 🌍 **Industry proof:** Anthropic's SDK, OpenAI's Chat Completions API, NVIDIA, Google, Adobe, Amazon — Pydantic's own docs list major companies relying on it. It's a **PyPI top-download library**.

> 💬 *"Do you think that because Swiggy/Zomato exist, we don't need to know how to cook? Same with AI writing your code — if you don't understand Pydantic, you can't debug or trust the code AI hands you."*

---

## 🧱 Three Ways to Define a "Blueprint" in Python

```mermaid
flowchart TD
    A["1️⃣ Plain class<br/>write __init__ yourself"] --> D["❌ No type checking at all"]
    B["2️⃣ @dataclass<br/>auto-generates __init__"] --> E["❌ Still no type checking"]
    C["3️⃣ Pydantic BaseModel<br/>inherit BaseModel"] --> F["✅ Full type validation<br/>on every field, automatically"]

    style C fill:#6366f1,color:#fff
    style F fill:#22c55e,color:#fff
```

```python
from pydantic import BaseModel

class UserModel(BaseModel):
    name: str
    email: str
    age: int

user = UserModel(name="Mayank", age="not-a-number")  # ❌ ValidationError, instantly
```

> 💡 Live demo confirmed: a plain class and a `@dataclass` both **silently accepted garbage data** (e.g. `age="banana"`). Only `BaseModel` raised an error immediately.

---

## 🎛️ Optional Fields & Defaults

```python
class SignupForm(BaseModel):
    name: str
    age: int
    is_interested: bool = False   # default value → makes it optional
    nationality: str | None = None  # unknown default → use None
```

- Giving a field a **default value** is what makes it optional — otherwise every field is required.
- If you don't know the right default, use Python's `None` (capital N — there's no `null` in Python).

---

## 🔄 Automatic Type Coercion — Pydantic's "Being Reasonable" Mode

```mermaid
flowchart LR
    A["age: '28'<br/>(string)"] -->|"Pydantic converts it"| B["age: 28<br/>(int) ✅"]
    C["age: '28eight'<br/>(garbage string)"] -->|"Can't be parsed"| D["❌ ValidationError"]
    E["age: 28.5<br/>(float into int field)"] -->|"Too lossy to guess"| F["❌ ValidationError"]

    style B fill:#dcfce7,stroke:#22c55e
    style D fill:#fecaca,stroke:#ef4444
    style F fill:#fecaca,stroke:#ef4444
```

> Pydantic will forgive a *reasonable* type mismatch (a numeric string → int) but draws the line at genuinely ambiguous or lossy conversions.

---

## 📏 `Field()` — Data Validation, Not Just Type Validation

Type validation confirms *what kind* of data you got. **Data validation** confirms the *value itself* makes sense (age of 1000? not valid, even though it's technically an int).

```python
from pydantic import BaseModel, Field

class JobApplication(BaseModel):
    full_name: str = Field(min_length=2, max_length=100)
    years_experience: int = Field(gt=0, le=50)
    portfolio_url: str
```

- Two equivalent syntaxes exist: `Field(...)` vs. `Annotated[...]` — Mayank prefers the **plain `Field()`** style for readability, but flagged that AI-generated code often defaults to `Annotated`, so recognize both.
- Built-in special types save you from writing your own regex:
  - `EmailStr` → validates real email format (requires `email-validator` package)
  - `HttpUrl` → validates URLs
  - `SecretStr` → **masks sensitive values** like API keys/passwords in logs/output — flagged as underused but important for this course, since API keys will be handled constantly.

---

## 🧭 field_validator vs. model_validator

### The Problem
> *"If someone applies from an `@infosys.com` email, they need at least 5 years experience." Can a single-field validator handle that?*

```mermaid
flowchart TD
    A["🔍 field_validator<br/>sees ONE field at a time"] --> B["❌ Can't compare email + years_experience together<br/>— 'Schrödinger's field': you see one or the other, never both"]
    C["🔍 model_validator<br/>sees the ENTIRE model"] --> D["✅ Can enforce cross-field rules<br/>e.g. password == confirm_password"]

    style B fill:#fecaca,stroke:#ef4444
    style D fill:#dcfce7,stroke:#22c55e
```

```python
from pydantic import field_validator, model_validator

class SignupForm(BaseModel):
    password: str
    confirm_password: str

    @field_validator("password")
    @classmethod
    def check_length(cls, value):
        if len(value) < 8:
            raise ValueError("Password must be at least 8 characters")
        return value

    @model_validator(mode="after")
    def check_passwords_match(self):
        if self.password != self.confirm_password:
            raise ValueError("Passwords do not match")
        return self
```

⚙️ **Execution order is fixed:** every `field_validator` runs first (per field) → *then* `model_validator` runs on the fully-validated object. This is logical — no point cross-checking fields that haven't even passed their own individual checks yet.

---

## 🪆 Nested Models

```mermaid
flowchart TD
    A["class Applicant(BaseModel)"] --> B["name: str"]
    A --> C["address: Address"]
    C --> D["class Address(BaseModel)<br/>street, city, pin_code"]

    style C fill:#f59e0b,color:#fff
```

- A Pydantic model can contain another Pydantic model as a field type — mirrors **nested JSON** (a JSON object inside a JSON object), which is exactly the shape of most real-world API payloads and LLM structured outputs.

---

## 🧠 AI Foundations — The "21 Terms Everyone Should Know"

> *"These aren't deep-dive lessons yet — think of these as the vocabulary you need before we build our first agent tomorrow."*

### 1️⃣ LLM (Large Language Model)
```mermaid
flowchart LR
    A["📚 Trained on nearly<br/>everything ever written"] --> B["🎲 Learns statistical patterns:<br/>what word usually comes next"]
    B --> C["🔮 Generates text one token<br/>at a time, based on probability"]

    style C fill:#6366f1,color:#fff
```
> 🧠 **Analogy:** *"A friend who's read every book ever published but was never told what any of it means — they've just seen 'thank you for' followed by 'your business' millions of times, so they continue it. Not understanding — pattern completion."*

- ChatGPT, Claude, Gemini = **LLMs**. OpenAI/Anthropic/Google = **LLM providers**.
- Technically: an LLM predicts the next **token**, not the next word.

### 2️⃣ Tokens — "The Currency of AI"
```mermaid
flowchart LR
    A["📝 Your text"] --> B["✂️ Broken into tokens<br/>~¾ of a word each"]
    B --> C["🤖 LLM sees ONLY tokens<br/>never raw words"]
    C --> D["💰 You're billed per token<br/>input + output"]

    style D fill:#f59e0b,color:#fff
```
> 🧠 **Analogy:** *"Like old prepaid phone plans charged per minute — more words = more minutes. With AI, tokens are that currency."*
- **Output tokens always cost more than input tokens** — because the model has to *think and generate*, not just receive.
- A token ≈ a "Lego brick," not a full word: `"unbelievable"` might split into 3 tokens (`un`, `believ`, `able`).

### 3️⃣ Vector Embeddings
```mermaid
flowchart LR
    A["🔤 Word: 'dog'"] --> B["📍 Converted to a vector<br/>(list of numbers = coordinates)"]
    B --> C["🗺️ Placed in high-dimensional space"]
    C --> D["🐕 'puppy' sits close to 'dog'<br/>😊 'happy' sits far from both"]

    style D fill:#a5b4fc,stroke:#6366f1
```
- Words with similar meaning end up **close together** in this space; unrelated words end up far apart.
- Demoed live with a **10,000-word, 200-dimension vector space visualization** — searching "guitar" surfaced musically-related nearest neighbors.
- ⚠️ Not unique to AI — this comes from classic **NLP (Natural Language Processing)**.

### 4️⃣ Context Window
```mermaid
flowchart LR
    A["🖊️ Whiteboard<br/>(fixed size)"] --> B["New message written"]
    B --> C{"Board full?"}
    C -->|No| D["Keeps writing"]
    C -->|Yes| E["🧹 Oldest content<br/>erased to make room"]

    style E fill:#fecaca,stroke:#ef4444
```
> 🧠 **Analogy:** *"Your context window as a student is about 3 hours — teach you for 6 and you'll forget what I said in hour one. Same with AI: once the token limit is hit, the oldest content quietly falls off."*
- E.g. a "400K context window" model can hold ~400,000 tokens of conversation/documents before older content starts getting dropped.
- **Not the same as memory** — memory is a separate, persistent mechanism (covered in a future class), while context window is just the model's live "whiteboard" for the current conversation.

### 5️⃣ Parameters
```mermaid
flowchart LR
    A["🎛️ Billions of tiny internal values<br/>(weights)"] --> B["Set during training"]
    B --> C["No single one means anything alone"]
    C --> D["✨ Combined = model's capability"]

    style D fill:#6366f1,color:#fff
```
> 🧠 **Analogy:** *"A mixing console with billions of tiny sliders, each nudged a fraction of a millimeter during training. You can't point to 'the slider that knows Paris is in France' — the magic is in the combination."*
- **Parameters ≠ Tokens**: parameters are fixed at training time (e.g. "1.7 trillion parameters"); tokens are what flows in/out every time you *use* the model.
- More parameters ≈ more potential capability, but **never a guarantee** of a better answer on any single query.

---

## 🗺️ What's Next

```mermaid
flowchart LR
    A["✅ Pydantic mastered"] --> B["✅ AI vocabulary set"]
    B --> C["🔧 Tomorrow: real API call<br/>via OpenRouter"]
    C --> D["🤖 Build your first agent<br/>— pure Python, no framework"]
    D --> E["🔗 Following weekend:<br/>LangChain begins"]

    style D fill:#f59e0b,color:#fff
    style E fill:#22c55e,color:#fff
```

---

## 💬 Live Q&A Highlights

| Question | Answer |
|---|---|
| Can I extend/reuse a Pydantic model's validators across classes (like Java extension methods)? | Yes — standard Python inheritance works: `class Address(BaseModel)`, then `class Applicant(Address)` inherits its fields/validators |
| Can `field_validator` be a `@staticmethod`? | Technically possible, but stay consistent — use Pydantic's own patterns (`@classmethod` + `@field_validator`) rather than mixing approaches |
| Difference between "parameters" and "tokens"? | Parameters = fixed internal weights set during training; tokens = the live currency of input/output when you *use* the model |
| Can vector embeddings have any number of dimensions? | Yes — dimension count is a design choice you make (e.g. 200D); more dimensions = better semantic separation but higher hardware cost |
| Is a "dimension" like a category (e.g. 7 colors)? | No — dimensions aren't hand-labeled categories; they're a mathematical space size you choose before training/plotting |

---

## ✅ Action Items After Class 3

- [ ] 🛡️ Re-practice defining a `BaseModel` with `Field()` constraints (min/max length, gt/le) from scratch
- [ ] 🧩 Write a `field_validator` and a `model_validator` from memory — know *why* each exists
- [ ] 🪆 Practice a nested Pydantic model (e.g. `Applicant` containing `Address`)
- [ ] 📖 Revise the 5 AI foundation terms: **LLM, Token, Vector Embedding, Context Window, Parameters** — know them cold before next class
- [ ] 🔁 Review today's GitHub code before the next session — tomorrow builds a real OpenRouter API call + first pure-Python agent on top of it
- [ ] 💼 Eligible + interested in the Puma roles? Send your CV before Monday

---

*📝 Notes compiled from the full Class 3 transcript — "Pydantic Deep Dive + AI Foundations," Agentic AI 3.0 Specialization, Krish Naik Academy.*
