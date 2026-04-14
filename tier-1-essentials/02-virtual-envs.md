# Virtual Environments

## What is it?

A virtual environment is an isolated directory tree that contains its own Python interpreter, packages, and scripts — separate from the system-wide Python and from every other project on your machine. Think of it like a quarantine room for your project's dependencies. When you install a package into a virtual environment, it goes into that specific folder and nowhere else. Every other project on your machine, including the system Python, is completely unaffected.

The practical analogy: imagine you have a kitchen (your system Python). If you bake a cake in the shared kitchen, the flour, eggs, and sugar you used are mixed in with everyone else's ingredients. A virtual environment is like putting your cake ingredients in a sealed plastic bin before you start — nothing contaminates the shared space, and nothing in the shared space contaminates your project.

This isolation matters enormously because Python projects depend on precise package versions. A package that works with one version of a library will often break with another. Without isolation, you end up with a global mess where Project A needs `requests>=2.28` and Project B needs `requests<2.0`, and there is no way to satisfy both.

## Why it matters for vibe coding

Vibe coding amplifies dependency chaos by an order of magnitude. When you ask an AI to build something, it reaches for packages prolifically and often without verifying what you already have installed. Here is what goes wrong without virtual environment discipline:

**The AI installs globally and corrupts your system.** You ask an AI to build a data processing script. It says "no problem, just run `pip install pandas numpy matplotlib`." You run it, it works, and your AI coding assistant suddenly breaks because it depended on a specific older version of `numpy` that you just overwrote. Your whole workspace is now broken and you do not know why.

**The "works on my machine" nightmare.** The AI generates a script that runs perfectly on your machine. You share it with a colleague or try to deploy it. It crashes immediately because the deployment machine has different package versions. This is the classic dependency hell that virtual environments exist to prevent — and without them, you will encounter it every single time.

**Unreproducible AI output.** When AI generates code using whatever packages happen to be installed at that moment, the output is not reproducible. Send the same prompt a month later and you get code that behaves differently because the underlying packages have updated. A lockfile (covered below) pins the versions at the time the environment was created, making AI output reproducible across time and machines.

**Dependency conflicts across AI sessions.** If you are working on multiple projects and use AI tools across all of them, each project needs its own isolated environment. Without isolation, an AI building your web app might pull in a package that breaks the script an AI built for your data analysis side project last week.

## The 20% you need to know

### Python virtual environments: the core tools

**venv** is the standard library built into Python 3.3+. It creates a minimal virtual environment without any extra dependencies. You activate it, and your terminal session now uses that environment's Python interpreter and packages.

```bash
# Create an environment
python -m venv .venv

# Activate it (Linux/macOS)
source .venv/bin/activate

# Activate it (Windows PowerShell)
.venv\Scripts\Activate.ps1

# Now pip install goes into .venv, not the system
pip install requests
```

**uv** is the tool you should actually use. It is a modern Python package manager from the Astral team (who also build Ruff) that is 10-100x faster than pip, handles dependency resolution correctly (pip often gets it wrong with complex dependency trees), and creates virtual environments as part of its workflow. The PRD and this curriculum recommend `uv` because it eliminates an entire class of AI-related dependency headaches.

```bash
# Install uv (once)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Create a new project with a virtual environment automatically
uv init --name myproject

# Or create a venv and install packages in one step
uv venv .venv
uv pip install requests flask
```

The key advantage of `uv pip install` is that it resolves conflicts before installing, not after. With pip, you can install Package A, then install Package B (which depends on a different version of Package A), and pip will happily let you create an impossible situation. `uv` catches this and tells you upfront.

**Conda and Miniforge** cover situations where you need to manage multiple language runtimes or work with packages that have complex binary dependencies (e.g., scientific computing packages with C extensions). Miniforge is a community-owned conda distribution that uses conda-forge channels by default, which is more transparent than the default Anaconda channels.

```bash
# Miniforge (recommended over stock conda)
conda create -n myproject python=3.11
conda activate myproject
```

Conda shines for data science stacks where `numpy`, `scipy`, and `pandas` need to be compiled with specific BLAS/LAPACK backends. For most web app and script work, `uv` is simpler and faster.

### The lockfile: your reproducibility safety net

A lockfile records the exact version of every package installed in an environment, plus the versions of all their transitive dependencies. When you share your project or move it to a different machine, you can recreate the exact same environment from the lockfile — not "approximately the same environment."

Without a lockfile, `pip install requests` installs whatever `requests` version is current today. In six months, a fresh install gets a different version that may behave differently. This is how "it worked yesterday" becomes "it's broken now" without any code changes.

```bash
# uv generates a lockfile automatically when you add packages
uv pip install requests
# This creates uv.lock with pinned versions

# Recreate the exact environment from the lockfile
uv pip install --frozen  # installs exactly what is in uv.lock
```

The equivalent with pip:
```bash
pip freeze > requirements.txt   # create the lockfile
pip install -r requirements.txt  # recreate from lockfile
```

But pip's `freeze` output is not guaranteed to be reproducible across platforms (binary wheels differ) and does not handle complex dependency resolution as robustly as `uv`.

### Node.js environments: nvm

Node.js uses a different model: the package manager (npm or pnpm) installs packages into a `node_modules` folder in the project directory, which provides project-level isolation without a separate activation step. However, managing Node.js versions across projects is a real problem — a project may need Node 18 while another needs Node 20.

**nvm** (Node Version Manager) solves this. It lets you switch between Node versions per project without affecting the system Node.

```bash
# Install nvm (once)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# Use a specific Node version in the current shell
nvm use 20
nvm use --lts
```

In a vibe coding context, AI tools often generate code that requires a specific Node version (especially with newer framework features). Using nvm ensures you can match what the AI assumed when it generated the code.

### pyenv: managing Python versions

**pyenv** manages multiple Python interpreter versions on a single machine. Similar to nvm but for Python. Useful when different projects need different Python versions.

```bash
# Install a specific Python version
pyenv install 3.11.7

# Set it for the current directory (creates .python-version file)
pyenv local 3.11.7
```

The `.python-version` file signals to pyenv which version to use automatically when you cd into that directory.

## Hands-on exercise

**Goal:** Create an isolated Python environment using `uv`, install specific packages, generate a lockfile, verify isolation, and simulate the "works on my machine" problem.

**Time:** 10-15 minutes

**Step 1: Verify you have uv installed (or install it)**
```bash
which uv || curl -LsSf https://astral.sh/uv/install.sh | sh
```

**Step 2: Create a project directory and a virtual environment**
```bash
mkdir ~/vibe-env-practice
cd ~/vibe-env-practice
uv venv
```

You should see `.venv` appear in the directory. The activation happens automatically in the next step.

**Step 3: Install packages and observe the lockfile creation**
```bash
uv pip install requests==2.31.0 urllib3==2.0.0
```

After this, you will have two new files in the directory:
- `uv.lock` — the lockfile with pinned versions
- `.venv/` — your isolated environment

**Step 4: Verify the environment is isolated**
```bash
# Show what is in the virtual env (should show requests and urllib3)
uv pip list

# Compare with your system-wide packages (should NOT show requests if you never installed it globally)
python -c "import requests; print(requests.__version__)"  # FAILS in system Python
source .venv/bin/activate && python -c "import requests; print(requests.__version__)"  # WORKS in venv
```

**Step 5: Simulate the problem without a lockfile**
```bash
# In your project directory with venv active
source .venv/bin/activate
uv pip install requests  # installs latest version

# Now change the pinned version and notice what happens
uv pip install requests==2.30.0
uv pip freeze  # notice the version changed
```

**Step 6: Restore from lockfile (the fix)**
```bash
# First, install a conflicting version to see the problem
uv pip install requests==2.25.0

# Now restore from lockfile — this should revert to the locked version
uv pip install --frozen

# Verify the version is back to what was locked
uv pip list | grep requests
```

**Step 7: Cleanup**
```bash
deactivate  # exit the venv
cd ~
rm -rf ~/vibe-env-practice
```

You should have observed:
1. The `.venv` folder is completely separate from the system Python
2. `uv.lock` records exact versions
3. `uv pip install --frozen` restores the exact locked state, reverting the conflicting installation

## Common mistakes

### 1. Forgetting to activate the environment

**What happens:** You run `pip install requests` and your script crashes with `ModuleNotFoundError`. The package went into the system site-packages, but `python script.py` uses the system interpreter — which does not have it.

**Why it happens:** There is no visual indicator in most terminals by default. The `.venv` prefix that signals an activated environment is easy to miss or forget.

**How to fix it:** Always check for the `(.venv)` prefix before running install or execution commands. If in doubt, run `which python` — it should point to `.venv/bin/python`.

### 2. Running AI-generated install commands without a virtual environment

**What happens:** An AI says "just run `pip install langchain openai tiktoken`." You run it globally. It works today. Next week your other project breaks because `openai` updated to a version incompatible with what `langchain` requires.

**Why it happens:** Global package installations modify shared state that affects every project on the machine.

**How to fix it:** Get into the habit of asking "is this inside an activated venv?" before running any install command. Make activation a reflex.

### 3. Committing the virtual environment folder to version control

**What happens:** Your git repository is 500MB because `.venv` got committed. CI builds fail because the binary artifacts are platform-specific.

**Why it happens:** `.venv` is a folder in your project directory. Without a `.gitignore` entry, Git tracks it automatically.

**How to fix it:** Add `.venv/` to your `.gitignore`. Commit the lockfile (`uv.lock` or `requirements.txt`) — not the binary artifacts.

### 4. Not regenerating the lockfile after adding a new dependency

**What happens:** You add a new package. It works locally. You push to production. A fresh CI machine installs from the stale lockfile and gets a different version of a transitive dependency, causing a conflict.

**Why it happens:** The lockfile was not updated after the new package was added, so the CI environment has an incomplete or outdated snapshot.

**How to fix it:** After adding any package, regenerate the lockfile and commit it alongside the code change: `uv pip freeze > uv.lock` or `pip freeze > requirements.txt`.

### 5. Mixing global and venv Python in AI coding sessions

**What happens:** Your AI coding tool runs with its own Python environment, separate from your project's venv. The AI generates code that imports a package, you run it in your project's venv, and it fails because the package is not installed there.

**Why it happens:** AI coding tools run in their own context. The AI does not automatically know which venv you have activated.

**How to fix it:** Make your AI tool aware of your project environment: activate the venv before starting the session, explicitly tell the AI which packages are available, or configure the tool to point at your project directory.

## AI-specific pitfalls

### AI suggests `pip install` without mentioning the virtual environment

This is the most common AI-generated mistake. AI coding assistants often produce install commands that assume a global or pre-configured environment. You will see code like:

```
!pip install transformers
```

or

```
pip install -q transformers datasets
```

When you run these in a terminal without an activated venv, the packages go into the system Python — which is exactly the problem described above.

**What to look for:** Any `pip install` block that lacks an activating step. When you see an install suggestion, ask: "Where does this go? Is my venv activated right now?"

**How to fix it:** Prepend the activation step yourself, or add a comment asking the AI to specify the environment. A good AI response should look like:

```bash
source .venv/bin/activate
pip install transformers
```

### AI does not generate or update lockfiles

AI tools often generate code and install packages but do not naturally produce the lockfile that makes the environment reproducible. You end up with a project that works today but has no record of which package versions made it work.

**What to look for:** After a session where the AI installed packages, check whether `uv.lock` or `requirements.txt` was updated. If it was not, the environment is not reproducible.

**How to fix it:** After any AI-assisted package installation, run `uv pip freeze > uv.lock` or `pip freeze > requirements.txt` and commit the updated lockfile. Consider this a mandatory step after any AI install command.

### AI does not understand environment activation persistence

AI tools generate scripts and configuration that assume the environment is activated at the time of execution. But scheduled jobs, CI pipelines, and other automated runs start fresh — with no activated venv. AI-generated code will fail in these contexts because the environment activation from a previous terminal session does not persist.

**What to look for:** AI-generated scripts that assume packages are available without activating the environment first. Any script that runs in a cron job, CI step, or systemd service.

**How to fix it:** Always use the full path to the venv Python interpreter in automated contexts, or use a virtual environment activator tool that handles activation automatically:

```bash
# Instead of relying on activation
/path/to/.venv/bin/python script.py

# Or with uv (which handles this better)
uv run python script.py  # runs inside the project's venv automatically
```

### AI generates code requiring a specific package version without specifying it

AI will often write code that relies on a feature from a specific package version, but generates the install command without pinning it. The code works on the AI's training data environment (which may have an old version installed) but fails on a fresh machine with the current version (or vice versa).

**What to look for:** Install commands like `pip install pandas` with no version specifier, followed by code using a feature that was added in a recent release. Or install commands with no version when the code uses a feature from a specific older release.

**How to fix it:** Always specify versions in install commands: `uv pip install requests==2.31.0`. If the AI gives you a package without a version, look up the version that introduced the feature the code uses and pin to that.

## Quick reference

### Decision tree: which tool to use

```
Do you need to manage multiple Python versions on one machine?
├── Yes → Use pyenv
└── No → Do you need complex binary dependencies (numpy, scipy, etc.)?
    ├── Yes → Use conda (or miniforge)
    └── No → Use uv (fast, modern, good lockfile support)
```

### Core commands

| Task | uv | venv | conda |
|---|---|---|---|
| Create environment | `uv venv .venv` | `python -m venv .venv` | `conda create -n name python=3.11` |
| Activate | Automatic with `uv run` | `source .venv/bin/activate` | `conda activate name` |
| Install package | `uv pip install pkg` | `pip install pkg` | `conda install pkg` |
| List packages | `uv pip list` | `pip list` | `conda list` |
| Generate lockfile | `uv pip freeze > uv.lock` | `pip freeze > requirements.txt` | `conda env export > env.yml` |
| Install from lockfile | `uv pip install --frozen` | `pip install -r requirements.txt` | `conda env create -f env.yml` |
| Run a script in env | `uv run python script.py` | Activate first, then `python script.py` | Activate first, then `python script.py` |

### Key files to commit

| File | What it is | Commit it? |
|---|---|---|
| `.venv/` | The virtual environment folder | NO — add to `.gitignore` |
| `uv.lock` | uv's lockfile with exact versions | YES |
| `requirements.txt` | pip's frozen dependencies | YES |
| `pyproject.toml` | uv/Poetry project configuration | YES |
| `environment.yml` | conda environment definition | YES |
| `.python-version` | pyenv version specifier | YES |

## Go deeper

1. **uv documentation** — The official uv docs cover installation, workflows, and lockfile management in detail. [https://docs.astral.sh/uv/](https://docs.astral.sh/uv/) (verified 2026-04)

2. **Python virtual environments — official documentation** — The Python docs explain venv thoroughly, including the activation mechanics that trip up beginners. [https://docs.python.org/3/library/venv.html](https://docs.python.org/3/library/venv.html) (verified 2026-04)

3. **uv vs. pip: Which Python package manager should you use?** — Astral's official comparison covering the real trade-offs in reproducibility, speed, and dependency resolution that matter for AI-assisted workflows. [https://astral.sh/blog/uv-vs-pip](https://astral.sh/blog/uv-vs-pip) (verified 2026-04)

4. **nvm documentation** — The official nvm readme covers installation, usage, and common gotchas. [https://github.com/nvm-sh/nvm](https://github.com/nvm-sh/nvm) (verified 2026-04)

5. **Python virtual environments: A mental model guide** — A deeper mental model for understanding why virtual environments exist and how they interact with the operating system. [https://realpython.com/python-virtual-environments-a-pragmatic-guide/](https://realpython.com/python-virtual-environments-a-pragmatic-guide/) (verified 2026-04)
