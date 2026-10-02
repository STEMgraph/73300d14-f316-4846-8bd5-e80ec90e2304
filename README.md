<!---
{
  "id": "73300d14-f316-4846-8bd5-e80ec90e2304",
  "teaches": "GNU Make: Recursive versus Non-Recursive Make",
  "depends_on": [
    "b9b4c728-6b02-40a0-94d3-18dae9aeb49a"
  ],
  "author": "Stephan Bökelmann",
  "first_used": "2026-10-02",
  "keywords": [
    "make",
    "recursive make",
    "$(MAKE) -C",
    "include",
    "non-recursive",
    "sub-makefile",
    "dependency graph"
  ]
}
--->

# GNU Make: Recursive versus Non-Recursive Make

> In this exercise you will build a project with a library and an application in separate directories twice: first with a top-level `Makefile` that calls `make` in each subdirectory, then with a single `make` process that includes the subdirectories' rules. You will see why the second approach knows about dependencies that the first one cannot see. No `sudo` is required.

## Introduction

Once a project has more than one directory, there are two ways to organise the `Makefile`s:

- **Recursive make**: every directory has its own complete `Makefile`, and the top-level one runs `$(MAKE) -C lib` followed by `$(MAKE) -C app`. Simple to understand, and the default in many old projects.
- **Non-recursive make**: there is one `make` process. Every directory contributes a fragment (`lib/rules.mk`) that the top-level `Makefile` includes. All rules live in one dependency graph.

The difference matters as soon as a file in `app/` depends on a file in `lib/`. A recursive build only works if the directories are processed in the right order and completely; it cannot do `make -j` across directories, and `make app/main.o` from the top level does not know that `lib/libvec2.a` must exist first. Peter Miller's 1998 paper *Recursive Make Considered Harmful* explains the consequences in detail.

### 1.1) Further Readings and Other Sources

* [Peter Miller: Recursive Make Considered Harmful](https://aegis.sourceforge.net/auug97.pdf)
* [GNU Make Manual: Recursive Use of make](https://www.gnu.org/software/make/manual/html_node/Recursion.html)
* [GNU Make Manual: The `include` Directive](https://www.gnu.org/software/make/manual/html_node/Include.html)

## Tasks

### Task 1: The Project Layout

```bash
mkdir -p ~/make-tree/lib ~/make-tree/app && cd ~/make-tree
cp ~/vec2lib/vec2.c ~/vec2lib/vec2.h lib/
cp ~/vec2app/main.c app/
sed -i 's/#include <vec2.h>/#include "vec2.h"/' app/main.c
```

### Task 2: Recursive Make

```make
# file: lib/Makefile
CC = gcc
CFLAGS = -Wall -Wextra -O2
libvec2.a: vec2.o
	ar rcs $@ $^
vec2.o: vec2.c vec2.h
	$(CC) $(CFLAGS) -c $< -o $@
clean:
	rm -f vec2.o libvec2.a
.PHONY: clean
```

```make
# file: app/Makefile
CC = gcc
CFLAGS = -Wall -Wextra -O2 -I../lib
main_program: main.o ../lib/libvec2.a
	$(CC) $< -o $@ -L../lib -lvec2 -lm
main.o: main.c ../lib/vec2.h
	$(CC) $(CFLAGS) -c $< -o $@
clean:
	rm -f main.o main_program
.PHONY: clean
```

```make
# file: Makefile
SUBDIRS = lib app
all:
	for d in $(SUBDIRS); do $(MAKE) -C $$d || exit 1; done
clean:
	for d in $(SUBDIRS); do $(MAKE) -C $$d clean; done
.PHONY: all clean
```

```bash
make
./app/main_program
```

### Task 3: Where Recursion Breaks

```bash
make clean
make -C app
```

`app` cannot be built on its own: `../lib/libvec2.a` is missing and the `app/Makefile` has no rule for it. Now reverse `SUBDIRS` to `app lib` in the top-level file and run `make clean && make`. Then try parallelism:

```bash
make clean
make -j4
```

Run `make clean && make -j4` several times. Does it fail every time, sometimes, or never? Explain what decides that.

### Task 4: Non-Recursive Make

Replace the three files with one `Makefile` and two fragments:

```make
# file: lib/rules.mk
LIB_SRCS := lib/vec2.c
LIB_OBJS := $(LIB_SRCS:.c=.o)
lib/libvec2.a: $(LIB_OBJS)
	ar rcs $@ $^
```

```make
# file: app/rules.mk
APP_SRCS := app/main.c
APP_OBJS := $(APP_SRCS:.c=.o)
app/main_program: $(APP_OBJS) lib/libvec2.a
	$(CC) $< -o $@ -Llib -lvec2 -lm
```

```make
# file: Makefile
CC = gcc
CFLAGS = -Wall -Wextra -O2 -Ilib -MMD -MP

all: app/main_program

include lib/rules.mk
include app/rules.mk

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(LIB_OBJS) $(APP_OBJS) lib/libvec2.a app/main_program *.d */*.d

.PHONY: all clean
-include $(LIB_OBJS:.o=.d) $(APP_OBJS:.o=.d)
```

```bash
rm -f lib/Makefile app/Makefile
make clean && make
```

Note that all paths in the fragments are relative to the **top-level** directory, because there is only one `make` process and one working directory.

### Task 5: The Graph Is Complete

```bash
make clean
make app/main.o
make -j4
make -n app/main_program
touch lib/vec2.h
make -n
```

Every one of these does the right thing. `make -j4` is now safe at any degree of parallelism, because the dependency between `app/main_program` and `lib/libvec2.a` is a real edge in one graph.

### Task 6: Measure It

Add thirty dummy library files:

```bash
for i in $(seq 1 30); do printf 'int f%d(void){ return %d; }\n' $i $i > lib/f$i.c; done
sed -i 's|LIB_SRCS := lib/vec2.c|LIB_SRCS := $(wildcard lib/*.c)|' lib/rules.mk
make clean && time make
make clean && time make -j4
```

Compare the two times. Then put the recursive `Makefile`s from Task 2 back temporarily, with a `$(wildcard *.c)` in `lib/Makefile`, and time `make -j4` there. Which variant parallelises across directories?

### Task 7: A Fragment Template

Write a generic `rules.mk` that works for any directory using `$(lastword $(MAKEFILE_LIST))` to find its own location:

```make
DIR := $(dir $(lastword $(MAKEFILE_LIST)))
SRCS_$(DIR) := $(wildcard $(DIR)*.c)
```

Use it from both `lib/` and `app/` and print `$(MAKEFILE_LIST)` from a phony `info` target to see which files `make` has read.

## Questions

- Why does `make -C app` fail in Task 3 but `make app/main_program` succeed in Task 5?
- What information does the top-level recursive `Makefile` have about the files in `lib/`? What does the non-recursive one have?
- Why can `-j` be unsafe with recursive make even when `SUBDIRS` is in the right order?
- Why must all paths in a fragment be relative to the top-level directory?
- In a non-recursive build, two fragments define a variable `SRCS`. What happens? How does the `DIR`-prefixed naming in Task 7 help?
- Many large projects (the Linux kernel, for example) still use recursive make successfully. What do they do to avoid the problems from Task 3?

## Advice

Recursive make is not wrong, it is incomplete: each sub-make sees only part of the dependency graph and has to trust that the rest was built already. For a two-directory project with a fixed order that is fine. As soon as you want `make -j` across directories or want to build a single file from the top level, the non-recursive layout pays off.

The fragment pattern from Task 4 and 7 is what CMake generates internally: one big dependency graph, assembled from per-directory descriptions. If you can read `rules.mk` files, you can read the generated `Makefile`s in a CMake build directory too.
