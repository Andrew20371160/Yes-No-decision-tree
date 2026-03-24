# Yes–No Decision Tree (C++)

A user-built **binary (yes/no) decision tree** framework for designing interactive flows such as customer service scripts, diagnostic tools, educational quizzes, and decision-making aids.

This project is a **command-line application** that supports **multiple decision trees** stored in a **circular doubly linked list**, with two usage modes:

- **Engineer Mode**: create/edit/manage trees
- **User Mode**: interact with existing trees (read-only operations)

## Demo

A short usage demo is linked in the repo:

- `DEMO.md`: https://youtu.be/nvzJ_w7uGpU?si=ZHDFNNWviP_-3gPr  
- `readme.md`: https://youtu.be/nvzJ_w7uGpU?si=JFgR6351umiwEVQu

## Features

- **Tree construction**
  - Build interactively via CLI prompts
  - Load/build from a breadth-first formatted text file
  - Construct from a breadth-first array (constructor exists)
- **Two modes**: Engineer mode (full control) and User mode (restricted)
- **File management**: save/load trees while preserving structure using breadth-first serialization with “filling slots”
- **Multiple trees**: manage many decision trees in a list with navigation/search
- **Search**
  - Search by text (substring match)
  - Search by path/pattern like `010001` (`0` = left/false, `1` = right/true)
- **Tree management**
  - Update node text
  - Remove nodes (with options to delete or reattach subtrees)
  - Switch a node’s left/right subtrees
  - Switch only the *texts* of left/right children (without moving subtrees)

## Project structure

- `main.cpp` — program entry point; top-level menus for Engineer/User modes
- `tree_list.h/.cpp` — circular doubly linked list of trees (each node owns a `decision_tree*`)
- `decision_tree.h/.cpp` — binary decision tree implementation and CLI operations
- `DOCUMENTATION.MD` — detailed design/usage/API notes
- `features.readme`, `readme.md`, `DEMO.md` — short descriptive docs / demo links
- `README.md`

## Architecture overview

High-level relationships (as described in `DOCUMENTATION.MD`):

- `tree_list` manages multiple trees (circular doubly linked list)
- each list node contains a `decision_tree`
- `decision_tree` is a binary tree of `node` objects

### Core data structures

**Tree node** (`decision_tree.h`):

- `text`: question/answer string
- `left`: “false” branch
- `right`: “true” branch
- `parent`: parent pointer

**List node** (`tree_list.h`):

- `name`: tree name
- `dtree`: pointer to a `decision_tree`
- `next/prev`: circular doubly linked list links

## Building / compiling

The repository documentation includes this compile command:

```bash
g++ -o decision_tree main.cpp decision_tree.cpp tree_list.cpp
```

### Dependencies / notes

- Uses only standard headers for most features: `<iostream>`, `<string>`, `<queue>`, `<fstream>`, `<limits>`
- Also includes **`<conio.h>`** and uses **`getch()`** for single-key input in the interactive menus (commonly Windows/MSVC; may require adjustments for Linux/macOS).

## Running

After compiling:

```bash
./decision_tree
```

You’ll see a main menu:

- **Engineer Mode**: add/edit trees and perform full editing operations
- **User Mode**: select a tree and interact with it (navigate questions with true/false)

## How saving/loading works (file format)

Trees are saved/loaded in **breadth-first order** (level order).  
To preserve the shape of incomplete trees, the program writes a special “filling string” for null slots:

> `This is a filling slot and won't be put into the tree`

When loading, those lines are recognized and not inserted as nodes, but they still keep the correct structure by maintaining the null positions in traversal.

## Repository entry points (what to read first)

- Start here: `main.cpp`
- Multi-tree management: `tree_list.h`, `tree_list.cpp`
- Decision tree logic: `decision_tree.h`, `decision_tree.cpp`
- Deep dive: `DOCUMENTATION.MD`
