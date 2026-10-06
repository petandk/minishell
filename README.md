*This project has been created as part of the 42 curriculum by rmanzana, gpolo*

# Minishell

## Description

**Minishell** is a simple Unix shell written in C, inspired by **bash**. It displays a prompt, reads commands with `readline`, and executes them with support for pipes, redirections, heredocs, environment variables, quotes and signals, plus a set of built-in commands.

The project is a deep dive into how a shell really works: parsing a command line into tokens, handling quotes and expansions, creating processes and pipes, redirecting file descriptors, and reacting correctly to signals like `Ctrl-C`.

### Features

- Interactive prompt with **command history** (`readline`).
- Executables are found through the `PATH` variable, or run with a relative or absolute path.
- **Quotes:** `'single quotes'` prevent any interpretation; `"double quotes"` prevent everything except `$` expansion. Unclosed quotes are detected as an error.
- **Redirections:**
  - `<` redirects input from a file
  - `>` redirects output to a file (truncate)
  - `>>` redirects output to a file (append)
  - `<<` **heredoc**: reads input until a delimiter line, with variable expansion unless the delimiter is quoted
- **Pipes** (`|`) connecting any number of commands.
- **Environment variables:** `$VAR` is expanded to its value, and `$?` to the exit status of the last pipeline.
- **Signals**, behaving like bash in interactive mode:
  - `Ctrl-C` shows a new prompt on a new line
  - `Ctrl-D` exits the shell
  - `Ctrl-\` does nothing
- **Built-ins:**

| Command | Behaviour |
|---|---|
| `echo` | Prints its arguments, with option `-n` |
| `cd` | Changes directory, with a relative or absolute path |
| `pwd` | Prints the current directory |
| `export` | Sets environment variables (prints them sorted with no arguments) |
| `unset` | Removes environment variables |
| `env` | Prints the environment |
| `exit` | Exits the shell with an optional numeric status |

## How it works

The shell runs a **read → parse → execute** loop:

1. **Read.** `readline()` shows the prompt and returns the line, which is added to the history.
2. **Tokenize.** The line is split into tokens, respecting quotes. Each token records whether it is an operator (`|`, `<`, `>`, `<<`, `>>`) and where its quotes start and end, so expansion can later tell quoted and unquoted parts apart. Syntax errors (unclosed quotes, misplaced operators) are detected at this stage.
3. **Build commands.** Tokens are grouped into commands separated by pipes. Each command stores its arguments, its input and output files and its heredocs.
4. **Expand.** `$VAR` and `$?` are replaced with their values, except inside single quotes, and the quotes are removed.
5. **Heredocs.** All heredocs are read **before** execution, in a child process, so `Ctrl-C` can cancel them cleanly without killing the shell.
6. **Execute.**
   - A single built-in runs **in the shell process itself**, so commands like `cd`, `export` or `exit` can change the shell's state.
   - Otherwise, one child process is created per command with `fork()`, connected with `pipe()`. Redirections are applied with `dup2()`, then the child runs the built-in or calls `execve()`.
   - The parent waits for every child and stores the exit status of the last one in `$?`.

### Design choices

- **The environment is a linked list** of `name` / `value` pairs, which makes `export` and `unset` simple, and it is converted back to a `char **` only when calling `execve()`.
- **A single global variable**, a `volatile sig_atomic_t`, is used to report that a signal was received, as the subject requires. Different signal handlers are installed for the interactive prompt, for heredocs and for child processes.
- **Memory:** everything is freed after each command; the only leaks Valgrind reports come from `readline` itself, and they are silenced with `readline.supp`.

## Instructions

### Requirements

The project links against the system's `readline` library. It is already installed on the 42 computers; on a fresh Debian / Ubuntu machine:

```bash
sudo apt install libreadline-dev
```

### Compilation

```bash
make          # builds libft and the minishell executable
make clean    # removes object files
make fclean   # removes object files and the executable
make re       # rebuilds everything
```

### Usage

```bash
./minishell
```

```bash
Minishell > echo "Hello $USER" > greeting.txt
Minishell > cat << EOF | grep -v skip | wc -l
> line one
> skip this
> line two
> EOF
2
Minishell > ls nonexistent
ls: cannot access 'nonexistent': No such file or directory
Minishell > echo $?
2
Minishell > export NAME=42
Minishell > env | grep NAME
NAME=42
Minishell > exit
```

To check for memory leaks while ignoring those from `readline`:

```bash
valgrind --leak-check=full --show-leak-kinds=all --suppressions=readline.supp ./minishell
```

## Resources

- [Bash Reference Manual — GNU](https://www.gnu.org/software/bash/manual/bash.html)
- [`readline(3)` manual](https://tiswww.case.edu/php/chet/readline/readline.html)
- [Writing Your Own Shell — Purdue University (CS 252)](https://www.cs.purdue.edu/homes/grr/SystemsProgrammingBook/Book/Chapter5-WritingYourOwnShell.pdf)
- [Tutorial — Write a Shell in C — Stephen Brennan](https://brennan.io/2015/01/16/write-a-shell-in-c/)
- [`signal(7)`](https://man7.org/linux/man-pages/man7/signal.7.html), [`fork(2)`](https://man7.org/linux/man-pages/man2/fork.2.html), [`pipe(2)`](https://man7.org/linux/man-pages/man2/pipe.2.html), [`dup2(2)`](https://man7.org/linux/man-pages/man2/dup.2.html) and [`execve(2)`](https://man7.org/linux/man-pages/man2/execve.2.html) manuals
