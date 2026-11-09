# Python for DevOps — Core Concepts

<!-- DEVOPS-SHACK:HEADER -->
<p align="center">
  <a href="https://www.youtube.com/@devopsshack"><img alt="YouTube DevOps Shack" src="https://img.shields.io/badge/YouTube-DevOps%20Shack-FF0000?style=flat-square&logo=youtube&logoColor=white"/></a>
  <a href="https://www.youtube.com/@shackverse"><img alt="YouTube DevOps Shack Hindi" src="https://img.shields.io/badge/Hindi-@shackverse-FF0000?style=flat-square&logo=youtube&logoColor=white"/></a>
  <a href="https://devopsshack.com"><img alt="devopsshack.com" src="https://img.shields.io/badge/Courses-devopsshack.com-C6FF4D?style=flat-square&labelColor=0B0F1A"/></a>
  <a href="https://www.instagram.com/devopsshack"><img alt="Instagram" src="https://img.shields.io/badge/Instagram-@devopsshack-E4405F?style=flat-square&logo=instagram&logoColor=white"/></a>
  <a href="https://github.com/jaiswaladi246/DevOps-Cloud-Interview-Preparation-by-DevOps-Shack"><img alt="Original repo" src="https://img.shields.io/badge/Original-DevOps%20Shack%20repo-4DE8FF?style=flat-square&logo=github&labelColor=0B0F1A"/></a>
</p>
<p align="center"><sub>Part of <b><a href="https://github.com/jaiswaladi246/DevOps-Cloud-Interview-Preparation-by-DevOps-Shack">DevOps &amp; Cloud Interview Preparation</a></b> — created by <b><a href="https://www.youtube.com/@devopsshack">DevOps Shack</a></b>. Free to learn from; please credit DevOps Shack when sharing.</sub></p>
<!-- /DEVOPS-SHACK:HEADER -->

> Language essentials, the standard library for ops, packaging, boto3 and infra SDKs, concurrency, testing, services and best practices, plus rapid-fire commands · 200 questions · 🟢 Beginner 🟡 Intermediate 🔴 Advanced

[⬅ Back to Python for DevOps](./README.md) · [🏠 Home](../README.md)

## Table of contents
- [Language essentials & functions](#language-essentials--functions) — Q1–Q40
- [Exceptions, context managers, files & logging](#exceptions-context-managers-files--logging) — Q41–Q60
- [Environments, dependencies & packaging](#environments-dependencies--packaging) — Q61–Q76
- [OS, filesystem, subprocess & CLIs](#os-filesystem-subprocess--clis) — Q77–Q96
- [Data formats, HTTP & REST APIs](#data-formats-http--rest-apis) — Q97–Q114
- [boto3 & AWS automation](#boto3--aws-automation) — Q115–Q134
- [Kubernetes, Docker & SSH clients](#kubernetes-docker--ssh-clients) — Q135–Q146
- [Concurrency: threads, processes & asyncio](#concurrency-threads-processes--asyncio) — Q147–Q160
- [Testing, services & metrics](#testing-services--metrics) — Q161–Q188
- [Best practices & rapid-fire commands](#best-practices--rapid-fire-commands) — Q189–Q200

---

## Language essentials & functions

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q1. Why is Python the default scripting language in so many DevOps teams, and where would you not use it? 🟢

<details>
<summary><b>Answer</b></summary>

Python has first-class SDKs for every cloud and platform (boto3, the Kubernetes client, Docker SDK, Azure and GCP libraries), ships on almost every Linux distro, is readable by people who are not full-time developers, and has a huge standard library for files, JSON, HTTP and processes. Ansible, SaltStack, the AWS CLI v1 and many CNCF tools are written in it.

Where we do not use it: a five-line glue job that is mostly piping commands (Bash is simpler), a small static binary that must run anywhere without a runtime (Go), or a high-throughput Kubernetes controller where Go's client-go and controller-runtime are the mature path.

</details>

### Q2. What is the difference between mutable and immutable types? Give examples. 🟢

<details>
<summary><b>Answer</b></summary>

A mutable object can change in place after creation; an immutable one cannot, so any "change" creates a new object.

- Mutable: `list`, `dict`, `set`, `bytearray`, most user-defined objects.
- Immutable: `int`, `float`, `bool`, `str`, `tuple`, `frozenset`, `bytes`.

It matters because names are references. If two variables point at the same list, appending through one is visible through the other. Only hashable (in practice, immutable) objects can be dict keys or set members.

</details>

### Q3. What does assignment do in Python, and how do shallow and deep copies differ? 🟢

<details>
<summary><b>Answer</b></summary>

Assignment never copies; it binds a name to an existing object. `b = a` makes both names point at the same list.

- Shallow copy (`a.copy()`, `list(a)`, `dict(a)`, `copy.copy(a)`) creates a new outer container but shares the nested objects.
- Deep copy (`copy.deepcopy(a)`) recursively copies everything.

```python
import copy
base = {"tags": {"env": "dev"}}
shallow = base.copy()
shallow["tags"]["env"] = "prod"   # also changes base!
deep = copy.deepcopy(base)
```

We hit this when we build per-environment configs from a shared template dict.

</details>

### Q4. Explain the mutable default argument bug and how to fix it. 🟡

<details>
<summary><b>Answer</b></summary>

Default values are evaluated once, when the function is defined, not on each call. A mutable default is therefore shared across calls.

```python
def add_tag(tag, tags=[]):      # bug
    tags.append(tag)
    return tags

add_tag("a")  # ['a']
add_tag("b")  # ['a', 'b']  surprise

def add_tag(tag, tags=None):    # fix
    tags = [] if tags is None else tags
    tags.append(tag)
    return tags
```

`ruff` flags this as `B006`, which is one reason we run it in CI.

</details>

### Q5. What is the difference between `==` and `is`? 🟢

<details>
<summary><b>Answer</b></summary>

`==` compares values (calls `__eq__`); `is` compares identity (same object in memory). Use `is` only for singletons: `None`, `True`, `False`, and sentinel objects. `x is None` is correct; `x == None` works but is unidiomatic and can be fooled by a custom `__eq__`. Never use `is` to compare strings or numbers, because small-int and string interning make it appear to work until it does not.

</details>

### Q6. What values are "falsy" in Python, and why does it matter in scripts? 🟢

<details>
<summary><b>Answer</b></summary>

Falsy: `None`, `False`, `0`, `0.0`, `""`, `[]`, `{}`, `set()`, `()`, and any object whose `__bool__` returns False or `__len__` returns 0. Everything else is truthy.

The trap is `if not value:` when `0` or `""` is a legitimate value. For example, `if not replicas:` treats `replicas=0` (a valid scale-down) as "not provided". Use `if replicas is None:` when you mean "missing".

</details>

### Q7. When do you choose a list, tuple, set or dict? 🟢

<details>
<summary><b>Answer</b></summary>

- **list**: ordered, mutable sequence; a list of instance IDs to process.
- **tuple**: ordered, immutable; fixed records like `(host, port)` or dict keys made of several fields.
- **set**: unordered, unique, O(1) membership; "which required tags are missing" is `required - set(tags)`.
- **dict**: key to value mapping with O(1) lookup, insertion-ordered since 3.7; tags, config, lookups by ID.

Choosing a set instead of a list for membership checks turns an O(n²) audit loop into O(n).

</details>

### Q8. Write a list, dict and set comprehension for a realistic ops task. 🟢

<details>
<summary><b>Answer</b></summary>

```python
instances = [...]  # from describe_instances

running_ids = [i["InstanceId"] for i in instances if i["State"]["Name"] == "running"]

tags_by_id = {i["InstanceId"]: {t["Key"]: t["Value"] for t in i.get("Tags", [])}
              for i in instances}

instance_types = {i["InstanceType"] for i in instances}
```

Comprehensions are faster than an explicit `append` loop and read as "what" rather than "how". If one needs more than one `if` and one `for`, write a normal loop for readability.

</details>

### Q9. What is the difference between a list comprehension and a generator expression? 🟡

<details>
<summary><b>Answer</b></summary>

A list comprehension `[...]` builds the whole list in memory immediately. A generator expression `(...)` yields one item at a time, on demand, and can only be iterated once.

```python
total = sum(len(line) for line in open("access.log", encoding="utf-8"))
```

That processes a 20 GB log in constant memory. Use a list when you need indexing, `len()`, or multiple passes; use a generator for streaming and for feeding `sum`, `any`, `all`, `max`.

</details>

### Q10. How do `any()` and `all()` short-circuit, and where do you use them? 🟢

<details>
<summary><b>Answer</b></summary>

`any(iterable)` returns True at the first truthy item; `all(iterable)` returns False at the first falsy item. Both stop consuming the iterable early, so with a generator they avoid unnecessary work.

```python
open_to_world = any(r.get("CidrIp") == "0.0.0.0/0" for r in perm["IpRanges"])
all_ready = all(c.status == "True" for c in node.status.conditions if c.type == "Ready")
```

Note `all([])` is True, which can make an empty result look like success.

</details>

### Q11. How do you safely read nested keys from an API response? 🟢

<details>
<summary><b>Answer</b></summary>

Use `dict.get()` with defaults so a missing key does not raise `KeyError`:

```python
name = next((t["Value"] for t in inst.get("Tags", []) if t["Key"] == "Name"), "unnamed")
public_ip = inst.get("PublicIpAddress")  # None if absent
state = inst.get("State", {}).get("Name", "unknown")
```

Use `d[key]` when the key must exist and a crash is the right behaviour; `.get()` when absence is normal. AWS omits keys like `Tags` and `PublicIpAddress` entirely rather than returning empty values.

</details>

### Q12. What do `collections.defaultdict`, `Counter` and `namedtuple` give you? 🟡

<details>
<summary><b>Answer</b></summary>

- `defaultdict(list)` creates a default value on first access, so grouping needs no `if key not in d` check.
- `Counter` counts hashable items and has `.most_common(n)`.
- `namedtuple` (or better, `typing.NamedTuple` / `dataclass`) gives tuples readable field names.

```python
from collections import Counter, defaultdict
by_type = defaultdict(list)
for i in instances:
    by_type[i["InstanceType"]].append(i["InstanceId"])
status_counts = Counter(line.split()[8] for line in open("access.log"))
print(status_counts.most_common(5))
```

</details>

### Q13. What is a dataclass and why use one in automation code? 🟡

<details>
<summary><b>Answer</b></summary>

`@dataclass` generates `__init__`, `__repr__` and `__eq__` from annotated fields. It gives you typed, self-documenting records instead of dicts with magic string keys.

```python
from dataclasses import dataclass, field

@dataclass(frozen=True)
class Finding:
    check: str
    resource_id: str
    severity: str = "LOW"
    tags: dict[str, str] = field(default_factory=dict)
```

`frozen=True` makes it immutable and hashable; `field(default_factory=...)` avoids the mutable default bug; `asdict()` turns it into JSON-ready data. For validation of untrusted input, `pydantic` models go further.

</details>

### Q14. How do f-strings work and what formatting tricks are useful in reports? 🟢

<details>
<summary><b>Answer</b></summary>

f-strings evaluate expressions inline: `f"{name} has {count} findings"`. Useful format specs:

```python
f"{cpu:.1f}%"         # 3.1%
f"{size_bytes:,}"     # 1,234,567
f"{name:<20}|"        # left-align in 20 chars
f"{ratio:.0%}"        # 87%
f"{value=}"           # value=42 (debugging, 3.8+)
```

Do not use f-strings to build SQL, shell commands or log messages. For logging, pass arguments (`log.info("found %d", n)`) so formatting is lazy and log aggregators can group messages.

</details>

### Q15. What are `*args` and `**kwargs`, and how do you unpack them at the call site? 🟢

<details>
<summary><b>Answer</b></summary>

`*args` collects extra positional arguments into a tuple; `**kwargs` collects extra keyword arguments into a dict. At a call site, `*` and `**` unpack.

```python
def run(cmd, *args, **kwargs):
    return subprocess.run([cmd, *args], **kwargs)

params = {"Filters": [{"Name": "tag:env", "Values": ["prod"]}]}
ec2.describe_instances(**params)
```

The `**params` pattern is common with boto3 because you build optional arguments conditionally and only pass the ones you have.

</details>

### Q16. What are keyword-only and positional-only parameters? 🟡

<details>
<summary><b>Answer</b></summary>

Parameters after a bare `*` must be passed by keyword; parameters before `/` must be passed positionally.

```python
def scale(deployment, replicas, /, *, namespace="default", dry_run=False): ...
scale("api", 3, dry_run=True)        # ok
scale("api", 3, True)                 # TypeError
```

We make destructive flags like `dry_run` and `force` keyword-only so nobody can accidentally pass `True` in the wrong position.

</details>

### Q17. Explain LEGB scope and the `global` / `nonlocal` keywords. 🟡

<details>
<summary><b>Answer</b></summary>

Name lookup goes Local, Enclosing (outer functions), Global (module), Built-in. Assigning to a name inside a function makes it local unless declared otherwise.

- `global x` rebinds a module-level name.
- `nonlocal x` rebinds a name in the enclosing function (closures).

`UnboundLocalError: cannot access local variable 'count'` happens when you read then assign a name that exists globally. In real code avoid `global`; pass state explicitly or use a class.

</details>

### Q18. What is a closure? Give a DevOps-flavoured example. 🟡

<details>
<summary><b>Answer</b></summary>

A closure is an inner function that remembers variables from the enclosing scope after that scope has returned.

```python
def make_tag_filter(key, value):
    def matches(resource):
        return any(t["Key"] == key and t["Value"] == value for t in resource.get("Tags", []))
    return matches

is_prod = make_tag_filter("Environment", "prod")
prod = [i for i in instances if is_prod(i)]
```

Watch the late-binding trap: lambdas created in a loop all see the last loop value unless you bind it, e.g. `lambda r, k=k: ...`.

</details>

### Q19. What is a decorator? Write a retry decorator. 🟡

<details>
<summary><b>Answer</b></summary>

A decorator takes a function and returns a new function that wraps it.

```python
import functools, logging, random, time
log = logging.getLogger(__name__)

def retry(times=5, base=0.5, exceptions=(ConnectionError,)):
    def deco(fn):
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):
            for attempt in range(1, times + 1):
                try:
                    return fn(*args, **kwargs)
                except exceptions as exc:
                    if attempt == times:
                        raise
                    delay = base * 2 ** (attempt - 1) + random.uniform(0, base)
                    log.warning("%s failed (%s), retry %d in %.1fs", fn.__name__, exc, attempt, delay)
                    time.sleep(delay)
        return wrapper
    return deco
```

In production we usually reach for `tenacity` instead of hand-rolling this.

</details>

### Q20. Why should every decorator use `functools.wraps`? 🟡

<details>
<summary><b>Answer</b></summary>

Without it the wrapper replaces the original's `__name__`, `__doc__`, `__module__`, `__qualname__` and signature metadata. Logs then say `wrapper failed`, `help()` shows nothing useful, click and FastAPI (which inspect signatures) break, and pytest fixtures or Flask routes collide because every view is named `wrapper`. `@functools.wraps(fn)` copies the metadata and sets `__wrapped__` so tools can reach the original.

</details>

### Q21. What is a generator function and why is it a good fit for paginated APIs? 🟡

<details>
<summary><b>Answer</b></summary>

A function containing `yield` returns a generator: it runs until the next `yield`, pauses, and resumes on the next iteration. That lets you hide pagination and stream items lazily.

```python
def all_pages(session, url):
    while url:
        resp = session.get(url, timeout=10)
        resp.raise_for_status()
        yield from resp.json()
        url = resp.links.get("next", {}).get("url")   # GitHub Link header

for repo in all_pages(s, "https://api.github.com/orgs/acme/repos?per_page=100"):
    ...
```

The caller can stop early (`break`) and no further pages are fetched.

</details>

### Q22. What does `yield from` do? 🔴

<details>
<summary><b>Answer</b></summary>

`yield from iterable` yields every item of a sub-iterable, and for sub-generators it also forwards `send()`, `throw()` and `close()` and returns the sub-generator's `return` value. In everyday DevOps code it is simply the clean way to flatten: `yield from page["Reservations"]` instead of a nested `for` with `yield`. It is also how generator-based coroutines were built before `async`/`await`.

</details>

### Q23. What are lambdas good for, and when should you avoid them? 🟢

<details>
<summary><b>Answer</b></summary>

A lambda is a one-expression anonymous function. Good uses are short key functions:

```python
newest = max(snapshots, key=lambda s: s["StartTime"])
by_cost = sorted(items, key=lambda x: (x["severity_rank"], -x["cost"]))
```

Avoid assigning lambdas to names (`f = lambda x: ...`; use `def`), multi-step logic, and lambdas created in loops that capture the loop variable. `operator.itemgetter("StartTime")` is a faster, clearer alternative for key lookups.

</details>

### Q24. How do `sorted()` and `list.sort()` differ, and how do you sort by multiple keys? 🟢

<details>
<summary><b>Answer</b></summary>

`sorted(iterable)` returns a new list; `list.sort()` sorts in place and returns `None` (a common bug is `x = x.sort()`). Both are stable and accept `key=` and `reverse=`.

```python
findings.sort(key=lambda f: (SEVERITY_RANK[f.severity], f.region, f.resource_id))
```

Tuple keys sort by the first element, then the next. Stability means you can also sort in multiple passes, least significant key first.

</details>

### Q25. What does `enumerate` and `zip` do, and what is `zip(strict=True)`? 🟢

<details>
<summary><b>Answer</b></summary>

`enumerate(seq, start=1)` yields `(index, item)`. `zip(a, b)` pairs items and stops at the shortest input, silently dropping extras. `zip(a, b, strict=True)` (3.10+) raises `ValueError` if lengths differ, which catches bugs like "12 hosts but 11 IPs".

```python
for n, (host, ip) in enumerate(zip(hosts, ips, strict=True), start=1):
    print(f"{n}. {host} -> {ip}")
```

</details>

### Q26. What is unpacking, including star-unpacking? 🟡

<details>
<summary><b>Answer</b></summary>

```python
host, port = "db.internal:5432".split(":")
first, *rest = sys.argv[1:]
*_, last = path.parts
merged = {**defaults, **overrides}      # later wins
cmd = ["kubectl", *base_args, "apply", "-f", path]
```

Unpacking raises `ValueError: too many values to unpack` (or "not enough") when counts do not match, which is a useful early failure when parsing `key=value` lines. Use `.split(":", 1)` or `.partition(":")` when the value itself may contain the separator.

</details>

### Q27. How do you merge dictionaries, and what about deep merges? 🔴

<details>
<summary><b>Answer</b></summary>

Shallow merge: `merged = base | override` (3.9+) or `{**base, **override}`; `base |= override` updates in place. Right-hand values win.

These replace nested dicts wholesale. For layered config (defaults, then environment, then CLI flags) you need a recursive merge:

```python
def deep_merge(a: dict, b: dict) -> dict:
    out = dict(a)
    for k, v in b.items():
        out[k] = deep_merge(out[k], v) if isinstance(out.get(k), dict) and isinstance(v, dict) else v
    return out
```

</details>

### Q28. What are type hints, and does Python enforce them at runtime? 🟢

<details>
<summary><b>Answer</b></summary>

Type hints annotate expected types: `def stop(ids: list[str], dry_run: bool = False) -> int:`. Python does not enforce them at runtime; they are metadata. Static checkers (`mypy`, `pyright`) use them to catch bugs before the code runs, IDEs use them for autocompletion, and libraries like pydantic, FastAPI and typer read them at runtime to validate and parse input.

</details>

### Q29. Explain `Optional`, union syntax, and `None` handling in type hints. 🟡

<details>
<summary><b>Answer</b></summary>

`Optional[str]` means `str | None` (the `|` syntax is available from 3.10). It does not mean "optional argument"; it means the value may be `None`.

```python
def find_instance(name: str) -> dict[str, Any] | None: ...

inst = find_instance("web-1")
inst["InstanceId"]          # mypy: Value of type "dict | None" is not indexable
if inst is not None:
    inst["InstanceId"]      # ok, narrowed
```

This forces you to handle the "not found" path, which is exactly the bug that pages someone at 3 AM.

</details>

### Q30. What are `TypedDict` and `Protocol`, and when are they useful? 🔴

<details>
<summary><b>Answer</b></summary>

- `TypedDict` types a dict with known string keys, perfect for API payloads you keep as dicts: `class Tag(TypedDict): Key: str; Value: str`. `boto3-stubs` uses TypedDicts for every boto3 response.
- `Protocol` is structural typing ("anything with a `.describe_instances()` method"), so you can type an injected client without depending on a concrete class, which makes fakes in tests type-check cleanly.

```python
class SupportsSend(Protocol):
    def send(self, msg: str) -> None: ...
```

</details>

### Q31. What are generics in type hints (`TypeVar`, `list[T]`)? 🔴

<details>
<summary><b>Answer</b></summary>

Generics let a function or class be typed in terms of its inputs:

```python
from typing import TypeVar
T = TypeVar("T")
def chunked(items: list[T], size: int) -> Iterator[list[T]]:
    for i in range(0, len(items), size):
        yield items[i:i + size]
```

`chunked(instance_ids, 50)` is inferred as `Iterator[list[str]]`. Python 3.12 adds the shorter syntax `def chunked[T](items: list[T], size: int) -> Iterator[list[T]]:`. Batching IDs is common because many AWS APIs cap a call (for example 1000 IDs for `describe_instances` filters, 200 for some tagging calls).

</details>

### Q32. What is `if __name__ == "__main__":` for? 🟢

<details>
<summary><b>Answer</b></summary>

When a file is run directly, `__name__` is `"__main__"`; when imported, it is the module name. The guard lets a file be both an importable module (for tests and reuse) and a script, without running side effects on import. It is also required on Windows and macOS (spawn start method) for `multiprocessing`, otherwise child processes re-import and re-execute your top-level code.

</details>

### Q33. How do modules, packages and `__init__.py` work? 🟢

<details>
<summary><b>Answer</b></summary>

A module is a `.py` file. A package is a directory of modules, traditionally marked with `__init__.py` (which runs on import and can re-export names). Without `__init__.py` you get a namespace package, which works but can confuse tools.

`import cloudaudit.checks` searches `sys.path`: the script's directory (or current directory), `PYTHONPATH`, then site-packages. Prefer absolute imports inside an installed package and use the `src/` layout so tests import the installed package, not a stray local folder.

</details>

### Q34. Why does naming a file `json.py` or `requests.py` break things? 🟡

<details>
<summary><b>Answer</b></summary>

The script's directory is first on `sys.path`, so `import json` finds your `json.py` instead of the standard library. You see confusing errors like `AttributeError: module 'json' has no attribute 'loads'` or `partially initialized module 'requests' has no attribute 'get' (most likely due to a circular import)`. Rename the file and delete the stale `__pycache__`. Check with `python -c "import json; print(json.__file__)"`.

</details>

### Q35. What is a class vs an instance attribute, and what goes wrong with mutable class attributes? 🟡

<details>
<summary><b>Answer</b></summary>

Class attributes are shared by all instances; instance attributes (set on `self` in `__init__`) are per object.

```python
class Inventory:
    hosts = []              # shared by every Inventory, bug
    def __init__(self):
        self.groups = {}    # per instance, correct
```

Appending to `self.hosts` mutates the shared list, so two inventories leak hosts into each other. Constants (`DEFAULT_REGION = "us-east-1"`) are fine as class attributes; mutable state belongs on the instance.

</details>

### Q36. What are `@staticmethod`, `@classmethod` and `@property`? 🟡

<details>
<summary><b>Answer</b></summary>

- `@staticmethod`: a plain function namespaced in the class, no `self`.
- `@classmethod`: receives the class (`cls`), typically for alternative constructors like `Config.from_yaml(path)`.
- `@property`: a method accessed like an attribute, used for computed or validated values.

```python
class Instance:
    def __init__(self, raw): self.raw = raw
    @property
    def name(self): return next((t["Value"] for t in self.raw.get("Tags", []) if t["Key"] == "Name"), "")
    @classmethod
    def from_id(cls, ec2, iid): return cls(ec2.describe_instances(InstanceIds=[iid])["Reservations"][0]["Instances"][0])
```

</details>

### Q37. What are dunder methods? Name the ones you actually use. 🟡

<details>
<summary><b>Answer</b></summary>

Double-underscore methods hook into language syntax. The useful ones in tooling:

- `__init__`, `__repr__` (debug output; dataclasses generate it)
- `__eq__`, `__hash__` (dedupe findings in a set)
- `__enter__`, `__exit__` (context managers)
- `__iter__`, `__len__`, `__contains__` (make an inventory iterable)
- `__call__` (callable objects)
- `__str__` (human output)

Implementing `__repr__` alone pays for itself the first time you read a log line.

</details>

### Q38. What does `enum.Enum` give you over string constants? 🟢

<details>
<summary><b>Answer</b></summary>

A closed set of named values: typos fail loudly (`Severity.HIHG` is an `AttributeError`), IDEs autocomplete, and mypy checks exhaustiveness. Subclass `str` (or use `StrEnum` in 3.11+) so values serialize to JSON naturally.

```python
from enum import StrEnum
class Env(StrEnum):
    DEV = "dev"
    PROD = "prod"
Env("prod") is Env.PROD   # True; Env("qa") raises ValueError
```

</details>

### Q39. What is `match` / `case` (structural pattern matching) and is it useful for ops code? 🔴

<details>
<summary><b>Answer</b></summary>

Python 3.10's `match` statement matches shapes, not just values, which is handy for webhook and event payloads:

```python
match event:
    case {"action": "opened", "pull_request": {"number": n}}:
        handle_pr_opened(n)
    case {"detail-type": "EC2 Instance State-change Notification", "detail": {"state": "stopped" as s}}:
        notify(s)
    case _:
        log.debug("ignored event")
```

It is not a C-style switch: a bare name in a pattern captures rather than compares, so compare constants with dotted names (`case Env.PROD:`) or guards (`case x if x == limit:`).

</details>

### Q40. What does the walrus operator `:=` do? 🟡

<details>
<summary><b>Answer</b></summary>

It assigns inside an expression, which removes duplicate calls in loops and comprehensions:

```python
while (chunk := f.read(65536)):
    h.update(chunk)

if (m := re.search(r"ERROR (\w+)", line)):
    codes[m.group(1)] += 1

big = [s for v in volumes if (s := v["Size"]) > 500]
```

Use it where it removes repetition; do not chain it into unreadable one-liners.

</details>

---

## Exceptions, context managers, files & logging

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q41. How does `try` / `except` / `else` / `finally` work? 🟢

<details>
<summary><b>Answer</b></summary>

- `try`: code that may raise.
- `except SomeError as e`: runs if that exception (or a subclass) is raised.
- `else`: runs only if `try` raised nothing; keep "success-only" code here so it is not accidentally covered by the `except`.
- `finally`: always runs, even on `return` or an uncaught exception; for cleanup.

```python
try:
    resp = s.get(url, timeout=5)
    resp.raise_for_status()
except requests.HTTPError as e:
    log.error("HTTP %s from %s", e.response.status_code, url)
    raise
else:
    data = resp.json()
finally:
    metrics.calls.inc()
```

</details>

### Q42. Why is a bare `except:` (or `except Exception: pass`) dangerous? 🟢

<details>
<summary><b>Answer</b></summary>

Bare `except:` catches `BaseException`, including `KeyboardInterrupt` and `SystemExit`, so Ctrl-C and `sys.exit()` stop working. `except Exception: pass` swallows every bug, typos included, and the script reports success while doing nothing. Catch the narrowest exception you can actually handle, log it with context, and re-raise anything you cannot handle. If you must catch broadly at the top level, log with `log.exception(...)` and exit non-zero.

</details>

### Q43. What is exception chaining, and what is `raise ... from`? 🟡

<details>
<summary><b>Answer</b></summary>

`raise NewError("...") from exc` attaches the original exception as `__cause__`, and the traceback shows both ("The above exception was the direct cause of..."). This lets you translate low-level errors into domain errors without losing the root cause.

```python
try:
    cfg = yaml.safe_load(path.read_text())
except (OSError, yaml.YAMLError) as exc:
    raise ConfigError(f"cannot load {path}") from exc
```

`raise ... from None` suppresses the context, which is occasionally right for user-facing CLI errors.

</details>

### Q44. When should you define custom exception classes? 🟡

<details>
<summary><b>Answer</b></summary>

When callers need to react differently to different failures. A small hierarchy is enough:

```python
class AuditError(Exception): """Base for this tool."""
class ConfigError(AuditError): ...
class PermissionDenied(AuditError): ...
```

The CLI entry point catches `AuditError` and maps subclasses to exit codes and friendly messages, while unexpected exceptions still produce a traceback. Do not create one class per call site; create them per decision a caller makes.

</details>

### Q45. What is the EAFP vs LBYL style, and which does Python prefer? 🟡

<details>
<summary><b>Answer</b></summary>

- LBYL (Look Before You Leap): `if path.exists(): path.read_text()`.
- EAFP (Easier to Ask Forgiveness than Permission): `try: path.read_text() except FileNotFoundError: ...`.

Python leans EAFP, and in ops code it is also more correct: between the check and the use, another process can delete the file or a resource can change state (a TOCTOU race). The same applies to AWS: try to create the bucket and handle `BucketAlreadyOwnedByYou` rather than checking first.

</details>

### Q46. How does a `with` statement work, and why use it for files? 🟢

<details>
<summary><b>Answer</b></summary>

`with` calls the object's `__enter__` at the start and guarantees `__exit__` at the end, even if an exception is raised. For files, `__exit__` closes the handle and flushes buffers.

```python
with open("/etc/hosts", encoding="utf-8") as f:
    for line in f:
        ...
```

Without it, an exception mid-loop leaks the file descriptor; in a long-running service that eventually becomes `OSError: [Errno 24] Too many open files`. The same pattern is used for locks, SSH and DB connections, temp dirs and `ThreadPoolExecutor`.

</details>

### Q47. How do you write your own context manager? 🔴

<details>
<summary><b>Answer</b></summary>

The easy way is `contextlib.contextmanager`: code before `yield` is setup, after `yield` is teardown, and wrap the `yield` in `try/finally` so teardown runs on errors.

```python
from contextlib import contextmanager
import os

@contextmanager
def chdir_temp(path):
    old = os.getcwd()
    os.chdir(path)
    try:
        yield path
    finally:
        os.chdir(old)
```

Python 3.11 ships `contextlib.chdir` for this exact case. For class-based managers implement `__enter__` and `__exit__(exc_type, exc, tb)`; returning True from `__exit__` suppresses the exception.

</details>

### Q48. What does `contextlib.ExitStack` solve? 🔴

<details>
<summary><b>Answer</b></summary>

It manages a dynamic number of context managers, for example opening one SSH connection per host where the host list is only known at runtime, and guarantees every one that opened is closed, in reverse order, even if a later one fails.

```python
from contextlib import ExitStack
with ExitStack() as stack:
    conns = [stack.enter_context(Connection(h)) for h in hosts]
    for c in conns:
        c.run("uptime")
```

It also supports `stack.callback(fn, *args)` for arbitrary cleanup.

</details>

### Q49. What file modes exist, and why always pass `encoding`? 🟢

<details>
<summary><b>Answer</b></summary>

Modes: `r` read, `w` truncate and write, `a` append, `x` create and fail if exists, `+` read/write, `b` binary. Text mode decodes bytes using `encoding`; if you omit it, Python uses the locale encoding, which is UTF-8 on most Linux boxes but can be `ascii` in a minimal container (`LANG` unset) or cp1252 on Windows. That produces `UnicodeDecodeError` only in production. Always pass `encoding="utf-8"` (3.15 makes UTF-8 the default; `PYTHONWARNDEFAULTENCODING=1` flags the omissions today).

</details>

### Q50. How do you read a huge log file efficiently? 🟡

<details>
<summary><b>Answer</b></summary>

Iterate the file object, which reads line by line with buffering, instead of `.read()` or `.readlines()` that load everything into memory.

```python
import gzip
from collections import Counter

def lines(path):
    opener = gzip.open if path.endswith(".gz") else open
    with opener(path, "rt", encoding="utf-8", errors="replace") as f:
        yield from f

codes = Counter(l.split()[8] for l in lines("access.log.gz") if len(l.split()) > 8)
```

`errors="replace"` keeps one bad byte from killing a multi-GB parse.

</details>

### Q51. How do you write a config file atomically? 🔴

<details>
<summary><b>Answer</b></summary>

Write to a temp file in the same directory, flush and fsync, then `os.replace()` it over the target. `os.replace` is atomic on POSIX when source and destination are on the same filesystem, so readers see either the old file or the new one, never a half-written file.

```python
import os, tempfile
def atomic_write(path, data):
    d = os.path.dirname(os.path.abspath(path))
    with tempfile.NamedTemporaryFile("w", dir=d, delete=False, encoding="utf-8") as tmp:
        tmp.write(data)
        tmp.flush()
        os.fsync(tmp.fileno())
    os.replace(tmp.name, path)
```

This is how we update nginx upstream files or kubeconfigs that other processes read.

</details>

### Q52. What is the `tempfile` module for? 🟡

<details>
<summary><b>Answer</b></summary>

It creates temp files and directories securely (unique names, `0600` permissions, no race with predictable `/tmp/foo` paths).

```python
import tempfile
with tempfile.TemporaryDirectory() as d:
    subprocess.run(["git", "clone", "--depth=1", repo, d], check=True)
    ...  # directory deleted on exit
```

Never build temp paths by hand like `/tmp/deploy.sh`; another user can pre-create or symlink that path.

</details>

### Q53. Why use `logging` instead of `print`? 🟢

<details>
<summary><b>Answer</b></summary>

`logging` gives you levels (DEBUG, INFO, WARNING, ERROR, CRITICAL) you can change without editing code, timestamps and logger names, output to stderr so stdout stays clean for data, handlers for files/syslog/JSON, and library logs (boto3, urllib3) in the same stream. `print` gives none of that and mixes diagnostics with the report a pipeline might be parsing.

</details>

### Q54. How should logging be configured in a library vs an application? 🟡

<details>
<summary><b>Answer</b></summary>

Libraries and modules only create loggers: `log = logging.getLogger(__name__)`. They never call `basicConfig` or add handlers, so the application stays in control.

The application entry point configures logging once:

```python
logging.basicConfig(level=os.getenv("LOG_LEVEL", "INFO"),
                    format="%(asctime)s %(levelname)s %(name)s: %(message)s")
logging.getLogger("botocore").setLevel(logging.WARNING)
```

Calling `basicConfig` in an imported module is why "my log level change does nothing": only the first `basicConfig` call has effect (unless `force=True`).

</details>

### Q55. What is the logger hierarchy and propagation? 🔴

<details>
<summary><b>Answer</b></summary>

Loggers are named with dots, forming a tree: `cloudaudit.checks` is a child of `cloudaudit`, which is a child of the root logger. A record is handled by the logger's own handlers and then propagates to ancestors' handlers (unless `propagate = False`). The effective level is the first level set walking up the tree.

Two classic bugs: duplicate log lines (a handler was added both to a child and to root), and silence (a library logger set to WARNING while you expected DEBUG). `logging.getLogger("urllib3").setLevel(logging.WARNING)` quiets noisy dependencies.

</details>

### Q56. How do you log an exception with its traceback? 🟢

<details>
<summary><b>Answer</b></summary>

Inside an `except` block, use `log.exception("msg")`, which logs at ERROR and appends the traceback. At other levels pass `exc_info=True`.

```python
try:
    client.delete_snapshot(SnapshotId=sid)
except ClientError:
    log.exception("failed to delete %s", sid)
    raise
```

`log.error(str(e))` loses the stack, which is the one thing you need at 3 AM.

</details>

### Q57. How do you produce structured (JSON) logs for a container? 🟡

<details>
<summary><b>Answer</b></summary>

Write one JSON object per line to stdout so Fluent Bit, CloudWatch or Loki can parse fields without regex. Use a library like `python-json-logger` or `structlog`, or a small formatter:

```python
class JsonFormatter(logging.Formatter):
    def format(self, r):
        return json.dumps({"ts": self.formatTime(r), "level": r.levelname,
                           "logger": r.name, "msg": r.getMessage(),
                           **getattr(r, "ctx", {})})

handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(JsonFormatter())
logging.basicConfig(level="INFO", handlers=[handler])
log.info("snapshot deleted", extra={"ctx": {"snapshot_id": sid, "region": region}})
```

</details>

### Q58. Why use `log.info("x=%s", x)` rather than an f-string in log calls? 🟡

<details>
<summary><b>Answer</b></summary>

With `%s` arguments, formatting only happens if the record is actually emitted, so a `log.debug` in a hot loop costs almost nothing when DEBUG is off. Aggregators like Sentry also group records by the unformatted template, so "failed to delete %s" is one issue rather than 10,000 unique ones. `ruff` rule `G004` flags f-strings in logging calls.

</details>

### Q59. How do you avoid leaking secrets into logs? 🔴

<details>
<summary><b>Answer</b></summary>

- Never log whole request objects, headers, environment dumps or config dicts.
- Wrap secrets in a type whose `__repr__` and `__str__` return `"****"` (pydantic's `SecretStr` does this).
- Add a `logging.Filter` that redacts patterns (`AKIA[0-9A-Z]{16}`, `Bearer \S+`, `password=\S+`) as a safety net.
- Keep urllib3/botocore at WARNING in production; DEBUG for botocore logs full request bodies.
- Mask in CI (`::add-mask::` in GitHub Actions).

Redaction filters are a backstop, not the control. The control is not logging the value at all.

</details>

### Q60. How do `sys.exit`, exit codes and uncaught exceptions relate? 🟢

<details>
<summary><b>Answer</b></summary>

`sys.exit(n)` raises `SystemExit`; the interpreter exits with `n` (0 success, non-zero failure). `sys.exit("message")` prints the message to stderr and exits 1. An uncaught exception prints a traceback and exits 1. A script that catches everything and falls off the end exits 0, which CI treats as success. Shell conventions: 2 for usage errors (argparse and click use it), 126/127 for not executable/not found, 128+N for killed by signal N.

</details>

---

## Environments, dependencies & packaging

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q61. What is a virtual environment and why do we always use one? 🟢

<details>
<summary><b>Answer</b></summary>

A venv is an isolated directory with its own `python` symlink, `pip` and `site-packages`. Each project gets its own dependency versions, nothing touches the system Python that the OS (and tools like `apt`, `yum`, `cloud-init`) depend on, and the environment can be deleted and rebuilt at will.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
deactivate
```

Activation only edits `PATH`; you can skip it and call `.venv/bin/python` directly, which is what cron jobs and systemd units should do.

</details>

### Q62. What does `error: externally-managed-environment` mean? 🟢

<details>
<summary><b>Answer</b></summary>

PEP 668: Debian 12+, Ubuntu 23.04+, Fedora and Homebrew Python mark the system interpreter as managed by the OS package manager, and `pip install` into it is refused, to stop pip from breaking OS tools. Fixes, in order of preference:

- `python3 -m venv .venv` and install there.
- `pipx install <tool>` for CLI applications.
- `apt install python3-<pkg>` for OS-level needs.

`--break-system-packages` exists, but using it on a server is how you break `apt` itself.

</details>

### Q63. `requirements.txt` vs `pyproject.toml`: what goes where? 🟡

<details>
<summary><b>Answer</b></summary>

- `pyproject.toml` (PEP 621) declares the project: name, version, Python requirement, abstract dependencies with ranges (`boto3>=1.35`), optional extras, entry points, and tool config (ruff, mypy, pytest).
- A lock file or a fully pinned `requirements.txt` records the exact resolved versions of everything, including transitive dependencies, for reproducible installs.

Libraries publish ranges; applications and Docker images install from a lock. We generate the lock with `pip-compile` (pip-tools), `uv pip compile` or `uv lock`, and commit it.

</details>

### Q64. Why is `pip freeze > requirements.txt` not a great lock strategy? 🟡

<details>
<summary><b>Answer</b></summary>

It dumps whatever happens to be installed in the current environment, including leftovers from experiments and tools like `pytest` or `ipython`, with no record of which packages you actually depend on directly. It has no hashes and no separation between prod and dev dependencies. Better: keep direct dependencies in `pyproject.toml` or `requirements.in`, compile them into a pinned, hashed lock (`pip-compile --generate-hashes`), and install with `pip install --require-hashes -r requirements.txt`.

</details>

### Q65. What do version specifiers like `~=`, `==`, `>=,<` mean? 🟢

<details>
<summary><b>Answer</b></summary>

- `==1.35.2`: exactly that version.
- `>=1.35,<2`: range; allows minor and patch upgrades but not the next major.
- `~=1.35.2`: compatible release, means `>=1.35.2,<1.36`; `~=1.35` means `>=1.35,<2`.
- `!=1.35.3`: exclude a broken version.

Applications pin exactly via a lock file; libraries use ranges so they do not create resolution conflicts for their users.

</details>

### Q66. What are `pipx` and `uv`, and when do you use them? 🟡

<details>
<summary><b>Answer</b></summary>

- **pipx** installs Python CLI applications (ansible-core, awscli-local, cookiecutter, pre-commit) each into its own venv and puts the executable on `PATH`. No dependency clashes, no system pollution.
- **uv** (from Astral) is a Rust-based replacement for pip, pip-tools, venv, pipx and pyenv. It resolves and installs much faster, supports `uv venv`, `uv pip install`, `uv lock`, `uv run`, `uvx <tool>` and can install Python versions itself.

In CI, uv often cuts dependency install time from minutes to seconds, which is why many teams have moved to it.

</details>

### Q67. How do you manage multiple Python versions on one machine? 🟡

<details>
<summary><b>Answer</b></summary>

Do not replace the OS `python3`. Install additional versions side by side using `pyenv`, `uv python install 3.12`, the deadsnakes PPA on Ubuntu, or official installers, then create venvs from the version you need: `python3.12 -m venv .venv` or `uv venv --python 3.12`. Pin the version in `.python-version` and in `requires-python`. In Docker, the base image tag (`python:3.12-slim`) is the version pin.

</details>

### Q68. What is a wheel vs an sdist? 🟡

<details>
<summary><b>Answer</b></summary>

- **sdist** (`.tar.gz`): source distribution; installing it runs a build step, which for C extensions needs a compiler and headers.
- **wheel** (`.whl`): a pre-built zip that is simply unpacked into site-packages. Pure-Python wheels are `py3-none-any`; binary wheels carry platform tags like `manylinux_2_28_x86_64` or `musllinux_1_2_aarch64`.

`pip` prefers wheels. When none matches your platform (Alpine/musl, new Python versions, ARM), it falls back to the sdist and compiles, which is why `pip install` on Alpine can take 20 minutes or fail with `gcc: not found`.

</details>

### Q69. How do you build and publish a Python package? 🟡

<details>
<summary><b>Answer</b></summary>

```bash
pip install build twine
python -m build              # dist/*.whl and dist/*.tar.gz
twine check dist/*
twine upload --repository-url https://pypi.internal/ dist/*
```

The build backend (setuptools, hatchling, flit-core, pdm-backend, poetry-core) is declared in `[build-system]`. For internal tools we publish to a private index (CodeArtifact, Artifactory, Nexus, GitLab package registry). For PyPI, use Trusted Publishing from CI via OIDC instead of API tokens.

</details>

### Q70. What is a console script entry point? 🟢

<details>
<summary><b>Answer</b></summary>

An entry in `pyproject.toml` that tells the installer to generate an executable wrapper on `PATH`:

```toml
[project.scripts]
cloudaudit = "cloudaudit.cli:main"
```

After `pip install .`, `cloudaudit --help` works from anywhere, using the venv's interpreter. It is cleaner than `python path/to/script.py` and is what makes `ENTRYPOINT ["cloudaudit"]` possible in a Dockerfile.

</details>

### Q71. What is an editable install (`pip install -e .`)? 🟢

<details>
<summary><b>Answer</b></summary>

It installs a package that points at your source tree instead of copying files, so code changes take effect without reinstalling. `pip install -e ".[dev]"` also installs the `dev` extras (pytest, moto, ruff). It is for local development and CI tests, not for production images, where you install the built wheel.

</details>

### Q72. Why use the `src/` layout? 🔴

<details>
<summary><b>Answer</b></summary>

With `src/cloudaudit/` instead of `cloudaudit/` at the repo root, the package is not importable just because you are in the repo directory. Tests therefore import the installed package, which catches packaging bugs (missing modules, missing package data, bad `packages.find` config) before you ship a broken wheel. It also prevents the accidental shadowing of installed packages by local folders.

</details>

### Q73. How do you install dependencies from a private index safely? 🔴

<details>
<summary><b>Answer</b></summary>

Use `--index-url` to point at the private index (which proxies PyPI), not `--extra-index-url`. With extra indexes pip picks the highest version across all indexes, so an attacker can publish `acme-utils 99.0` on public PyPI and win (dependency confusion). Also:

- Pin and hash (`--require-hashes`).
- Keep credentials out of URLs in files: use `keyring`, `PIP_INDEX_URL` from a secret, or `aws codeartifact login --tool pip`.
- Reserve your internal names on PyPI.

</details>

### Q74. What does `python -m` do, and why prefer `python -m pip`? 🟢

<details>
<summary><b>Answer</b></summary>

`python -m module` runs a module as a script using that interpreter's `sys.path`. `python -m pip install ...` guarantees pip installs into the same interpreter you are running, avoiding "I installed it but `ModuleNotFoundError`" when `pip` on `PATH` belongs to a different Python. Other useful ones: `python -m venv`, `python -m http.server 8000`, `python -m json.tool file.json`, `python -m pytest`, `python -m cProfile`.

</details>

### Q75. How do you keep dependencies up to date without breaking things? 🟡

<details>
<summary><b>Answer</b></summary>

- Commit a lock file; never float in production.
- Let Dependabot or Renovate open PRs for updates, grouped (for example all `boto3`/`botocore`/`boto3-stubs` together, since they release almost daily).
- CI runs tests on every update PR.
- Run `pip-audit` (or Trivy/OSV-Scanner) on the lock file to catch known CVEs.
- `pip list --outdated` for a quick local view.

</details>

### Q76. What are optional dependencies (extras)? 🟡

<details>
<summary><b>Answer</b></summary>

Named groups of extra dependencies declared in `[project.optional-dependencies]`, installed with `pip install "pkg[extra]"`. We use `dev` for test and lint tools, and feature extras like `moto[ec2,s3]` or `requests[socks]` pull in only what a feature needs. Quote the argument in shells like zsh, where `[` is a glob character (`zsh: no matches found: .[dev]`).

</details>

---

## OS, filesystem, subprocess & CLIs

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q77. `os.path` vs `pathlib`: which do you use and why? 🟢

<details>
<summary><b>Answer</b></summary>

`pathlib.Path` is the modern, object-oriented API and what we use in new code. Paths compose with `/`, and methods cover most file tasks:

```python
from pathlib import Path
cfg = Path.home() / ".aws" / "config"
cfg.exists(); cfg.read_text(encoding="utf-8")
for p in Path("/var/log/app").glob("*.log"): print(p.name, p.stat().st_size)
Path("reports").mkdir(parents=True, exist_ok=True)
p.suffix, p.stem, p.parent, p.with_suffix(".bak")
```

`os.path` still works and you will read plenty of it. Most standard library functions accept `Path` objects directly.

</details>

### Q78. How do you read environment variables safely? 🟢

<details>
<summary><b>Answer</b></summary>

```python
import os
region = os.environ.get("AWS_REGION", "us-east-1")   # optional with default
token = os.environ["GITHUB_TOKEN"]                    # required: KeyError if missing
debug = os.getenv("DEBUG", "false").lower() in {"1", "true", "yes"}
```

Environment values are always strings, so parse them explicitly (`int(os.getenv("WORKERS", "4"))`). Validate required variables at startup and fail fast with a clear message rather than deep inside the run. `os.environ` changes only affect the current process and its children.

</details>

### Q79. How do you walk a directory tree and find files? 🟢

<details>
<summary><b>Answer</b></summary>

```python
from pathlib import Path
big = [p for p in Path("/var/log").rglob("*.log") if p.is_file() and p.stat().st_size > 100 * 2**20]

import os
for root, dirs, files in os.walk("/etc"):
    dirs[:] = [d for d in dirs if not d.startswith(".")]   # prune in place
    for f in files:
        ...
```

`os.walk` lets you prune directories as you go, which matters on huge trees; `rglob` is shorter. Catch `PermissionError` when walking system paths as a non-root user.

</details>

### Q80. What does `shutil` give you that `os` does not? 🟢

<details>
<summary><b>Answer</b></summary>

High-level file operations:

- `shutil.copy2(src, dst)` copy with metadata; `copytree(src, dst, dirs_exist_ok=True)`.
- `shutil.move`, `shutil.rmtree(path)` (careful).
- `shutil.make_archive("backup", "gztar", root_dir="/etc/app")`, `unpack_archive`.
- `shutil.disk_usage("/")` returns total/used/free.
- `shutil.which("kubectl")` finds an executable on `PATH` (check before calling it).
- `shutil.chown(path, user, group)`.

</details>

### Q81. How do you run an external command correctly with `subprocess`? 🟢

<details>
<summary><b>Answer</b></summary>

```python
import subprocess
result = subprocess.run(
    ["kubectl", "get", "pods", "-n", ns, "-o", "json"],
    check=True,            # raise CalledProcessError on non-zero exit
    capture_output=True,   # capture stdout and stderr
    text=True,             # str instead of bytes
    timeout=60,            # raise TimeoutExpired instead of hanging forever
)
pods = json.loads(result.stdout)
```

Pass a list of arguments, not a string, so there is no shell parsing and no injection. Always set a timeout for anything that talks to the network.

</details>

### Q82. Why is `shell=True` dangerous, and when is it acceptable? 🟡

<details>
<summary><b>Answer</b></summary>

With `shell=True`, the string is passed to `/bin/sh -c`, so any shell metacharacters in interpolated input execute. `subprocess.run(f"git checkout {branch}", shell=True)` with `branch = "main; rm -rf ~"` is command injection.

Acceptable only with fully constant strings where you genuinely need shell features (pipes, globbing) and no user input. Even then it is usually better to do the pipe in Python or quote with `shlex.quote()`. Bandit flags this as `B602`.

</details>

### Q83. How do you handle a failed command and show useful output? 🟡

<details>
<summary><b>Answer</b></summary>

```python
try:
    subprocess.run(cmd, check=True, capture_output=True, text=True, timeout=120)
except subprocess.CalledProcessError as e:
    log.error("%s exited %d\nstdout:\n%s\nstderr:\n%s",
              shlex.join(e.cmd), e.returncode, e.stdout, e.stderr)
    raise
except subprocess.TimeoutExpired as e:
    log.error("%s timed out after %ss", shlex.join(e.cmd), e.timeout)
    raise
except FileNotFoundError:
    sys.exit(f"{cmd[0]} not found on PATH")
```

`FileNotFoundError` means the executable itself was not found, which is different from the command failing.

</details>

### Q84. How do you stream a long-running command's output line by line? 🔴

<details>
<summary><b>Answer</b></summary>

Use `Popen` and iterate stdout, so logs appear live instead of after the process ends:

```python
with subprocess.Popen(["terraform", "apply", "-auto-approve"], stdout=subprocess.PIPE,
                      stderr=subprocess.STDOUT, text=True, bufsize=1) as p:
    for line in p.stdout:
        log.info("tf: %s", line.rstrip())
if p.returncode != 0:
    raise RuntimeError(f"terraform exited {p.returncode}")
```

Merging stderr into stdout avoids the deadlock you get when reading one pipe while the other fills its buffer.

</details>

### Q85. What is the pipe-buffer deadlock with `Popen`? 🔴

<details>
<summary><b>Answer</b></summary>

OS pipes have a limited buffer (64 KiB on Linux). If you create both `stdout=PIPE` and `stderr=PIPE` and then call `p.wait()` or read only one pipe, the child blocks writing to the full pipe while you block waiting for it: neither progresses. Fixes: use `p.communicate()` (reads both concurrently), `subprocess.run(capture_output=True)` which uses `communicate`, merge stderr into stdout, or read pipes from separate threads. The `subprocess` docs warn about this explicitly for `wait()`.

</details>

### Q86. How do `shlex.split`, `shlex.quote` and `shlex.join` help? 🟡

<details>
<summary><b>Answer</b></summary>

- `shlex.split('ssh -o "StrictHostKeyChecking no" host')` turns a shell-like string into a correct argument list (respects quotes).
- `shlex.quote(value)` safely quotes a value for inclusion in a shell string, for example a command sent over SSH, where a remote shell will parse it.
- `shlex.join(args)` renders a list as a copy-pasteable command for logs.

Over SSH there is always a remote shell, so quoting matters even when the local call uses a list.

</details>

### Q87. How do you handle signals like SIGTERM in a Python process? 🔴

<details>
<summary><b>Answer</b></summary>

Kubernetes and Docker send SIGTERM and then SIGKILL after the grace period. Install a handler that sets a flag, finish the current unit of work, and exit cleanly.

```python
import signal, threading
stop = threading.Event()
def handle(signum, frame):
    log.info("received %s, shutting down", signal.Signals(signum).name)
    stop.set()
signal.signal(signal.SIGTERM, handle)
signal.signal(signal.SIGINT, handle)

while not stop.is_set():
    process_one_batch()
    stop.wait(5)
```

Handlers run only in the main thread. If Python is not PID 1 (a shell wrapper is), it may never see the signal, so use exec-form `ENTRYPOINT`.

</details>

### Q88. How do you parse command-line arguments with `argparse`? 🟢

<details>
<summary><b>Answer</b></summary>

```python
import argparse
p = argparse.ArgumentParser(description="Stop idle dev instances")
p.add_argument("--region", default=os.getenv("AWS_REGION", "us-east-1"))
p.add_argument("--tag", action="append", default=[], help="key=value, repeatable")
p.add_argument("--days", type=int, default=7)
p.add_argument("--dry-run", action="store_true")
p.add_argument("-v", "--verbose", action="count", default=0)
args = p.parse_args()
```

argparse generates `--help`, validates types, and exits with code 2 and a usage message on bad input. It is in the standard library, so scripts that must run on a bare host prefer it.

</details>

### Q89. How do you build subcommands with `argparse`? 🟡

<details>
<summary><b>Answer</b></summary>

```python
p = argparse.ArgumentParser(prog="ops")
sub = p.add_subparsers(dest="command", required=True)

s = sub.add_parser("snapshot", help="snapshot volumes")
s.add_argument("--volume-id", required=True)
s.set_defaults(func=cmd_snapshot)

c = sub.add_parser("cleanup", help="delete old snapshots")
c.add_argument("--older-than", type=int, default=30)
c.set_defaults(func=cmd_cleanup)

args = p.parse_args()
sys.exit(args.func(args))
```

`set_defaults(func=...)` is the standard dispatch trick; each handler returns an exit code.

</details>

### Q90. Why might you choose `click` (or `typer`) over `argparse`? 🟡

<details>
<summary><b>Answer</b></summary>

`click` gives decorators for options and commands, nested command groups, `envvar=` defaults, `type=click.Choice([...])` and `click.Path(exists=True)` validation, prompts and confirmation (`@click.confirmation_option`), colored output, and `click.testing.CliRunner` for tests without spawning processes. `typer` builds on click using type hints.

```python
@click.command()
@click.option("--region", envvar="AWS_REGION", required=True)
@click.option("--dry-run/--no-dry-run", default=True)
def main(region, dry_run): ...
```

Trade-off: a third-party dependency, which matters for scripts dropped onto bare servers.

</details>

### Q91. What makes a CLI tool "pipeline-friendly"? 🟡

<details>
<summary><b>Answer</b></summary>

- Data on stdout, logs and progress on stderr.
- Meaningful exit codes: 0 ok, 1 error, 2 usage or "findings found" (documented).
- Machine-readable output option (`--format json`) alongside human output.
- Non-interactive by default: no prompts when stdin is not a TTY; `--yes` to confirm.
- Configuration via flags and env vars, no hidden state files.
- `--dry-run` for anything destructive, idempotent when re-run.
- No colors when output is not a TTY (`NO_COLOR` respected).

</details>

### Q92. How do you implement a proper `--dry-run`? 🟡

<details>
<summary><b>Answer</b></summary>

Compute the full plan first, print it, and only execute when not in dry-run mode, so dry-run exercises the same code path up to the side effect.

```python
plan = [s for s in snapshots if is_expired(s)]
for s in plan:
    log.info("%s delete %s", "[dry-run] would" if dry_run else "will", s["SnapshotId"])
if not dry_run:
    for s in plan:
        ec2.delete_snapshot(SnapshotId=s["SnapshotId"])
```

Many AWS APIs also accept `DryRun=True`, which validates IAM permissions server-side and returns `DryRunOperation` on success; that is a stronger test than a local flag.

</details>

### Q93. How do you check whether stdout is a terminal, and why would you? 🟡

<details>
<summary><b>Answer</b></summary>

`sys.stdout.isatty()` is True for an interactive terminal and False when piped or redirected (CI logs, `| jq`, `> file`). Use it to disable colors and progress bars, emit JSON by default in pipelines, and refuse interactive prompts. `sys.stdin.isatty()` tells you if you can prompt the user at all.

</details>

### Q94. How do you get host information (hostname, CPU, memory, disk) from Python? 🟡

<details>
<summary><b>Answer</b></summary>

```python
import os, platform, shutil, socket
socket.gethostname(); platform.system(); platform.release()
os.cpu_count(); os.getloadavg()
shutil.disk_usage("/")          # total, used, free
```

For memory and per-process stats use `psutil`: `psutil.virtual_memory().percent`, `psutil.process_iter(["pid", "name", "memory_info"])`, `psutil.net_connections()`. In containers, remember `os.cpu_count()` reports host CPUs, not your cgroup quota; read `/sys/fs/cgroup/cpu.max` or use `os.process_cpu_count()` (3.13) for affinity-aware counts.

</details>

### Q95. How do you work with dates, times and time zones correctly? 🟡

<details>
<summary><b>Answer</b></summary>

Always use timezone-aware UTC datetimes in automation.

```python
from datetime import UTC, datetime, timedelta
now = datetime.now(UTC)
cutoff = now - timedelta(days=90)
datetime.fromisoformat("2026-01-05T10:00:00+00:00")
now.isoformat(timespec="seconds")
```

boto3 returns aware datetimes (`StartTime`, `LaunchTime`). Comparing them with a naive `datetime.now()` raises `TypeError: can't compare offset-naive and offset-aware datetimes`. `datetime.utcnow()` is deprecated since 3.12 because it returns a naive value. For local display use `zoneinfo.ZoneInfo("Asia/Kolkata")`.

</details>

### Q96. How do you use regular expressions for log parsing efficiently? 🟡

<details>
<summary><b>Answer</b></summary>

Compile once, use named groups, and anchor patterns.

```python
import re
LINE = re.compile(r'^(?P<ip>\S+) \S+ \S+ \[(?P<ts>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+) [^"]*" (?P<status>\d{3}) (?P<bytes>\d+|-)')
for line in f:
    if (m := LINE.match(line)):
        status = int(m["status"])
```

Use raw strings (`r"..."`) so backslashes are not interpreted twice. `re.match` anchors at the start, `re.search` scans, `re.fullmatch` must match all. For simple splits, `str.split()` is much faster than a regex.

</details>

---

## Data formats, HTTP & REST APIs

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q97. How do you read and write JSON, and what are the common gotchas? 🟢

<details>
<summary><b>Answer</b></summary>

```python
import json
data = json.loads(text)                      # str -> obj
with open("state.json", encoding="utf-8") as f:
    data = json.load(f)                      # file -> obj
json.dumps(data, indent=2, sort_keys=True, default=str)
```

Gotchas: `datetime`, `Decimal` and `set` are not serializable (`TypeError: Object of type datetime is not JSON serializable`; pass `default=str` or a custom function). JSON keys are always strings, so `{1: "a"}` round-trips as `{"1": "a"}`. `json.load` reads the whole file; for huge files use JSON Lines.

</details>

### Q98. Why must you use `yaml.safe_load` rather than `yaml.load`? 🟢

<details>
<summary><b>Answer</b></summary>

PyYAML's full loader can construct arbitrary Python objects from tags like `!!python/object/apply:os.system`, so loading an untrusted YAML file can execute code. `yaml.safe_load` only builds plain types (dict, list, str, int, float, bool, None, dates). Since PyYAML 5.1, calling `yaml.load` without a `Loader` warns, and 6.0 made the argument required. For writing, `yaml.safe_dump(data, sort_keys=False)`.

</details>

### Q99. How do you load a multi-document Kubernetes YAML file? 🟡

<details>
<summary><b>Answer</b></summary>

```python
import yaml
with open("manifests.yaml", encoding="utf-8") as f:
    docs = [d for d in yaml.safe_load_all(f) if d]   # skip empty docs between ---
for d in docs:
    print(d["kind"], d["metadata"]["name"])
```

`yaml.safe_load` on a multi-document file raises `expected a single document in the stream`. Watch for YAML 1.1 surprises in PyYAML: unquoted `on`, `yes`, `no` become booleans and `010` can become octal, which is why we quote values in manifests. `ruamel.yaml` preserves comments and ordering when you need to edit files round-trip.

</details>

### Q100. How do you read and write CSV files properly? 🟢

<details>
<summary><b>Answer</b></summary>

```python
import csv
with open("inventory.csv", newline="", encoding="utf-8") as f:
    for row in csv.DictReader(f):
        print(row["hostname"], row["ip"])

with open("report.csv", "w", newline="", encoding="utf-8") as f:
    w = csv.DictWriter(f, fieldnames=["id", "severity", "detail"])
    w.writeheader()
    w.writerows(findings)
```

Open with `newline=""` or you get blank lines on Windows and broken quoted newlines. The csv module handles quoting of commas and quotes in fields; never build CSV with `",".join(...)`.

</details>

### Q101. How do you make an HTTP request with `requests` the right way? 🟢

<details>
<summary><b>Answer</b></summary>

```python
import requests
resp = requests.get("https://api.github.com/repos/acme/app/releases/latest",
                    headers={"Authorization": f"Bearer {token}",
                             "Accept": "application/vnd.github+json"},
                    timeout=(3.05, 15))
resp.raise_for_status()
tag = resp.json()["tag_name"]
```

Three non-negotiables: a timeout (the default is to wait forever), `raise_for_status()` (requests does not raise on 4xx/5xx), and credentials from the environment or a secrets manager, never hardcoded.

</details>

### Q102. Why use a `requests.Session`? 🟡

<details>
<summary><b>Answer</b></summary>

A Session reuses TCP and TLS connections through a connection pool (huge latency win for many calls to the same host), keeps default headers and auth in one place, persists cookies, and is where you mount retry adapters.

```python
s = requests.Session()
s.headers.update({"Authorization": f"Bearer {token}", "User-Agent": "ops-bot/1.0"})
for repo in repos:
    s.get(f"https://api.github.com/repos/acme/{repo}", timeout=10)
```

Sessions are generally used per thread; sharing one across threads usually works but is not officially guaranteed thread-safe.

</details>

### Q103. How do you add retries with exponential backoff to `requests`? 🟡

<details>
<summary><b>Answer</b></summary>

Mount an `HTTPAdapter` configured with urllib3's `Retry`:

```python
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

retry = Retry(total=5, connect=5, read=3, backoff_factor=0.5,
              status_forcelist=(429, 500, 502, 503, 504),
              allowed_methods=frozenset({"GET", "HEAD", "PUT", "DELETE", "OPTIONS"}),
              respect_retry_after_header=True)
s = requests.Session()
s.mount("https://", HTTPAdapter(max_retries=retry, pool_maxsize=20))
```

Backoff grows as `backoff_factor * 2 ** (retry - 1)`, capped by `backoff_max`. POST is excluded by default because it is not idempotent.

</details>

### Q104. Why add jitter to backoff, and what is "full jitter"? 🔴

<details>
<summary><b>Answer</b></summary>

If 500 workers fail at the same moment and all retry at exactly 1s, 2s, 4s, they hit the recovering service in synchronized waves (thundering herd). Jitter spreads retries out. "Full jitter" sleeps a random time between 0 and the exponential cap: `sleep = random.uniform(0, min(cap, base * 2 ** attempt))`. AWS's own SDK retry modes use jittered exponential backoff, and `tenacity` offers `wait_random_exponential`.

</details>

### Q105. Which HTTP errors should you retry, and which should you not? 🟡

<details>
<summary><b>Answer</b></summary>

Retry: connection errors, timeouts, 429 (honor `Retry-After`), 502, 503, 504, and usually 500 for idempotent calls.

Do not retry: 400 (your request is wrong), 401/403 (credentials or permissions; retrying can lock accounts), 404, 409 conflicts (handle explicitly), 422 validation errors.

Retrying non-idempotent POSTs can create duplicates (two deployments, two tickets), unless the API supports an idempotency key header.

</details>

### Q106. What is idempotency, and how do you design idempotent automation? 🟡

<details>
<summary><b>Answer</b></summary>

An operation is idempotent if running it once or ten times leaves the system in the same state. It is what makes retries and re-runs safe.

- Check-then-act on desired state: "ensure tag `Owner=x` exists", not "add tag".
- Use APIs that are naturally idempotent (PUT, `put_bucket_tagging`, `kubectl apply`).
- Pass client tokens: many AWS create calls accept `ClientToken` (`run_instances`, `create_snapshot` in some services) so a retried call does not create a second resource.
- Treat "already exists" and "not found on delete" as success.

</details>

### Q107. How do you handle API pagination generically? 🔴

<details>
<summary><b>Answer</b></summary>

APIs paginate by page number, offset, cursor/token, or `Link` headers. Wrap it once in a generator:

```python
def paginate(session, url, params=None):
    params = dict(params or {}, per_page=100)
    while url:
        r = session.get(url, params=params, timeout=10)
        r.raise_for_status()
        yield from r.json()
        url, params = r.links.get("next", {}).get("url"), None   # next URL already has params
```

Cursor APIs (Slack, Stripe, AWS) are safer than offsets when data changes during iteration, because offsets skip or duplicate items when rows are inserted.

</details>

### Q108. How do you respect API rate limits proactively? 🔴

<details>
<summary><b>Answer</b></summary>

- Read rate-limit headers (`X-RateLimit-Remaining`, `X-RateLimit-Reset` on GitHub; `Retry-After` on 429) and sleep until reset when near zero.
- Bound concurrency (a fixed worker pool or `asyncio.Semaphore`), because parallelism multiplies request rate.
- Use a token-bucket limiter for steady pacing (`limits`, `aiolimiter`, or a small class).
- Cache responses (ETags with `If-None-Match`; GitHub does not count 304s against the limit).
- Prefer bulk or GraphQL endpoints to reduce call count.

</details>

### Q109. How do you call an API that uses OAuth2 client credentials or short-lived tokens? 🔴

<details>
<summary><b>Answer</b></summary>

Fetch a token from the token endpoint, cache it with its expiry, and refresh shortly before it expires (or on a 401), instead of fetching per call.

```python
class TokenAuth(requests.auth.AuthBase):
    def __init__(self): self.token, self.exp = None, 0
    def __call__(self, r):
        if time.time() > self.exp - 60:
            t = requests.post(TOKEN_URL, data={"grant_type": "client_credentials"},
                              auth=(CLIENT_ID, CLIENT_SECRET), timeout=10).json()
            self.token, self.exp = t["access_token"], time.time() + t["expires_in"]
        r.headers["Authorization"] = f"Bearer {self.token}"
        return r
session.auth = TokenAuth()
```

</details>

### Q110. What does `verify=False` do, and what should you do instead? 🟡

<details>
<summary><b>Answer</b></summary>

It disables TLS certificate verification, so any man-in-the-middle can impersonate the server and read your credentials. It also spams `InsecureRequestWarning`. The real fix for internal or proxied endpoints is to trust the right CA: `verify="/etc/ssl/certs/corp-ca.pem"` or `REQUESTS_CA_BUNDLE=/path/ca.pem` (requests uses `certifi`'s bundle, not the OS store, unless told otherwise; the `truststore` package can make it use the OS store).

</details>

### Q111. How do you send JSON, form data and files with `requests`? 🟢

<details>
<summary><b>Answer</b></summary>

```python
s.post(url, json={"name": "api", "replicas": 3}, timeout=10)      # sets Content-Type: application/json
s.post(url, data={"user": "ci", "token": t}, timeout=10)           # form-encoded
with open("report.html", "rb") as f:
    s.post(url, files={"file": ("report.html", f, "text/html")}, timeout=30)
```

A common mistake is `data=json.dumps(payload)` without the header, which the server receives as `text/plain` or form data and rejects with 415 or 400.

</details>

### Q112. How do you download a large file without loading it into memory? 🟡

<details>
<summary><b>Answer</b></summary>

Stream it in chunks and verify a checksum:

```python
import hashlib
h = hashlib.sha256()
with s.get(url, stream=True, timeout=(5, 60)) as r:
    r.raise_for_status()
    with open(dest, "wb") as f:
        for chunk in r.iter_content(chunk_size=1 << 20):
            f.write(chunk)
            h.update(chunk)
if h.hexdigest() != expected_sha256:
    raise ValueError("checksum mismatch")
```

Always verify checksums for binaries you install (kubectl, terraform, helm).

</details>

### Q113. `requests` vs `httpx` vs `aiohttp`: when would you pick each? 🟡

<details>
<summary><b>Answer</b></summary>

- **requests**: synchronous, ubiquitous, the default for scripts.
- **httpx**: requests-like API with both sync and async clients, HTTP/2 support, strict timeouts by default (5s), and `respx` for mocking. A good choice if you might go async later.
- **aiohttp**: async-only client and server, very mature, high throughput.

For a script calling 20 endpoints, requests. For 5,000 concurrent health checks, httpx async or aiohttp with a semaphore.

</details>

### Q114. How do you validate untrusted JSON or config input? 🔴

<details>
<summary><b>Answer</b></summary>

Do not trust shape or types from webhooks, config files or APIs. Use `pydantic` models (or `jsonschema` against a schema) to validate and coerce at the boundary, then pass typed objects inward.

```python
from pydantic import BaseModel, Field
class DeployRequest(BaseModel):
    service: str = Field(pattern=r"^[a-z0-9-]{1,40}$")
    version: str
    replicas: int = Field(ge=1, le=50)

req = DeployRequest.model_validate(payload)   # raises ValidationError with field-level errors
```

FastAPI does this automatically for request bodies.

</details>

---

## boto3 & AWS automation

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q115. What is the difference between a boto3 Session, client and resource? 🟢

<details>
<summary><b>Answer</b></summary>

- **Session**: holds configuration, credentials and the default region. `boto3.client(...)` uses a hidden default session.
- **Client**: low-level, one method per API operation, returns plain dicts, covers every service and every operation. Thread-safe.
- **Resource**: higher-level object model (`s3.Bucket("x").objects.all()`), available for a handful of services. AWS has stopped adding features to resources, so new code should use clients.

```python
session = boto3.Session(profile_name="prod", region_name="ap-south-1")
ec2 = session.client("ec2")
```

</details>

### Q116. How does boto3 find credentials? 🟢

<details>
<summary><b>Answer</b></summary>

The credential provider chain, in order (simplified):

1. Explicit parameters to `Session`/`client` (avoid).
2. Environment variables `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`.
3. Web identity token (`AWS_WEB_IDENTITY_TOKEN_FILE` + `AWS_ROLE_ARN`), used by IRSA and some CI setups.
4. Shared credentials file `~/.aws/credentials`, then `~/.aws/config` profiles (including SSO and `role_arn`/`source_profile` assume-role).
5. Container credentials (ECS task role, EKS Pod Identity via `AWS_CONTAINER_CREDENTIALS_FULL_URI`).
6. EC2 instance metadata (IMDSv2) instance profile.

`AWS_PROFILE` selects the profile. The first source found wins, which is why a stale exported key overrides your SSO profile.

</details>

### Q117. How does boto3 decide the region, and what is the gotcha with `AWS_REGION`? 🔴

<details>
<summary><b>Answer</b></summary>

Order: explicit `region_name`, then `AWS_DEFAULT_REGION`, then the `region` in the active profile in `~/.aws/config`. boto3 does not read `AWS_REGION` (that is the variable the Lambda runtime and several other AWS SDKs use), so scripts that set only `AWS_REGION` failed with `botocore.exceptions.NoRegionError: You must specify a region.` In CI, `aws-actions/configure-aws-credentials` sets both. Our habit: pass `region_name` explicitly for multi-region code, and set `AWS_DEFAULT_REGION` for single-region scripts.

</details>

### Q118. Why do you need paginators, and how do you use them? 🟢

<details>
<summary><b>Answer</b></summary>

List and describe APIs return one page at a time (often 50 to 1000 items) plus a `NextToken`/`Marker`. Code that reads only the first response silently misses resources.

```python
paginator = ec2.get_paginator("describe_instances")
for page in paginator.paginate(Filters=[{"Name": "instance-state-name", "Values": ["running"]}]):
    for res in page["Reservations"]:
        for inst in res["Instances"]:
            ...
```

`client.can_paginate("list_objects_v2")` tells you if a paginator exists. `PaginationConfig={"PageSize": 100, "MaxItems": 500}` controls sizing.

</details>

### Q119. What is JMESPath `search()` on paginators? 🔴

<details>
<summary><b>Answer</b></summary>

Page iterators support `.search(expression)`, which applies a JMESPath query across all pages and yields matching items, the same query language as `aws --query`.

```python
pages = ec2.get_paginator("describe_instances").paginate()
for iid in pages.search("Reservations[].Instances[?State.Name=='running'][].InstanceId"):
    print(iid)
```

It is concise for extraction, but plain Python is easier to debug once the logic has conditions on tags.

</details>

### Q120. How do you handle errors from boto3? 🟢

<details>
<summary><b>Answer</b></summary>

Service errors raise `botocore.exceptions.ClientError`; inspect the code and HTTP status.

```python
from botocore.exceptions import ClientError
try:
    s3.head_bucket(Bucket=name)
except ClientError as e:
    code = e.response["Error"]["Code"]
    if code in ("404", "NoSuchBucket"):
        create_bucket(name)
    elif code in ("403", "AccessDenied"):
        raise PermissionError(f"no access to {name}") from e
    else:
        raise
```

Modelled errors are also available as `s3.exceptions.NoSuchKey` etc. Client-side failures (no credentials, no region, connection errors) are subclasses of `BotoCoreError`, not `ClientError`.

</details>

### Q121. What are waiters, and why use them instead of `sleep` loops? 🟡

<details>
<summary><b>Answer</b></summary>

Waiters poll a describe call until a resource reaches a state, with a built-in delay and max attempts, and raise `WaiterError` on failure states or timeout.

```python
ec2.start_instances(InstanceIds=[iid])
ec2.get_waiter("instance_running").wait(InstanceIds=[iid],
                                         WaiterConfig={"Delay": 10, "MaxAttempts": 30})
```

Other common ones: `instance_stopped`, `snapshot_completed`, `db_instance_available`, `stack_create_complete`, `object_exists`. They encode failure states (for example a stack that rolled back) that a homemade loop often forgets.

</details>

### Q122. How do botocore retry modes work, and how do you configure them? 🔴

<details>
<summary><b>Answer</b></summary>

botocore retries transient errors (throttling, 5xx, connection errors) automatically.

- `legacy`: the long-standing boto3 default if you configure nothing; 5 total attempts for most services.
- `standard`: 3 attempts by default, consistent jittered exponential backoff, broader set of retryable errors. The mode AWS recommends.
- `adaptive`: standard plus client-side rate limiting that slows down when throttled; good for bulk scanners.

```python
from botocore.config import Config
cfg = Config(retries={"max_attempts": 10, "mode": "adaptive"},
             connect_timeout=5, read_timeout=30, max_pool_connections=50)
ec2 = boto3.client("ec2", config=cfg)
```

Can also be set via `AWS_RETRY_MODE` and `AWS_MAX_ATTEMPTS`.

</details>

### Q123. How do you assume a role into another account with boto3? 🔴

<details>
<summary><b>Answer</b></summary>

```python
sts = boto3.client("sts")
creds = sts.assume_role(RoleArn="arn:aws:iam::222222222222:role/audit-readonly",
                        RoleSessionName="cloudaudit", DurationSeconds=3600)["Credentials"]
session = boto3.Session(aws_access_key_id=creds["AccessKeyId"],
                        aws_secret_access_key=creds["SecretAccessKey"],
                        aws_session_token=creds["SessionToken"])
```

For long-running jobs, use a profile with `role_arn` and `source_profile` (or `credential_source = Ec2InstanceMetadata`) so botocore refreshes credentials automatically, instead of hand-built static sessions that expire mid-run with `ExpiredToken`.

</details>

### Q124. How do you iterate over all regions? 🟡

<details>
<summary><b>Answer</b></summary>

```python
regions = [r["RegionName"] for r in
           boto3.client("ec2", region_name="us-east-1").describe_regions(
               Filters=[{"Name": "opt-in-status", "Values": ["opt-in-not-required", "opted-in"]}]
           )["Regions"]]
```

`describe_regions` returns only enabled regions by default (add `AllRegions=True` to see disabled ones). `boto3.Session().get_available_regions("ec2")` comes from the SDK's local data and may include regions not enabled for your account, which then fail with `AuthFailure` or `UnrecognizedClientException`.

</details>

### Q125. How do you work with tags in boto3? 🟡

<details>
<summary><b>Answer</b></summary>

Most services represent tags as a list of `{"Key": ..., "Value": ...}` dicts; convert to a dict for lookups.

```python
tags = {t["Key"]: t["Value"] for t in inst.get("Tags", [])}
ec2.create_tags(Resources=[iid], Tags=[{"Key": "Owner", "Value": "platform"}])
ec2.describe_instances(Filters=[{"Name": "tag:Environment", "Values": ["prod"]},
                                {"Name": "tag-key", "Values": ["Owner"]}])
```

Some services differ (S3 uses `TagSet`, Lambda uses a plain dict). The Resource Groups Tagging API (`resourcegroupstaggingapi.get_resources`) searches tags across services in one place.

</details>

### Q126. How do you upload and download S3 objects efficiently? 🟡

<details>
<summary><b>Answer</b></summary>

Use the managed transfer methods, which do multipart uploads and parallel parts automatically:

```python
from boto3.s3.transfer import TransferConfig
cfg = TransferConfig(multipart_threshold=64 * 2**20, max_concurrency=10)
s3.upload_file("backup.tar.gz", "acme-backups", "db/2026-11-09.tar.gz", Config=cfg,
               ExtraArgs={"ServerSideEncryption": "aws:kms", "StorageClass": "STANDARD_IA"})
s3.download_file("acme-backups", key, "/tmp/restore.tar.gz")
```

`put_object` is a single request limited to 5 GB and has no parallelism. Use `list_objects_v2` paginators, not `list_objects`.

</details>

### Q127. How do you generate a presigned S3 URL, and what limits apply? 🟡

<details>
<summary><b>Answer</b></summary>

```python
url = s3.generate_presigned_url("get_object",
                                Params={"Bucket": "acme-reports", "Key": "audit/latest.html"},
                                ExpiresIn=3600)
```

It is generated locally (no API call) using your current credentials, and it stops working when those credentials expire, so a URL signed with a 1-hour STS session never lasts longer than that session regardless of `ExpiresIn`. SigV4 presigned URLs max out at 7 days. Pass `region_name` and, for newer regions, ensure the client uses the regional endpoint, or you get `SignatureDoesNotMatch` or redirects.

</details>

### Q128. How do you read secrets from AWS Secrets Manager or SSM Parameter Store? 🟡

<details>
<summary><b>Answer</b></summary>

```python
sm = boto3.client("secretsmanager")
secret = json.loads(sm.get_secret_value(SecretId="prod/db")["SecretString"])

ssm = boto3.client("ssm")
token = ssm.get_parameter(Name="/ops/slack/token", WithDecryption=True)["Parameter"]["Value"]
```

Cache secrets in memory for the process lifetime (or with a TTL) rather than fetching per request; both APIs are priced and throttled. AWS provides `aws-secretsmanager-caching` for this. Never write secrets to disk or logs.

</details>

### Q129. How would you find the latest AMI programmatically? 🟡

<details>
<summary><b>Answer</b></summary>

The cleanest way is the public SSM parameter:

```python
ami = ssm.get_parameter(
    Name="/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64")["Parameter"]["Value"]
```

For your own golden AMIs, filter and sort:

```python
imgs = ec2.describe_images(Owners=["self"], Filters=[{"Name": "tag:Role", "Values": ["web"]}])["Images"]
latest = max(imgs, key=lambda i: i["CreationDate"])["ImageId"]
```

`CreationDate` is an ISO string, so lexical max works.

</details>

### Q130. What is CloudWatch `get_metric_statistics` vs `get_metric_data`? 🔴

<details>
<summary><b>Answer</b></summary>

- `get_metric_statistics`: one metric per call, simple, returns `Datapoints` unordered, limited to 1,440 datapoints per call.
- `get_metric_data`: up to 500 metric queries per call, supports metric math and Metrics Insights, returns ordered values with pagination. Far cheaper and faster for scanning a fleet.

For an "idle instance" check across 2,000 instances, batching CPU queries in `get_metric_data` turns 2,000 calls into 4. Remember that `CPUUtilization` exists only for running instances, and the default period is 5 minutes (1 minute with detailed monitoring).

</details>

### Q131. Why should automation never use long-lived access keys in CI? 🟡

<details>
<summary><b>Answer</b></summary>

Static keys leak through logs, forks, laptops and old pipelines, never expire, and are hard to attribute. Instead:

- GitHub Actions/GitLab: OIDC federation to an IAM role (`aws-actions/configure-aws-credentials` with `role-to-assume`), with the trust policy restricted to a repo and branch.
- EC2: instance profiles. EKS: IRSA or Pod Identity. ECS: task roles. Lambda: execution role.
- Humans: IAM Identity Center (SSO) with `aws sso login`.

boto3 picks all of these up through the default chain without code changes.

</details>

### Q132. What does `DryRun=True` do on EC2 API calls? 🟡

<details>
<summary><b>Answer</b></summary>

EC2 checks whether you have permission to perform the action, without performing it. Success is reported as an error with code `DryRunOperation`; lack of permission as `UnauthorizedOperation`.

```python
try:
    ec2.stop_instances(InstanceIds=[iid], DryRun=True)
except ClientError as e:
    if e.response["Error"]["Code"] != "DryRunOperation":
        raise
```

It is a great preflight for destructive scripts. Not every service supports it.

</details>

### Q133. How do you type-check boto3 code? 🔴

<details>
<summary><b>Answer</b></summary>

boto3 builds clients dynamically, so out of the box everything is `Any`. Install `boto3-stubs[ec2,s3,...]` (mypy-boto3), and annotate:

```python
from mypy_boto3_ec2 import EC2Client
from mypy_boto3_ec2.type_defs import InstanceTypeDef

def running(ec2: EC2Client) -> list[InstanceTypeDef]: ...
```

Now mypy and the IDE know every method, parameter and response key, catching typos like `ec2.describe_instance()` or `inst["InstanceID"]` before runtime. Keep the stubs in dev dependencies only.

</details>

### Q134. How would you speed up a boto3 script that audits 30 accounts and 17 regions? 🔴

<details>
<summary><b>Answer</b></summary>

- Parallelize across account/region pairs with `ThreadPoolExecutor`; boto3 calls are I/O bound.
- Create one Session per thread (Sessions are not thread-safe; clients are), and one client per service and region, reused across calls.
- Raise `max_pool_connections` in `Config` to match workers.
- Use `adaptive` retry mode so throttling slows you down instead of failing.
- Filter server-side (`Filters=`) rather than downloading everything.
- Use batch APIs (`get_metric_data`, Resource Groups Tagging API), or AWS Config advanced queries / Resource Explorer for inventory instead of describe calls at all.

</details>

---

## Kubernetes, Docker & SSH clients

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q135. How does the official Kubernetes Python client authenticate? 🟢

<details>
<summary><b>Answer</b></summary>

```python
from kubernetes import client, config
try:
    config.load_incluster_config()     # inside a pod: service account token + CA
except config.ConfigException:
    config.load_kube_config()          # local: ~/.kube/config, current context
v1 = client.CoreV1Api()
```

In-cluster it reads `/var/run/secrets/kubernetes.io/serviceaccount/` and the `KUBERNETES_SERVICE_HOST` env var. Locally it honors `KUBECONFIG` and exec plugins (for EKS, the `aws eks get-token` exec entry in kubeconfig).

</details>

### Q136. How do you list pods with high restart counts using the Python client? 🟢

<details>
<summary><b>Answer</b></summary>

```python
v1 = client.CoreV1Api()
for pod in v1.list_pod_for_all_namespaces(watch=False, limit=500).items:
    for cs in pod.status.container_statuses or []:
        if cs.restart_count > 5:
            reason = cs.last_state.terminated.reason if cs.last_state.terminated else "-"
            print(pod.metadata.namespace, pod.metadata.name, cs.name, cs.restart_count, reason)
```

Note `container_statuses` can be `None` for pending pods. For large clusters, paginate with `limit` and `_continue=resp.metadata._continue`, and use `label_selector`/`field_selector` to filter on the server.

</details>

### Q137. How do you patch or scale a Deployment with the Python client? 🟡

<details>
<summary><b>Answer</b></summary>

```python
apps = client.AppsV1Api()
apps.patch_namespaced_deployment_scale("api", "prod", {"spec": {"replicas": 5}})

# trigger a rolling restart, same as kubectl rollout restart
now = datetime.now(UTC).isoformat()
apps.patch_namespaced_deployment("api", "prod", {"spec": {"template": {"metadata": {
    "annotations": {"kubectl.kubernetes.io/restartedAt": now}}}}})
```

A dict body is sent as a strategic merge patch by default. Errors raise `kubernetes.client.exceptions.ApiException` with `.status` (404, 409, 422) and `.body`.

</details>

### Q138. How does `watch.Watch()` work, and what is a `410 Gone`? 🔴

<details>
<summary><b>Answer</b></summary>

A watch streams ADDED/MODIFIED/DELETED events from a `resourceVersion` onward.

```python
from kubernetes import watch
w = watch.Watch()
rv = v1.list_namespaced_pod("prod").metadata.resource_version
for ev in w.stream(v1.list_namespaced_pod, "prod", resource_version=rv, timeout_seconds=300):
    print(ev["type"], ev["object"].metadata.name)
```

The API server only keeps a window of history. If you reconnect with an old `resourceVersion`, you get `410 Gone` ("too old resource version") and must re-list and start again from the new version. Long-running watchers must handle that, plus timeouts and reconnects. For serious controllers use `kopf` or Go's controller-runtime, which handle informers and caching.

</details>

### Q139. How do you apply a YAML manifest from Python? 🔴

<details>
<summary><b>Answer</b></summary>

`kubernetes.utils.create_from_yaml(api_client, "app.yaml")` creates objects but does not update existing ones (409 Conflict). For apply semantics, use server-side apply through the dynamic client:

```python
from kubernetes import dynamic
dyn = dynamic.DynamicClient(client.ApiClient())
for doc in yaml.safe_load_all(open("app.yaml")):
    api = dyn.resources.get(api_version=doc["apiVersion"], kind=doc["kind"])
    api.server_side_apply(body=doc, namespace=doc["metadata"].get("namespace", "default"),
                          field_manager="ops-bot", force_conflicts=True)
```

Many teams simply call `kubectl apply` via `subprocess` for this, which is acceptable and easier to reason about.

</details>

### Q140. How do you use the Docker SDK for Python? 🟢

<details>
<summary><b>Answer</b></summary>

```python
import docker
dc = docker.from_env()                 # honours DOCKER_HOST, defaults to the Unix socket
dc.images.pull("nginx", tag="1.27-alpine")
c = dc.containers.run("nginx:1.27-alpine", detach=True, name="web",
                      ports={"80/tcp": 8080}, mem_limit="256m")
print(c.status, c.logs(tail=20).decode())
for ct in dc.containers.list(filters={"status": "exited"}):
    ct.remove()
```

Remember that access to the Docker socket is equivalent to root on the host, so a tool using the SDK must be treated as privileged.

</details>

### Q141. How do you clean up dangling Docker images and stopped containers with Python? 🟡

<details>
<summary><b>Answer</b></summary>

```python
dc = docker.from_env()
dc.containers.prune(filters={"until": "24h"})
freed = dc.images.prune(filters={"dangling": True})["SpaceReclaimed"]
dc.volumes.prune()          # careful: removes unused volumes, which may hold data
log.info("reclaimed %.1f MB", freed / 1e6)
```

This maps to the Engine API prune endpoints, the same thing `docker system prune` uses. We run this on CI runners on a schedule; never on hosts where unused volumes may still hold data.

</details>

### Q142. What is paramiko, and how do you run a command over SSH with it? 🟢

<details>
<summary><b>Answer</b></summary>

paramiko is a pure-Python SSHv2 implementation.

```python
import paramiko
ssh = paramiko.SSHClient()
ssh.load_system_host_keys()
ssh.set_missing_host_key_policy(paramiko.RejectPolicy())
ssh.connect("10.0.1.15", username="ec2-user", key_filename="~/.ssh/ops.pem", timeout=10)
stdin, stdout, stderr = ssh.exec_command("uptime")
rc = stdout.channel.recv_exit_status()
print(rc, stdout.read().decode())
ssh.close()
```

`AutoAddPolicy` is convenient but trusts any host key on first connect (MITM risk); in production pre-populate known_hosts and reject unknown keys.

</details>

### Q143. What does Fabric add on top of paramiko? 🟡

<details>
<summary><b>Answer</b></summary>

Fabric (2.x/3.x) gives a high-level `Connection` with `run`, `sudo`, `put`, `get`, `cd` context managers, and `ThreadingGroup` for parallel execution across hosts.

```python
from fabric import Connection, ThreadingGroup
c = Connection("web1", user="deploy", connect_kwargs={"key_filename": "~/.ssh/deploy"})
c.put("app.tar.gz", "/tmp/")
c.sudo("systemctl restart app", hide=True)

results = ThreadingGroup("web1", "web2", "web3").run("df -h /", hide=True, warn=True)
for conn, r in results.items():
    print(conn.host, r.ok, r.stdout.strip())
```

`warn=True` returns failures instead of raising, so one bad host does not abort the whole group.

</details>

### Q144. When would you write Python with paramiko/fabric instead of using Ansible? 🟡

<details>
<summary><b>Answer</b></summary>

Use Ansible for configuration management across a fleet: idempotent modules, inventory, roles, facts, check mode, and a shared language your team already reads. Use Python SSH libraries for targeted tooling where you need logic Ansible makes awkward: a custom health probe that parses output and decides the next step, a bastion-hopping diagnostic CLI, or embedding SSH inside a larger Python service. Avoid reimplementing package/service/file management in paramiko; that is reinventing Ansible badly.

</details>

### Q145. How do you SSH through a bastion host from Python? 🔴

<details>
<summary><b>Answer</b></summary>

With Fabric, pass a gateway `Connection`:

```python
bastion = Connection("bastion.acme.com", user="ops")
c = Connection("10.0.2.20", user="ec2-user", gateway=bastion)
c.run("hostname")
```

With paramiko, open a `direct-tcpip` channel on the bastion's transport and pass it as `sock=` when connecting to the target. On AWS, many teams avoid bastions entirely with SSM Session Manager (`aws ssm start-session`, or `ssm.send_command` from boto3 for remote command execution without any inbound SSH).

</details>

### Q146. How do you run a command on many EC2 instances without SSH? 🟡

<details>
<summary><b>Answer</b></summary>

Use SSM Run Command via boto3:

```python
ssm = boto3.client("ssm")
cmd = ssm.send_command(Targets=[{"Key": "tag:Role", "Values": ["web"]}],
                       DocumentName="AWS-RunShellScript",
                       Parameters={"commands": ["systemctl is-active nginx"]},
                       MaxConcurrency="10%", MaxErrors="1")
cid = cmd["Command"]["CommandId"]
# later: ssm.list_command_invocations(CommandId=cid, Details=True)
```

No open port 22, no keys, IAM-controlled and logged in CloudTrail; output can go to S3 or CloudWatch Logs. Instances need the SSM agent and an instance profile with `AmazonSSMManagedInstanceCore`.

</details>

---

## Concurrency: threads, processes & asyncio

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q147. What is the GIL, and what does it mean for DevOps scripts? 🟡

<details>
<summary><b>Answer</b></summary>

The Global Interpreter Lock in CPython allows only one thread to execute Python bytecode at a time. It is released during blocking I/O (network, disk, `time.sleep`, many C extensions), so threads give real speedups for I/O-bound work like API calls and SSH, and none for CPU-bound pure-Python work like hashing or parsing in a loop. For CPU-bound work use processes. Python 3.13 introduced an experimental free-threaded build (`python3.13t`) without the GIL, and 3.14 made it officially supported but optional; the default build still has the GIL.

</details>

### Q148. How do you use `ThreadPoolExecutor` for parallel API calls? 🟢

<details>
<summary><b>Answer</b></summary>

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

def check(url):
    r = session.get(url, timeout=5)
    return url, r.status_code

with ThreadPoolExecutor(max_workers=20) as pool:
    futures = {pool.submit(check, u): u for u in urls}
    for fut in as_completed(futures):
        try:
            url, code = fut.result()
        except requests.RequestException as e:
            log.warning("%s failed: %s", futures[fut], e)
```

`as_completed` yields results as they finish; `pool.map` preserves input order but raises the first exception when you reach it. Exceptions inside a worker are only re-raised when you call `.result()`, so always call it.

</details>

### Q149. Threads vs processes vs asyncio: how do you choose? 🟡

<details>
<summary><b>Answer</b></summary>

- **Threads** (`ThreadPoolExecutor`): I/O bound, sync libraries (boto3, requests, paramiko), tens to a few hundred concurrent tasks.
- **Processes** (`ProcessPoolExecutor`, `multiprocessing`): CPU bound; each process has its own interpreter and GIL. Higher memory and startup cost; arguments and results must be picklable.
- **asyncio**: thousands of concurrent I/O tasks with async libraries (httpx, aiohttp, asyncssh), single thread, cooperative scheduling. Every library in the path must be async.

Our default for ops tooling is threads, because boto3 is synchronous.

</details>

### Q150. What is a race condition, and how do you prevent one in threaded code? 🟡

<details>
<summary><b>Answer</b></summary>

A race happens when threads read-modify-write shared state without coordination, for example `counter += 1` (load, add, store) from 20 threads losing increments.

- Avoid shared mutable state: have workers return results and aggregate in the main thread.
- If you must share, protect with `threading.Lock()` (`with lock: counter += 1`).
- Use thread-safe structures: `queue.Queue` for producer/consumer.

The GIL does not make your code thread-safe; it only protects the interpreter's internals.

</details>

### Q151. How does a producer/consumer pattern with `queue.Queue` work? 🔴

<details>
<summary><b>Answer</b></summary>

```python
import queue, threading
q: queue.Queue[str | None] = queue.Queue(maxsize=100)

def worker():
    while (item := q.get()) is not None:
        try:
            process(item)
        finally:
            q.task_done()
    q.task_done()

threads = [threading.Thread(target=worker, daemon=True) for _ in range(8)]
for t in threads: t.start()
for key in s3_keys(): q.put(key)        # blocks when full: back-pressure
for _ in threads: q.put(None)           # one sentinel per worker
q.join()
```

`maxsize` gives back-pressure so a fast producer does not fill memory.

</details>

### Q152. What are coroutines, `async def` and `await`? 🟡

<details>
<summary><b>Answer</b></summary>

`async def` defines a coroutine function; calling it returns a coroutine object that does nothing until awaited or scheduled. `await` suspends the coroutine until the awaited operation completes, letting the event loop run other coroutines meanwhile. `asyncio.run(main())` creates the loop, runs `main` to completion and closes the loop.

```python
import asyncio, httpx
async def main():
    async with httpx.AsyncClient(timeout=5) as c:
        r = await c.get("https://example.com/healthz")
        print(r.status_code)
asyncio.run(main())
```

Forgetting `await` gives `RuntimeWarning: coroutine '...' was never awaited` and nothing happens.

</details>

### Q153. How do you run many async tasks concurrently with a limit? 🟡

<details>
<summary><b>Answer</b></summary>

```python
async def probe(client, sem, url):
    async with sem:
        try:
            r = await client.get(url)
            return url, r.status_code
        except httpx.HTTPError as e:
            return url, repr(e)

async def main(urls):
    sem = asyncio.Semaphore(50)
    async with httpx.AsyncClient(timeout=5) as client:
        return await asyncio.gather(*(probe(client, sem, u) for u in urls))
```

Without the semaphore, 10,000 URLs means 10,000 simultaneous sockets, which hits file descriptor limits and rate limits.

</details>

### Q154. What is the difference between `asyncio.gather` and `TaskGroup`? 🔴

<details>
<summary><b>Answer</b></summary>

- `gather(*aws)` runs awaitables concurrently and returns results in order. By default the first exception propagates but the other tasks keep running unattended; `return_exceptions=True` returns exceptions as values.
- `asyncio.TaskGroup` (3.11+) is structured concurrency: if any task fails, the others are cancelled, and errors are raised together as an `ExceptionGroup` (handled with `except*`).

```python
async with asyncio.TaskGroup() as tg:
    t1 = tg.create_task(fetch("a"))
    t2 = tg.create_task(fetch("b"))
print(t1.result(), t2.result())
```

Use TaskGroup when partial results are useless; gather with `return_exceptions=True` for "probe everything, report failures".

</details>

### Q155. What happens if you call a blocking function inside a coroutine? 🔴

<details>
<summary><b>Answer</b></summary>

It blocks the entire event loop: every other coroutine freezes until it returns. `time.sleep(5)`, `requests.get`, boto3 calls and heavy CPU work all do this. Fixes: use the async equivalent (`await asyncio.sleep`, httpx/aiohttp, aioboto3), or offload with `await asyncio.to_thread(blocking_fn, arg)` (or `loop.run_in_executor`). Enable debug mode (`PYTHONASYNCIODEBUG=1` or `asyncio.run(main(), debug=True)`) to log callbacks that take longer than 100 ms.

</details>

### Q156. How do you add timeouts to async operations? 🟡

<details>
<summary><b>Answer</b></summary>

```python
try:
    async with asyncio.timeout(10):            # 3.11+
        data = await fetch_all()
except TimeoutError:
    log.error("fetch_all exceeded 10s")

result = await asyncio.wait_for(fetch_one(), timeout=3)   # older API
```

A timeout cancels the inner task by raising `CancelledError` inside it, so cleanup code in `finally` blocks runs. Also set client-level timeouts (httpx/aiohttp) so individual requests cannot hang.

</details>

### Q157. How do `multiprocessing` and `ProcessPoolExecutor` differ from threads in practice? 🔴

<details>
<summary><b>Answer</b></summary>

Each worker is a separate OS process with its own memory and GIL, so CPU-bound work scales across cores. Costs: start-up time, memory per process, and every argument and result is pickled and sent over a pipe (lambdas, open sockets and boto3 clients cannot be pickled). Start methods: `fork` (Linux default before 3.14, fast but unsafe with threads), `spawn` (macOS/Windows default, re-imports your module, which is why the `__main__` guard is required), `forkserver` (Linux default from 3.14).

</details>

### Q158. When is `subprocess` itself a form of parallelism? 🔴

<details>
<summary><b>Answer</b></summary>

External commands run as separate processes, so you can start several with `Popen` and wait for all, or run `subprocess.run` calls from a thread pool. The GIL is irrelevant because the work happens in the child processes.

```python
with ThreadPoolExecutor(max_workers=4) as pool:
    results = list(pool.map(lambda d: subprocess.run(["terraform", "-chdir=" + d, "plan", "-no-color"],
                                                      capture_output=True, text=True), dirs))
```

Bound the parallelism: four `terraform plan` processes each hitting provider APIs can trigger throttling or exhaust memory on a CI runner.

</details>

### Q159. What is a thread-safe way to share a boto3 client or Session? 🔴

<details>
<summary><b>Answer</b></summary>

boto3 low-level clients are documented as generally thread-safe, so one client can be shared across threads. Session objects and resources are not thread-safe. The safe pattern is: create the Session per thread (or create all clients up front in the main thread) and never create clients concurrently from a shared default session, which can raise intermittent errors like `KeyError: 'endpoint_resolver'` or `credential_provider` races. In our lab, each region worker creates its own `boto3.session.Session()`.

</details>

### Q160. How do you stop a pool of workers cleanly on Ctrl-C or SIGTERM? 🔴

<details>
<summary><b>Answer</b></summary>

Use a shared `threading.Event` that workers check between units of work, and on interrupt set it and cancel pending futures:

```python
stop = threading.Event()
with ThreadPoolExecutor(max_workers=8) as pool:
    futures = [pool.submit(worker, item, stop) for item in items]
    try:
        for f in as_completed(futures):
            f.result()
    except KeyboardInterrupt:
        stop.set()
        pool.shutdown(wait=True, cancel_futures=True)   # 3.9+
        raise
```

Threads cannot be killed from outside, so long-running workers must cooperate by checking the event. Daemon threads die abruptly at exit, skipping cleanup.

</details>

---

## Testing, services & metrics

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q161. Why should automation scripts have tests at all? 🟢

<details>
<summary><b>Answer</b></summary>

Because infrastructure scripts are the code most likely to delete something important. Tests catch the cheap bugs (typo in a dict key, wrong filter, off-by-one on a date cutoff, missing pagination) before they run against production, let you refactor safely, and document the intended behaviour. With moto, tests for AWS logic run in seconds, offline, for free, so there is no excuse to test destructive scripts "live in dev and hope".

</details>

### Q162. How does pytest discover and run tests? 🟢

<details>
<summary><b>Answer</b></summary>

By default pytest collects files named `test_*.py` or `*_test.py`, functions named `test_*`, and methods named `test_*` in classes named `Test*` (without `__init__`). Configuration lives in `pyproject.toml` under `[tool.pytest.ini_options]` (testpaths, addopts, markers).

```bash
pytest -q                      # quiet
pytest tests/test_checks.py::test_open_ssh_is_high -vv
pytest -k "snapshot and not slow" -x --lf    # filter, stop at first failure, last-failed
pytest --cov=cloudaudit --cov-report=term-missing
```

</details>

### Q163. Why does pytest use plain `assert` instead of `assertEqual`? 🟢

<details>
<summary><b>Answer</b></summary>

pytest rewrites `assert` statements at import time to show detailed introspection on failure: both sides of a comparison, the diff of dicts and lists, and intermediate values. So `assert findings == expected` gives a readable diff without needing the `unittest` method zoo. To check exceptions use `with pytest.raises(ConfigError, match="missing region"):`.

</details>

### Q164. What are pytest fixtures, and what scopes exist? 🟡

<details>
<summary><b>Answer</b></summary>

Fixtures provide setup (and teardown after `yield`) to tests that name them as arguments. They live in the test file or a shared `conftest.py`.

```python
@pytest.fixture
def ec2(aws):                       # depends on another fixture
    return boto3.client("ec2", region_name="us-east-1")

@pytest.fixture(scope="session")
def docker_image():
    subprocess.run(["docker", "build", "-t", "app:test", "."], check=True)
    yield "app:test"
    subprocess.run(["docker", "rmi", "app:test"])
```

Scopes: `function` (default), `class`, `module`, `package`, `session`. Wider scopes save time but share state, so keep mutable resources function-scoped.

</details>

### Q165. What built-in fixtures do you use most? 🟡

<details>
<summary><b>Answer</b></summary>

- `tmp_path`: a unique `pathlib.Path` temp directory per test.
- `monkeypatch`: set/delete env vars, attributes and dict items, automatically undone (`monkeypatch.setenv("AWS_DEFAULT_REGION", "us-east-1")`).
- `capsys` / `capfd`: capture stdout/stderr.
- `caplog`: capture log records (`assert "deleted" in caplog.text`).
- `request`: access test metadata and parametrization inside fixtures.

</details>

### Q166. How does `@pytest.mark.parametrize` work? 🟢

<details>
<summary><b>Answer</b></summary>

It runs one test function with multiple input sets, each reported as its own test.

```python
@pytest.mark.parametrize("cidr,port,expected", [
    ("0.0.0.0/0", 22, True),
    ("10.0.0.0/8", 22, False),
    ("0.0.0.0/0", 443, False),
    ("::/0", 3389, True),
], ids=["ssh-world", "ssh-private", "https-world", "rdp-ipv6"])
def test_is_risky(cidr, port, expected):
    assert is_risky(cidr, port) is expected
```

Edge cases become one line each, which is exactly how you should test rule engines like security-group audits.

</details>

### Q167. What is moto, and how do you use it with pytest? 🟡

<details>
<summary><b>Answer</b></summary>

moto mocks AWS services in-process by intercepting botocore's HTTP layer, keeping a fake in-memory backend that behaves like the real API. In moto 5, a single `mock_aws` decorator or context manager covers all services.

```python
from moto import mock_aws

@pytest.fixture
def s3():
    with mock_aws():
        c = boto3.client("s3", region_name="us-east-1")
        c.create_bucket(Bucket="test")
        yield c

def test_upload(s3):
    upload_report(s3, "test", "r.json", b"{}")
    assert s3.get_object(Bucket="test", Key="r.json")["Body"].read() == b"{}"
```

Always set fake credentials in env (`AWS_ACCESS_KEY_ID=testing`) so a bug can never reach a real account.

</details>

### Q168. Where are moto's limits, and what do you do about them? 🔴

<details>
<summary><b>Answer</b></summary>

moto implements most common operations but not every API, field or behaviour: IAM policies are not enforced by default, some timestamps are always "now" (for example snapshot `StartTime`), CloudWatch only returns metrics you put yourself, and eventual consistency is not simulated. Approaches:

- For time-based logic, stub the response with `unittest.mock` or botocore's `Stubber`, or inject a clock.
- Keep business logic in pure functions that take dicts, test those directly, and use moto for the API glue.
- Run a small integration suite against a real sandbox account (or LocalStack) on a schedule.

</details>

### Q169. What is botocore's `Stubber`, and when do you prefer it to moto? 🔴

<details>
<summary><b>Answer</b></summary>

`Stubber` queues exact expected requests and canned responses on a real client, and fails if calls differ.

```python
from botocore.stub import Stubber
ec2 = boto3.client("ec2", region_name="us-east-1")
with Stubber(ec2) as st:
    st.add_response("describe_snapshots", {"Snapshots": [{"SnapshotId": "snap-1",
        "StartTime": datetime(2024, 1, 1, tzinfo=UTC), "VolumeSize": 8}]},
        {"OwnerIds": ["self"]})
    st.add_client_error("delete_snapshot", service_error_code="InvalidSnapshot.InUse")
    ...
```

Prefer it when you need exact control: specific error codes, throttling, odd fields, or asserting the exact parameters sent. moto is better for stateful flows (create then list).

</details>

### Q170. How do you mock things with `unittest.mock`, and what is "patch where it is looked up"? 🔴

<details>
<summary><b>Answer</b></summary>

`patch` replaces a name for the duration of a test. You must patch the name in the module that uses it, not where it was defined.

```python
# app/notify.py:  import requests  ...  requests.post(...)
@patch("app.notify.requests.post")
def test_slack(mock_post):
    mock_post.return_value.status_code = 200
    notify("hi")
    mock_post.assert_called_once()
```

If `app/notify.py` did `from requests import post`, you would patch `app.notify.post`. Use `autospec=True` so the mock enforces the real signature and catches wrong arguments.

</details>

### Q171. How do you test HTTP-calling code without hitting the network? 🟡

<details>
<summary><b>Answer</b></summary>

Use `responses` (for requests) or `respx` (for httpx) to register fake responses:

```python
import responses
@responses.activate
def test_latest_release():
    responses.get("https://api.github.com/repos/acme/app/releases/latest",
                  json={"tag_name": "v1.4.2"}, status=200)
    assert latest_release("acme/app") == "v1.4.2"
```

Unregistered calls raise `ConnectionError`, which also guarantees tests never leak to real services. Also test the failure paths (500, timeout, 429 with Retry-After) since retry logic is where bugs hide.

</details>

### Q172. How do you test a click or argparse CLI? 🟡

<details>
<summary><b>Answer</b></summary>

For click, use `CliRunner`, which invokes the command in-process and captures output and exit code:

```python
from click.testing import CliRunner
def test_cli_fails_on_high(aws):
    r = CliRunner().invoke(main, ["--region", "us-east-1", "--format", "json"])
    assert r.exit_code == 2
    assert '"HIGH"' in r.output
```

For argparse, structure `main(argv=None)` to accept a list and return an exit code, then call `main(["--dry-run"])` directly and use `capsys`. Assert on exit codes explicitly; they are the contract with CI.

</details>

### Q173. What does test coverage tell you, and what target do you set? 🟡

<details>
<summary><b>Answer</b></summary>

Coverage (`pytest --cov=pkg --cov-report=term-missing`) shows which lines and branches ran during tests. It finds untested code; it does not prove the tests assert anything meaningful. We enforce a floor in CI (`--cov-fail-under=80`) and care most that destructive paths, error handling and edge cases are covered. Use `--cov-branch` to catch untested `if` branches.

</details>

### Q174. How do you structure code so it is easy to test? 🔴

<details>
<summary><b>Answer</b></summary>

- Dependency injection: pass clients, clocks and config into functions instead of creating them inside (`def old_snapshots(ec2, now, max_age)`).
- Separate pure logic (decide what is non-compliant) from I/O (fetch and act).
- Keep the CLI layer thin: parse args, build clients, call functions, render, exit.
- Avoid module-level side effects (clients created at import, `basicConfig` in modules).
- Return data instead of printing.

Code written this way can be tested with plain dicts in milliseconds, and moto covers the thin I/O layer.

</details>

### Q175. What do `ruff` and `mypy` do, and why run them in CI? 🟢

<details>
<summary><b>Answer</b></summary>

- `ruff` is a very fast linter and formatter (replacing flake8, isort, pyupgrade, black-style formatting and many pylint rules). It catches unused imports, undefined names, mutable defaults, bare excepts, f-strings in logging and more.
- `mypy` is a static type checker that catches wrong types, `None` handling bugs and misspelled attributes.

```bash
ruff check . && ruff format --check . && mypy src
```

They run in seconds and catch bugs that otherwise surface only at runtime, often in production. We also run them as pre-commit hooks.

</details>

### Q176. What is `pre-commit`, and what hooks are standard for Python repos? 🟡

<details>
<summary><b>Answer</b></summary>

`pre-commit` manages Git hooks defined in `.pre-commit-config.yaml`, running checks on staged files before each commit. Typical hooks: `ruff` (lint + format), `mypy`, `check-yaml`, `end-of-file-fixer`, `detect-private-key`, `gitleaks` for secrets, `bandit` for security issues. Run `pre-commit run --all-files` in CI as well, since local hooks can be skipped with `--no-verify`.

</details>

### Q177. What does Bandit catch? 🟡

<details>
<summary><b>Answer</b></summary>

Bandit is a security-focused static analyser for Python. Typical findings: `subprocess` with `shell=True` (B602), `yaml.load` (B506), `requests` with `verify=False` (B501), hardcoded passwords (B105), `pickle` loads (B301), `assert` used for security checks (B101, since asserts are stripped with `-O`), weak hashes like MD5 for security (B324), binding to `0.0.0.0` (B104). Ruff implements many Bandit rules under the `S` prefix.

</details>

### Q178. How do you test code that depends on the current time? 🔴

<details>
<summary><b>Answer</b></summary>

Do not call `datetime.now()` deep inside logic. Either inject `now` as a parameter (simplest, clearest), or freeze time in tests with `freezegun` or `time-machine`:

```python
import time_machine
@time_machine.travel("2026-11-09 00:00:00+00:00")
def test_expiry():
    assert is_expired({"StartTime": datetime(2026, 7, 1, tzinfo=UTC)}, days=90)
```

Time-based retention rules (snapshot cleanup, certificate expiry, key rotation) are exactly where an off-by-one deletes something it should not.

</details>

### Q179. How do you write a minimal health-check service with Flask? 🟢

<details>
<summary><b>Answer</b></summary>

```python
from flask import Flask, jsonify
app = Flask(__name__)

@app.get("/healthz")
def healthz():
    return jsonify(status="ok")

@app.get("/readyz")
def readyz():
    ok = db_ping()
    return jsonify(db=ok), (200 if ok else 503)
```

Run it with a production server: `gunicorn -w 2 -b 0.0.0.0:8000 app:app`. `flask run` and `app.run()` start the Werkzeug development server, which prints a warning and is not meant for production.

</details>

### Q180. What is the difference between liveness and readiness endpoints? 🟡

<details>
<summary><b>Answer</b></summary>

- **Liveness** (`/healthz`, `/livez`): "is the process alive and not deadlocked?" Must be cheap and must not check dependencies. If it fails, Kubernetes restarts the container.
- **Readiness** (`/readyz`): "can I serve traffic right now?" Checks critical dependencies (DB, cache) with short timeouts. If it fails, the pod is removed from Service endpoints but not restarted.

Checking the database in liveness is a classic mistake: a DB blip restarts every pod at once and turns a partial outage into a full one.

</details>

### Q181. How do you build the same service in FastAPI, and why do teams choose it? 🟡

<details>
<summary><b>Answer</b></summary>

```python
from fastapi import FastAPI, Response
from pydantic import BaseModel
app = FastAPI()

class Deploy(BaseModel):
    service: str
    version: str

@app.get("/healthz")
async def healthz():
    return {"status": "ok"}

@app.post("/deploy", status_code=202)
async def deploy(req: Deploy):
    enqueue(req)
    return {"accepted": req.service}
```

Run with `uvicorn app:app --host 0.0.0.0 --port 8000` (or `fastapi run`). FastAPI validates bodies from type hints via pydantic, returns 422 with field errors automatically, generates OpenAPI docs at `/docs`, and supports async handlers.

</details>

### Q182. How do you verify a GitHub webhook signature? 🔴

<details>
<summary><b>Answer</b></summary>

GitHub sends `X-Hub-Signature-256: sha256=<hex>`, an HMAC-SHA256 of the raw body using the shared secret. Compute it over the raw bytes (not re-serialized JSON) and compare in constant time.

```python
import hashlib, hmac
@app.post("/webhook")
def webhook():
    sig = request.headers.get("X-Hub-Signature-256", "")
    expected = "sha256=" + hmac.new(SECRET.encode(), request.get_data(), hashlib.sha256).hexdigest()
    if not hmac.compare_digest(sig, expected):
        abort(401)
    event = request.headers["X-GitHub-Event"]
    ...
    return "", 204
```

`==` leaks timing information; `compare_digest` does not. Also dedupe on `X-GitHub-Delivery`, since GitHub can redeliver.

</details>

### Q183. Why should webhook handlers respond fast and process asynchronously? 🔴

<details>
<summary><b>Answer</b></summary>

Senders have short timeouts (GitHub waits 10 seconds) and may retry or mark the delivery failed. If the handler runs a deployment inline, the request times out, the sender redelivers, and you deploy twice. Pattern: validate the signature, persist or enqueue the event (SQS, Redis, a DB table), return 2xx immediately, and process in a worker that is idempotent keyed on the delivery ID.

</details>

### Q184. What metric types does `prometheus_client` provide? 🟢

<details>
<summary><b>Answer</b></summary>

- **Counter**: only goes up (resets on restart). Requests served, findings detected. Query with `rate()`.
- **Gauge**: goes up and down. Queue depth, in-flight requests, last successful run timestamp.
- **Histogram**: observations in configurable buckets, plus `_sum` and `_count`. Request latency; aggregatable across instances with `histogram_quantile()`.
- **Summary**: client-side quantiles; cheap but cannot be aggregated across instances.

Prefer histograms for latency in anything that runs as more than one replica.

</details>

### Q185. How do you instrument a Python service with Prometheus metrics? 🟡

<details>
<summary><b>Answer</b></summary>

```python
from prometheus_client import Counter, Histogram, start_http_server

REQS = Counter("webhook_requests_total", "Webhook requests", ["event", "status"])
LAT = Histogram("webhook_duration_seconds", "Handler latency",
                buckets=(0.05, 0.1, 0.25, 0.5, 1, 2.5, 5))

@LAT.time()
def handle(event):
    ...
    REQS.labels(event=event, status="ok").inc()

start_http_server(9100)   # serves /metrics on a separate port
```

With Flask you can mount `prometheus_client.make_wsgi_app()` at `/metrics`; with FastAPI, `make_asgi_app()`. Naming: snake_case, base units (seconds, bytes), `_total` suffix on counters.

</details>

### Q186. What is label cardinality, and why can it take down Prometheus? 🔴

<details>
<summary><b>Answer</b></summary>

Every unique combination of label values is a separate time series. Labels like `user_id`, `request_id`, full URL paths with IDs, or raw error messages create unbounded series, exploding Prometheus memory and making queries slow or the server OOM. Keep labels to small, bounded sets (method, status class, route template like `/users/{id}`, region). Put high-cardinality detail in logs or traces instead.

</details>

### Q187. How do you expose metrics from a batch job or cron script? 🔴

<details>
<summary><b>Answer</b></summary>

A short-lived job is gone before Prometheus scrapes it. Options:

- Push to a Pushgateway at the end of the run with `push_to_gateway("pushgateway:9091", job="cloudaudit", registry=reg)`, recording `last_success_timestamp_seconds`, duration and findings count. Alert when `time() - last_success > 2 * interval`.
- Write a `.prom` file for node_exporter's textfile collector on the host.
- In CI, record results in the workflow summary and send a notification on failure.

Use a dedicated `CollectorRegistry()` for pushes so you do not push default process metrics.

</details>

### Q188. What changes when you run `prometheus_client` under gunicorn with multiple workers? 🔴

<details>
<summary><b>Answer</b></summary>

Each gunicorn worker is a separate process with its own in-memory metrics, so a scrape hits one random worker and you see inconsistent, jumping counters. Use multiprocess mode: set `PROMETHEUS_MULTIPROC_DIR` to an empty directory before workers start, build the `/metrics` response with a `CollectorRegistry` plus `multiprocess.MultiProcessCollector(registry)`, call `multiprocess.mark_process_dead(worker.pid)` in gunicorn's `child_exit` hook, and set `multiprocess_mode` on Gauges (for example `"livesum"` or `"max"`). Clear the directory on each start.

</details>

---

## Best practices & rapid-fire commands

<!-- DEVOPS-SHACK:SECTION -->
> 🎥 **Learn this hands-on with DevOps Shack** → [YouTube @devopsshack](https://www.youtube.com/@devopsshack) · [Hindi @shackverse](https://www.youtube.com/@shackverse) · [Courses at devopsshack.com](https://devopsshack.com)

### Q189. What does a production-quality automation script always include? 🟡

<details>
<summary><b>Answer</b></summary>

- A CLI with `--help`, `--dry-run` for anything destructive, and documented exit codes.
- Logging (levels, stderr, no secrets), not `print`.
- Timeouts on every network call, retries with backoff for transient errors, pagination everywhere.
- Credentials from the environment or role chain; nothing hardcoded.
- Idempotency: safe to re-run after a partial failure.
- Pinned dependencies, a venv or container, tests (pytest + moto/mocks), ruff and mypy in CI.
- A README with the IAM permissions it needs, and an owner.

</details>

### Q190. How do you handle configuration in a Python tool? 🟡

<details>
<summary><b>Answer</b></summary>

Layer it, with clear precedence: built-in defaults, then a config file (YAML/TOML), then environment variables, then CLI flags (highest). Validate the merged result at startup with pydantic or explicit checks and fail with a clear message. Use `tomllib` (standard library since 3.11) to read TOML. Keep secrets out of config files; reference a secret name and resolve it at runtime. Twelve-factor services usually need only env vars (`pydantic-settings` reads them into a typed model).

</details>

### Q191. What are the most common security mistakes in Python ops code? 🔴

<details>
<summary><b>Answer</b></summary>

- `shell=True` with interpolated input (command injection).
- `yaml.load`, `pickle.loads` on untrusted data (code execution).
- `verify=False` on HTTPS calls.
- Hardcoded keys and tokens, or secrets in logs and exception messages.
- `paramiko.AutoAddPolicy()` in production (MITM).
- `eval()`/`exec()` on input.
- Path traversal: joining user-supplied names into paths without checking `resolved.is_relative_to(base)`.
- Unpinned dependencies from public indexes (typosquatting, dependency confusion).
- Overly broad IAM for the script's role.

</details>

### Q192. Bash or Python for this task: how do you decide? 🟢

<details>
<summary><b>Answer</b></summary>

Bash when the job is short, linear, and mostly composes existing commands (`tar`, `rsync`, `systemctl`), with little data handling. Python when you need data structures, JSON/YAML parsing, API calls, error handling beyond `set -e`, retries, concurrency, tests, or the script is passing 50 lines. A good rule: once you reach for `jq` pipelines inside loops with arrays, switch to Python.

</details>

### Q193. Rapid fire: set up a project environment from scratch. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
python3 -m venv .venv && source .venv/bin/activate
python -m pip install --upgrade pip
pip install -e ".[dev]"
pip install -r requirements.txt
pip list --outdated
pip show boto3
python -c "import sys; print(sys.version, sys.executable)"
uv venv && uv pip install -r requirements.txt     # uv equivalent
```

</details>

### Q194. Rapid fire: one-liners useful during an incident. 🟡

<details>
<summary><b>Answer</b></summary>

```bash
python3 -m http.server 8000 --bind 127.0.0.1            # serve a directory
python3 -m json.tool response.json                      # pretty-print JSON
python3 -c 'import yaml,sys,json; print(json.dumps(yaml.safe_load(sys.stdin)))' < values.yaml
python3 -c 'import secrets; print(secrets.token_urlsafe(32))'
python3 -c 'import socket; print(socket.gethostbyname("db.internal"))'
python3 -m timeit '"-".join(map(str, range(100)))'
```

</details>

### Q195. Rapid fire: pytest commands. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
pytest -q                                   # quick run
pytest -x --lf                              # stop at first failure, rerun last failed
pytest -k "sg and not slow" -vv             # filter by name expression
pytest tests/test_cli.py::test_exit_code    # single test
pytest --cov=src --cov-report=term-missing --cov-fail-under=80
pytest -n auto                              # parallel (pytest-xdist)
pytest --pdb                                # drop into debugger on failure
```

</details>

### Q196. Rapid fire: packaging and quality commands. 🟢

<details>
<summary><b>Answer</b></summary>

```bash
python -m build                             # wheel + sdist into dist/
twine check dist/*
pip install dist/cloudaudit-0.1.0-py3-none-any.whl
ruff check . --fix && ruff format .
mypy src
pip-audit -r requirements.txt               # known CVEs
bandit -r src
pip-compile --generate-hashes requirements.in
```

</details>

### Q197. Rapid fire: boto3 snippets you should be able to write from memory. 🟡

<details>
<summary><b>Answer</b></summary>

```python
boto3.client("sts").get_caller_identity()["Arn"]                      # who am I
boto3.Session(profile_name="prod").client("s3", region_name="eu-west-1")
ec2.get_paginator("describe_instances").paginate(Filters=[...])
ec2.get_waiter("instance_stopped").wait(InstanceIds=[iid])
e.response["Error"]["Code"]                                            # in except ClientError as e
Config(retries={"mode": "adaptive", "max_attempts": 10})
s3.generate_presigned_url("get_object", Params={...}, ExpiresIn=900)
```

</details>

### Q198. Rapid fire: debugging and profiling commands. 🔴

<details>
<summary><b>Answer</b></summary>

```bash
python -X importtime -c "import cloudaudit.cli" 2>&1 | sort -t'|' -k2 -n | tail   # slow imports
python -m cProfile -s cumtime script.py | head -30
py-spy top --pid 1234                     # sample a running process, no restart
py-spy dump --pid 1234                    # stack of every thread (hung process)
python -X faulthandler script.py          # dump traceback on segfault/hang signal
python -m pdb script.py                   # or breakpoint() in code
PYTHONASYNCIODEBUG=1 python app.py        # slow callbacks, unawaited coroutines
```

`py-spy dump` is the fastest way to see where a stuck production script is blocked.

</details>

### Q199. Rapid fire: Kubernetes and Docker client snippets. 🟡

<details>
<summary><b>Answer</b></summary>

```python
config.load_kube_config(context="prod-eks")
client.CoreV1Api().list_namespaced_pod("prod", label_selector="app=api")
client.CoreV1Api().read_namespaced_pod_log("api-7d9f", "prod", tail_lines=100, container="api")
client.AppsV1Api().patch_namespaced_deployment_scale("api", "prod", {"spec": {"replicas": 0}})
docker.from_env().containers.list(all=True, filters={"status": "exited"})
docker.from_env().images.build(path=".", tag="app:dev", rm=True)
```

</details>

### Q200. Rapid fire: what are the three questions you ask before running any destructive Python script in production? 🟢

<details>
<summary><b>Answer</b></summary>

1. **What will it touch?** Run it with `--dry-run` (and EC2 `DryRun=True` where available) and read the plan, including the account (`sts get-caller-identity`) and region.
2. **Can I undo it?** Snapshots, backups, versioning, or a documented rollback exist before the run.
3. **What limits the blast radius?** Scoped IAM role, explicit filters (tags, account, region), a maximum-items safety cap, and confirmation for counts above a threshold.

Interviewers ask this to see whether you treat automation as production code.

</details>

[⬅ Back to Python for DevOps](./README.md) · [🏠 Home](../README.md)

<!-- DEVOPS-SHACK:FOOTER -->
---

<div align="center">

### 🚀 Made with ❤️ by [DevOps Shack](https://www.youtube.com/@devopsshack)

**Learn DevOps, DevSecOps, Cloud and Kubernetes the practical way.**

[▶️ YouTube — DevOps Shack](https://www.youtube.com/@devopsshack) · [▶️ Hindi — @shackverse](https://www.youtube.com/@shackverse) · [🎓 devopsshack.com](https://devopsshack.com) · [📸 Instagram @devopsshack](https://www.instagram.com/devopsshack)

[🏠 Back to Home](../README.md) · [⭐ Star the original repo](https://github.com/jaiswaladi246/DevOps-Cloud-Interview-Preparation-by-DevOps-Shack)

<sub>© DevOps Shack · Licensed CC BY-NC-SA 4.0 · If you found this copy anywhere other than <a href="https://github.com/jaiswaladi246/DevOps-Cloud-Interview-Preparation-by-DevOps-Shack">github.com/jaiswaladi246</a>, the original and latest version is maintained by DevOps Shack.</sub>

</div>
<!-- /DEVOPS-SHACK:FOOTER -->
