## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture & Design](#architecture--design)
3. [Core Components](#core-components)
   - [Data Structures](#data-structures)
   - [Decision Tree Class](#decision-tree-class)
   - [Tree List Class](#tree-list-class)
4. [API Reference](#api-reference)
   - [decision_tree Methods](#decision_tree-methods)
   - [tree_list Methods](#tree_list-methods)
   - [Helper Functions](#helper-functions)
5. [Usage Guide](#usage-guide)
6. [File Organization](#file-organization)

---

## Project Overview

The **Yes-No Decision Tree** is a C++ application that implements an interactive binary decision tree system. It allows users to create, manage, and interact with multiple decision trees stored in a circular doubly linked list structure. The system supports two modes of operation:

- **Engineer Mode**: Create, edit, and modify decision trees with full control
- **User Mode**: Interact with pre-built trees with read-only functionality

### Key Features

- Create and manage multiple binary decision trees
- Save/load trees to/from files in breadth-first format
- Search trees by text patterns or binary navigation paths
- Interactive navigation and tree manipulation
- Separate interfaces for engineers (creators) and users (consumers)

---

## Architecture & Design

### System Hierarchy

```
main.cpp (Entry Point)
    ├── create() - Engineer Interface
    ├── user() - User Interface
    └── main() - Program Driver
          |
          └── tree_list (Circular Doubly Linked List)
              └── list_node (contains decision_tree)
                  └── decision_tree (Binary Tree)
                      └── node (Binary Tree Node)
```

### Design Patterns

1. **Container Pattern**: `tree_list` manages multiple `decision_tree` instances
2. **Traverser Pattern**: Both tree and list use pointer-based traversal
3. **Mode-Based Access**: Engineer vs. User interfaces with different capabilities
4. **File Persistence**: Breadth-first serialization for tree storage

---

## Core Components

### Data Structures

#### Node Structure (decision_tree.h)

```cpp
struct node {
    string text;           // Question/answer text
    int parent_answer;     // Answer leading to this node (0=false, 1=true, -1=none)
    node *parent;          // Parent node pointer
    node *left;            // Left child (false branch)
    node *right;           // Right child (true branch)
};
```

#### List Node Structure (tree_list.h)

```cpp
struct list_node {
    decision_tree *dtree;  // Pointer to the tree itself
    string name;           // Name of the tree
    list_node *next;       // Next node in circular list
    list_node *prev;       // Previous node in circular list
};
```

### Decision Tree Class

**File**: `decision_tree.h` & `decision_tree.cpp`

A complete binary search tree implementation for managing yes/no decision paths.

#### Private Members
- `node *root` - Root of the binary tree
- `node *traverser` - Current position in the tree during navigation
- Helper functions for tree manipulation

#### Public Interface

See [decision_tree Methods](#decision_tree-methods) below.

### Tree List Class

**File**: `tree_list.h` & `tree_list.cpp`

A circular doubly linked list that manages multiple decision trees.

#### Private Members
- `list_node *tail` - Tail pointer of circular list
- `list_node *traverser` - Current tree pointer
- Helper function: `get_list_node()` for memory allocation

#### Public Interface

See [tree_list Methods](#tree_list-methods) below.

---

## API Reference

### decision_tree Methods

#### Construction & Destruction

```cpp
decision_tree::decision_tree(void);
```
Creates an empty decision tree with `root` and `traverser` set to NULL.

```cpp
decision_tree::decision_tree(string *arr, int size);
```
Creates a tree from a breadth-first array initialization. Empty strings ("") represent null slots.

```cpp
~decision_tree(void);
```
Destructor. Prompts user to save before deleting the tree from heap memory.

#### Tree Building

```cpp
void construct(void);
```
Interactive interface for the engineer to build the tree. Guides user through adding questions/answers node by node, handling relationships to parent questions.

```cpp
void insert(void);
```
Insert a single question/answer node. User can search by text or manually navigate to insertion point.

#### Tree Persistence

```cpp
void save(void);
```
Saves the entire tree to a file in breadth-first order. Handles incomplete trees by using a "filling string" for null nodes, preventing tree structure corruption during reload.

```cpp
void load(void);
```
Loads a tree from a file in breadth-first format. Prompts to save current tree if one exists. Handles filling strings correctly to restore tree structure.

#### Navigation & Search

```cpp
bool move(void);
```
Interactive tree navigation. User can:
- `w` - Move to parent node
- `a` - Move to left child (false branch)
- `d` - Move to right child (true branch)
- `s` - Display entire tree
- `q` - Quit
- `Enter` - Stop at current node

Returns `true` if user presses Enter, `false` if quit.

```cpp
bool search(void);
```
Main search interface offering three methods:
1. Search by text (substring matching)
2. Search by pattern (binary path like "010001")
3. Manual navigation via `move()`

Returns `true` if node found, `false` otherwise.

```cpp
bool search_by_text(string &t);
```
Breadth-first search using substring matching. Time complexity: **O(N)**.
Returns `true` on first match, sets `traverser` to found node.

```cpp
bool search_by_pattern(string &pattern);
```
Binary path navigation. Pattern uses '0' for left (false) and '1' for right (true).
Example: "010" means left-right-left traversal.
Time complexity: **O(log₂ N)** for balanced trees.

#### Tree Modification

```cpp
void update(void);
```
Search for a node and update its text content. Preserves tree structure.

```cpp
void remove(void);
```
Delete a node with complex options:
- If node is root: Select which child becomes new root or delete entire tree
- If node has children: Choose to delete subtrees or reattach them elsewhere
- If node is leaf: Simply disconnect from parent

```cpp
void switch_nodes(void);
```
Swap the left and right subtrees of a node (including all descendants).

```cpp
void switch_answers(void);
```
Swap the text content of left and right children only (subtrees stay attached).

#### Tree Display

```cpp
void print(void);
```
Breadth-first tree display with level-by-level formatting.
Example output:
```
Question1 , Question2 , Question3 ,
Answer2a , Answer2b , Answer3a , Answer3b ,
```

#### User Interface

```cpp
void interface(void);
```
Interactive interface for end users (non-engineers). User can navigate the tree answering yes/no questions:
- `t` - True (move right)
- `f` - False (move left)
- `b` - Back to parent
- `r` - Back to root
- `q` - Quit

#### Utility Methods

```cpp
void point_to_root(void);
```
Resets `traverser` to the tree's root node.

```cpp
bool is_left(void);
```
Checks if `traverser` is a left child of its parent.

```cpp
void promt_question(void);
```
Prompts user to enter a new question/answer and adds it to the tree at the `traverser`'s position.

### tree_list Methods

#### Construction & Destruction

```cpp
tree_list::tree_list();
```
Creates empty list with `tail` and `traverser` set to NULL.

```cpp
~tree_list();
```
Destructor. Iterates through circular list, deletes all trees, then list nodes. Prints tree names during deletion.

#### Tree Management

```cpp
void add(void);
```
Creates and adds a new tree to the list. Prompts for tree name. Inserts after tail (maintains circular structure).

```cpp
void del_list_node(void);
```
Deletes current tree node. Handles special case of single-node list. Prints tree name during deletion.

#### Navigation

```cpp
bool move(void);
```
Interactive list navigation. User can:
- `a` - Previous tree
- `d` - Next tree
- `s` - Show all tree names
- `q` - Quit
- `Enter` - Select current tree

Returns `true` if tree selected, `false` if quit.

#### Search

```cpp
bool search(void);
```
Main search interface:
1. Search by text (substring matching in tree names)
2. Manual navigation

Returns `true` if tree found and selected.

```cpp
bool search_by_text(void);
```
Searches tree list for substring match in tree names. First match wins.

#### Display

```cpp
void print(void);
```
Displays all trees in circular list format:
```
Tree1 -> Tree2 -> Tree3 -> Tree1 ...
```

#### Utility

```cpp
string get_traverser(void);
```
Returns the name of the tree that `traverser` currently points to.

#### User Interfaces

```cpp
void user_interface(void);
```
Restricted interface for end users. Options:
1. Interface with decision tree
2. Load a tree
3. Move through tree
4. Search for node
5. Show tree
6. Quit

```cpp
void engineer_interface(void);
```
Full-featured interface for engineers:
1. Insert multiple questions/answers
2. Insert single question/answer
3. Save tree
4. Load tree
5. Update questions/answers
6. Delete questions/answers
7. Test user interface
8. Move through tree
9. Search for node
10. Show tree
11. Switch subtrees
12. Switch answers/questions
13. Quit

### Helper Functions

```cpp
bool getInput(int &choice);
bool getInput(char &choice);
```
Overloaded input functions with error handling. Return `true` if valid input received, `false` if error occurs (type mismatch). Clears input buffer on failure.

```cpp
bool is_related(string &user_input, string &tree_node_text);
```
Checks if `user_input` is a substring of `tree_node_text`. Used for text-based searching.

```cpp
void del_tree(node *ptr);
```
Recursive tree deletion. Post-order traversal ensures children are deleted before parents.

```cpp
void clear_file(const std::string &filename);
```
Clears file content by opening in write mode.

```cpp
bool openFileForWriting(const std::string &filePath);
bool openFileForReading(const std::string &filePath);
```
File validation functions. Return `true` if file can be opened, `false` otherwise.

```cpp
bool is_null_queue(queue<node*> &q);
```
Checks if queue contains only NULL pointers. Used during breadth-first saving to detect tree completion.

---

## Usage Guide

### Main Program Flow

```cpp
// main.cpp
int main() {
    tree_list tree;
    // Loop: Choose Engineer Mode, User Mode, or Quit
    // Engineer Mode: Create/edit trees
    // User Mode: Interact with trees
}
```

### Engineer Workflow

1. **Create Tree**
   - Engineer Mode → Add a new tree → Enter tree name
   - Construct tree interactively
   - Add questions and associate true/false answers

2. **Edit Tree**
   - Engineer Mode → Edit a tree → Search for tree
   - Use 13 available operations (insert, update, delete, save, etc.)

3. **Save Tree**
   - Engineer Interface → Option 3 (Save the tree)
   - Enter file path
   - Tree saved in breadth-first format

4. **Load Tree**
   - Engineer Interface → Option 4 (Load a tree from a file)
   - Enter file path
   - Previous tree (if any) can be saved first

### User Workflow

1. **Access Tree**
   - User Mode → Add a new tree OR Search for tree
   - Select tree from list

2. **Navigate Tree**
   - User Interface presents current question
   - Enter `t` for true, `f` for false
   - Navigate to leaf node (answer)

3. **Options During Navigation**
   - `b` - Go back to parent
   - `r` - Return to root
   - `q` - Quit

### Binary Path Notation

When searching by pattern in a decision tree:
- `0` = Go left (false answer)
- `1` = Go right (true answer)
- Example: `"0101"` means left-right-left-right traversal

---

## File Organization

| File | Purpose |
|------|---------|
| `decision_tree.h` | Decision tree class declaration |
| `decision_tree.cpp` | Decision tree implementation (968 lines) |
| `tree_list.h` | Tree list class declaration |
| `tree_list.cpp` | Tree list implementation (322 lines) |
| `main.cpp` | Program entry point and main interfaces |

### Compilation

```bash
g++ -o decision_tree main.cpp decision_tree.cpp tree_list.cpp
```

### Dependencies

- Standard C++ library: `<iostream>`, `<string>`, `<queue>`, `<fstream>`
- Platform-specific: `<conio.h>` (Windows) for `getch()` function

---

## Technical Notes

### Memory Management

- Dynamic allocation for all tree nodes and list nodes
- Recursive deletion in post-order traversal
- Destructor prompts for save before cleanup

### File Format

Breadth-first tree serialization with filling strings:
```
Root Question
Left Child Question / Filling String
Right Child Question / Filling String
...
```

Filling strings ("This is a filling slot and won't be put into the tree") preserve tree structure for incomplete trees.

### Search Complexity

| Operation | Time Complexity |
|-----------|-----------------|
| Search by text | O(N) |
| Search by pattern | O(log₂ N) average |
| Insert | O(1) if location known |
| Delete | O(N) |
| Print | O(N) |

### Error Handling

- Input validation with `getInput()` functions
- File operation verification
- Circular list integrity checks
- Null pointer validation before dereferencing
