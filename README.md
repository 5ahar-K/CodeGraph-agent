# Code-Mapper

A tool that reads through a Python codebase and figures out how all the functions actually connect to each other: who calls what, whose output feeds into what, and which functions would cause the most damage if broken. Once it has that map, it uses an LLM to answer questions about the code and even write tests, but only using real information from the map, not hallucinations.

I built this after talking to someone from Conduct at a careers fair. I wanted to try building a small version of a Conduct product which helps companies make sense of huge, messy, undocumented codebases, from scratch, to see how hard it actually is and what breaks along the way.

## How it functions

1. Reads every `.py` file in a repo and parses it into an AST (Python's own way of representing code as a tree).
2. Turns every function and method into a node in a graph. Every function call becomes an edge: "A calls B."
3. Because lots of codebases reuse the same function name in different files or classes, it tries to work out which actual function a call is pointing to, using a few simple rules (same file first, then imports, then just picking whichever it can).
4. On top of "who calls who," it also tracks something a bit more specific: if a function's return value gets passed straight into another function, that's a real data dependency, not just a call, so this gets tracked separately.
5. You can ask it questions about any function, such as *"what would break if I changed this?"*, and it answers using only that function's real source code plus its actual callers and callees, nothing invented. If the graph does not have enough information, it says so instead of guessing.
6. It can also generate a basic unit test for a function.
7. And it can rank every function in the repo by **blast radius**, which is basically how many other functions would be affected if you changed it, so you can spot the riskiest parts of a codebase at a glance.

## Bugs I actually ran into (and what they taught me)

The most useful parts of this project were not the bits that worked first try. They were the moments something *looked* fine until I checked it against the real code.

**Same function name, different files, silently overwriting each other.**
Click's own repo has two completely unrelated functions both called `echo`. At first, I was only keying nodes by function name, so the second `echo` I found just quietly replaced the first one in my data, no error, just wrong information. Fixed by keying every function as `(file, class, function_name)` instead of just the name.

**Same function name, different classes, same file.**
The Gilded Rose kata has two classes in one file, each with their own `__init__`. I was using `ast.walk()` to scan the file, which flattens everything and loses track of which class a method actually belongs to. Both `__init__`s collided under the same key too. Fixed by walking the tree in a way that keeps track of which class you're inside.

**`what_calls` using the wrong shape of key.**
I noticed this one just by rereading my own code: the graph nodes are all 3-part keys `(file, class, function)`, but `what_calls` was building a 2-part key `(file, function)` to look someone up. That key would never match anything in the graph, it would just quietly fail. Simple fix, but the kind of bug that's easy to miss because it doesn't throw an obvious error.

**`AttributeError: 'FunctionDef' object has no attribute 'parent'`**
When I tried to fetch a specific function's source code and check whether it belonged to a certain class, I needed to look at "the parent of this node." Turns out Python's `ast` module doesn't track parent-child relationships by default, a node only knows its children, never who it belongs to. Fixed by writing a small helper that walks the tree once and manually attaches a `.parent` attribute to every node before I try to use it.

**A "no answer" that was actually correct.**
At one point I asked the tool what would break if a function's return type changed, and it said it couldn't tell, because at that point, the graph only tracked "A calls B," not "A's return value flows into B." That's not a bug, that's the tool working exactly as intended: refusing to guess when it genuinely doesn't have enough information. It's also what pushed me to go build the return-value tracking part next.

## Testing it on a real, messy codebase: Mailpile

Everything above was worked out on small teaching examples (Click, Gilded Rose kata) where I could predict the right answer before running anything. Once the logic held up there, I ran the whole pipeline on **Mailpile**, a real, old, fairly large open-source email client, to see what breaks when the code is genuinely messy instead of a clean teaching example.

**Results:**

- **2,538 functions found, 15,118 call relationships**, much bigger than any kata I'd tested before.
- **8 files failed to parse at all**, all for genuine reasons: old Python 2-style print statements without brackets, mixed tabs and spaces, and a few plain syntax errors. Nothing to fix on my end, this is just old code.
- **219 function names were ambiguous** (defined in more than one place). The single biggest one, unsurprisingly, was `__init__`, defined 137 times, once per class, exactly as you'd expect.
- Out of **4,163 calls** where the name matched more than one function, my resolution rules only managed to narrow **544** down to one clear answer. The other **3,619** stayed genuinely ambiguous, meaning the tool honestly couldn't tell which specific function was meant.
- Digging into a few of those ambiguous ones, the pattern was clear: they're mostly ordinary-sounding names like `open`, `join`, `read`, `save`, `decode`, names that happen to be reused constantly across a big codebase in totally unrelated places. One case, a function called `_open_log`, had a call to `save` that was ambiguous between **12 different candidates**.

That last point is the most honest and useful finding from the whole project: my resolution heuristic works reasonably well on small code, but real-world codebases reuse common short names far more than a small kata does, so the ambiguity rate jumps a lot at scale. That's a genuine limitation, not something I'm going to pretend is solved.

## What it still can't do (on purpose, not fixed yet)

- It can't work out which specific class a `self.method()` call belongs to when there are multiple classes with a method of that name. That needs real type inference, which is a much bigger project on its own.
- Return-value tracking only catches the simple case: `x = some_function()` then `x` gets passed straight into another call. It doesn't yet handle things like unpacking (`a, b = f()`) or attribute access (`f().something`).
- It doesn't resolve relative imports (`from . import x`) back to a specific file.
- When a call is genuinely ambiguous, it currently just connects to every possible candidate rather than picking one, which can make some functions look riskier ("higher blast radius") than they really are. Worth eyeballing the top results rather than trusting the numbers blindly.

## How to run it

```bash
pip install networkx google-generativeai
```

You'll need a free Gemini API key from [aistudio.google.com/app/apikey](https://aistudio.google.com/app/apikey).

```python
g, function_bodies, name_to_keys = build_graph("target_repo")

# Ask a question about a function, grounded in its real code
ask_about_function(g, key, "What would break if this function's return type changed?")

# Or generate a test for it
generate_test(g, key)

# Or see which functions are riskiest to touch
for key, score in top_blast_radius(g, n=10):
    print(score, key)
```

There's also a simple interactive loop. Type a function name, and it'll ask you which one you meant if there are several, then let you ask it questions or generate a test.

## Further steps

- Basic type inference, to fix the `self.method()` resolution problem.
- Handle tuple-unpacking and chained calls in return-value tracking.
- Look more closely at which ambiguous calls actually matter (a call to `open` probably doesn't matter much, a call to a genuinely custom, similarly-named internal function might).
