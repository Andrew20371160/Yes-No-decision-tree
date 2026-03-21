## 1) Library overview

This project implements:

- A **binary decision tree** (`decision_tree`) where:
  - each node stores a **question/answer text**,
  - each node can have a **false branch** (`left`) and **true branch** (`right`),
  - a `traverser` pointer is used for interactive navigation and editing.

- A **circular doubly-linked list of decision trees** (`tree_list`) where:
  - each list node owns a `decision_tree*`,
  - users can manage multiple named trees through a menu system.

- A CLI program (`main.cpp`) that exposes:
  - **Engineer Mode** (create/edit trees),
  - **User Mode** (interact with trees, limited editing).

---

## 2) Core data structures

### 2.1 `struct node` (decision_tree node)
Defined in `decision_tree.h`.

**Fields**
- `string text`  
  The question/answer text stored at this node.

- `int parent_answer`  
  Stores the answer value that led from the parent into this node.
  - In practice, the code uses:
    - `0` for false branch
    - `1` for true branch
    - `-1` for root/no answer (see `enum{False=0,True,no_answer=-1}` in `decision_tree.cpp`)

- `node *parent, *left, *right`  
  Pointers to parent, false subtree (left), and true subtree (right).

**Notes**
- There’s no constructor; allocation is done through `decision_tree::get_node()`.

---

### 2.2 `struct list_node` (tree_list node)
Defined in `tree_list.h`.

**Fields**
- `decision_tree *dtree`  
  Heap-allocated tree owned by this list node.

- `string name`  
  Human-readable name for the tree.

- `list_node *next, *prev`  
  Links for a **circular doubly linked list**.

---

## 3) Helper utilities (shared / global helpers)

These are not members of a class, but the whole program relies on them.

### 3.1 `bool getInput(int& choice)`
Defined in `decision_tree.h`, implemented in `decision_tree.cpp`.

**Purpose**
Safely read an integer from `cin` and prevent an infinite fail loop if the user types non-numeric input.

**Parameters**
- `int& choice`: output variable that receives the integer.

**Returns**
- `true` if an integer was read successfully.
- `false` if input failed (`cin.fail()`), in which case it clears the error and ignores the invalid line.

**Typical use**
Used in almost every menu to validate numeric input.

---

### 3.2 `bool getInput(char& choice)`
Defined in `decision_tree.h`, implemented in `decision_tree.cpp`.

**Purpose**
Same as the integer version, but reads a single character.

**Parameters**
- `char& choice`: output variable.

**Returns**
- `true` on success, `false` on failure.

---

### 3.3 `bool is_related(string& user_input, string& tree_node_text)`
Declared in `decision_tree.h`, implemented in `decision_tree.cpp`.

**Purpose**
Substring-style matching: returns true if `user_input` appears inside `tree_node_text`.

**Parameters**
- `string& user_input`: text to search for.
- `string& tree_node_text`: text to search within.

**Returns**
- `true` if `user_input` is found as a contiguous substring inside `tree_node_text`.
- `false` otherwise.

**Used by**
- `decision_tree::search_by_text()`
- `tree_list::search_by_text()` (to match tree names by partial text)

**Important behavior**
- Case-sensitive.
- Implements manual scanning; does not use `std::string::find()`.

---

### 3.4 File helpers (decision_tree.cpp local helpers)

#### `void clear_file(const std::string& filename)`
**Purpose**
Opens a file in output mode to truncate/clear it.

**Returns**
- Nothing.

**Note**
Currently not used by `save()` (because `save()` directly opens with `std::ios::trunc`).

#### `bool openFileForWriting(const std::string& filePath)`
**Purpose**
Checks whether a file can be opened for writing.

**Returns**
- `true` if file opens successfully, else `false`.

#### `bool openFileForReading(const std::string& filePath)`
**Purpose**
Checks whether a file can be opened for reading.

**Returns**
- `true` if file opens successfully, else `false`.

**Note**
Not currently used in `load()` (which opens `ifstream file(file_path)` directly), but still a helper.

---

### 3.5 Tree serialization helper

#### `bool is_null_queue(queue<node*>& q)`
Defined in `decision_tree.cpp`.

**Purpose**
When saving an incomplete tree breadth-first, null placeholders are pushed to preserve structure. This helper checks whether the queue has become “all null”, which indicates saving should stop (prevents an infinite loop from continually expanding null children).

**Parameters**
- `queue<node*>& q`: queue of node pointers used during BFS save.

**Returns**
- `true` if all entries are `NULL` (meaning no real nodes remain).
- `false` otherwise.

---

## 4) Module: `decision_tree` (binary yes/no decision tree)

### 4.1 Construction / lifetime

#### `decision_tree::decision_tree(void)`
**Purpose**
Create an empty decision tree.

**State after**
- `root = NULL`
- `traverser = NULL`

---

#### `decision_tree::decision_tree(string* arr, int size)`
**Purpose**
Build a tree from a breadth-first array representation.

**Parameters**
- `string* arr`: array of node texts in BFS order.
- `int size`: number of entries.

**Behavior**
- `arr[0]` becomes the root if non-empty.
- Empty strings (`""`) represent “no node here”, and queue `NULL`s are used internally to preserve positions.

**Notes / caveats**
- The code assigns `parent` pointers for created children.
- It does **not** set `parent_answer` meaningfully for these nodes (it calls `get_node(arr[i])` without passing an answer), so `parent_answer` defaults to `-1` for all nodes constructed this way.

---

#### `decision_tree::~decision_tree(void)`
**Purpose**
Destructor for the decision tree.

**Behavior**
- If `root` exists:
  - asks user if they want to save: `Do you want to save current tree?(y/n)`
  - if not `'n'`, calls `save()`
  - deletes the entire tree with `del_tree(root)`
  - sets `root` and `traverser` to `NULL`

**Implication**
This class is interactive even in destruction (it prompts), which is unusual for libraries; but it matches this repo’s CLI design.

---

#### `node* decision_tree::get_node(string _text, bool p_answer = -1)`
**Purpose**
Allocate and initialize a `node` on the heap.

**Parameters**
- `_text`: node text
- `p_answer`: the answer value (0/1) associated with the edge from parent to this node (default `-1`)

**Returns**
- Pointer to a fully initialized node with `parent/left/right = NULL`.

**Used by**
Most tree construction/editing operations.

---

#### `void del_tree(node* ptr)` (global)
Declared in `decision_tree.h`, implemented in `decision_tree.cpp`.

**Purpose**
Recursively delete an entire subtree (post-order).

**Parameters**
- `node* ptr`: subtree root.

**Behavior**
- Recursively deletes `left` then `right`, then deletes `ptr`.

---

### 4.2 Navigation (traverser-based movement)

#### `bool decision_tree::move(void)`
**Purpose**
Interactive “editor navigation” inside the tree using `traverser`.

**Controls**
- `w` = go to parent
- `a` = go to left child (false branch)
- `d` = go to right child (true branch)
- `s` = print whole tree
- `Enter` = stop at current node (returns success)
- `q` = quit movement (resets traverser to root, returns false)

**Returns**
- `true` if user presses Enter to select current node
- `false` if user quits (`q`) or tree is empty

**Notes**
Uses `getch()` from `<conio.h>` (Windows/console-specific).

---

#### `void decision_tree::point_to_root(void)`
**Purpose**
Reset traversal pointer.

**Effect**
- `traverser = root`

---

#### `bool decision_tree::is_left(void)`
**Purpose**
Check whether `traverser` is the left child of its parent.

**Returns**
- `true` if `traverser->parent` exists and `parent->left == traverser`, else `false`.

**Used by**
- `change_parent_child()`

---

### 4.3 Adding nodes / building the tree

#### `void decision_tree::promt_question(void)`
**Purpose**
Interactive insertion of a question/answer node.
(Spelling is “promt” in code.)

**Behavior**
- If the tree is empty (`root == NULL`):
  - prompts for root question/answer text
  - creates root node
- Otherwise:
  - asks whether the new entry is for **false(0)** or **true(1)** branch of the current `traverser`
  - if that child pointer is null, it creates it and reads its text
  - if that child pointer is already occupied, it prints “use update to update the question” and aborts

**Side effects**
- Temporarily moves `traverser` to the newly created child for text input, then sets it back to parent.

---

#### `void decision_tree::construct(void)`
**Purpose**
Interactive loop to insert *multiple* nodes/questions into the tree.

**Flow**
- Repeats “Enter new node?” until user says `n`
- If tree is empty, it creates the root first.
- Otherwise it tries to determine whether the new node is “related” to current question:
  - If “related”, calls `promt_question()`
  - If “not related”, calls `move()` to navigate to a better location; then calls `promt_question()`

**Quit**
- Inside the “related?” prompt, entering `q` exits the construct process.

---

#### `void decision_tree::insert(void)`
**Purpose**
Insert exactly one question/answer into the tree.

**Behavior**
- If tree is empty → `promt_question()` (creates root)
- Else user chooses:
  - (1) search by text → `search()` then `promt_question()`
  - (2) manually traverse → `move()` then `promt_question()`

**Post-condition**
- Resets `traverser = root`

---

### 4.4 Display

#### `void decision_tree::print(void)`
**Purpose**
Print the tree breadth-first, with each level on its own line.

**Output format**
- Prints node texts separated by `" , "`
- Ends a line when the BFS finishes a level.

**Notes**
Despite a comment about “elegantly”, it currently prints plain BFS level lines.

---

### 4.5 Searching

#### `bool decision_tree::search_by_text(string& t)`
**Purpose**
Breadth-first search for a node by matching text (substring).

**Parameters**
- `string& t`: query text.

**Returns**
- `true` if found (and sets `traverser` to the found node)
- `false` otherwise (and resets `traverser = root`)

**Complexity**
- O(N) nodes scanned.

---

#### `bool decision_tree::search_by_pattern(string& pattern)`
**Purpose**
Navigate by an answer pattern like `"01001"` where:
- `'0'` means go left (false),
- `'1'` means go right (true).

**Parameters**
- `string& pattern`: path pattern.

**Returns**
- `true` if the full path exists; sets `traverser` to destination.
- `false` if invalid character or missing child on path; resets `traverser = root`.

**Complexity**
- O(length(pattern)).

---

#### `bool decision_tree::search(void)`
**Purpose**
Top-level search interface (interactive).

**Modes**
1. search by text → calls `search_by_text`
2. search by pattern → calls `search_by_pattern`
3. manually select node → calls `move`

**Returns**
- `true` if selection found/succeeded, else `false`.

---

### 4.6 Updating / restructuring

#### `void decision_tree::update(void)`
**Purpose**
Update the `text` of a found node.

**Flow**
- Calls `search()`
- If found: reads a new line and assigns `traverser->text`
- Resets `traverser = root`

---

#### `void decision_tree::switch_nodes(void)`
**Purpose**
Swap the *left and right child pointers* of the currently selected node (swap subtrees).

**Flow**
- Calls `search()`
- Confirms with user
- Swaps:
  - `traverser->left` ↔ `traverser->right`

**Effect**
Entire subtrees are reversed.

---

#### `void decision_tree::switch_answers(void)`
**Purpose**
Swap only the *texts* of the left and right child (does not swap subtree pointers).

**Flow**
- Calls `search()`
- Confirms with user
- If both children exist:
  - `swap(traverser->left->text, traverser->right->text)`
- If one child is missing:
  - denies and suggests using `switch_nodes()`.

---

#### `void decision_tree::remove(void)`
**Purpose**
Delete a node from the tree, with options to reattach or delete its children.

**High-level behavior**
- Calls `search()` to locate the node (sets `traverser`)
- Special handling if `traverser == root`:
  - Can promote left child or right child to be new root, and then either:
    - move the other subtree elsewhere (`move_left_subtree` / `move_right_subtree`)
    - or delete it
  - Also can delete whole tree
- For non-root:
  - If node has children:
    - Option 1: delete node + both subtrees (`del_traverser_with_children`)
    - Option 2: delete node but relocate left and right subtrees elsewhere (`move_left_subtree`, `move_right_subtree`)
  - If node is a leaf: just unlink from parent and delete node

**Key helper functions used**
- `change_parent_child(node* other)`
- `del_traverser_with_children()`
- `move_left_subtree(node* pleft)`
- `move_right_subtree(node* pright)`

---

#### `void decision_tree::change_parent_child(node* other)`
**Purpose**
Replace the parent’s pointer to `traverser` with another pointer.

**Parameters**
- `node* other`: node pointer to set in place of `traverser` (often `NULL`).

**Behavior**
- If `traverser` is a left child → `parent->left = other`
- Else → `parent->right = other`

**Used by**
Deletion/relocation logic.

---

#### `void decision_tree::del_traverser_with_children(void)`
**Purpose**
Delete the node currently pointed to by `traverser` and **delete both of its subtrees**, safely unlinking from the tree.

**Behavior**
- Deletes `traverser->left` subtree and sets left to NULL
- Deletes `traverser->right` subtree and sets right to NULL
- If traverser is root:
  - deletes root and clears tree
- Else:
  - unlinks from parent, deletes traverser, resets traverser to root

---

#### `void decision_tree::move_left_subtree(node* pleft)`
**Purpose**
Relocate a left subtree root (`pleft`) to become the left child of some other node chosen via `search()`.

**Behavior**
- Interactively asks where to insert it
- Repeats until:
  - `search()` succeeds AND
  - `traverser->left == NULL`
- Attaches:
  - `traverser->left = pleft`
  - `pleft->parent = traverser`

---

#### `void decision_tree::move_right_subtree(node* pright)`
**Purpose**
Same as `move_left_subtree` but for right child.

**Attach condition**
- Requires `traverser->right == NULL`

---

#### `bool decision_tree::is_same_subtree(node* ptr)` *(implemented but commented out)*
**Purpose**
Would check whether `ptr` is an ancestor of `traverser` (i.e., whether `ptr` is in the chain from `traverser` to root).

**Status**
- Commented out in `decision_tree.cpp` and noted as “not used but implemented” in header.

---

### 4.7 Persistence (save/load)

#### `void decision_tree::save(void)`
**Purpose**
Save the tree to a file in breadth-first order.

**How it preserves structure**
- Uses a special global string:

  `filling_string = "This is a filling slot and won't be put into the tree"`

- When encountering a null node position during BFS, it writes `filling_string` and enqueues two null children to preserve shape.
- Uses `is_null_queue()` to stop once the queue is entirely nulls.

**User interaction**
- Prompts for file path (expects file already exists, but actually `ofstream(file_path)` will create it if allowed).

---

#### `void decision_tree::load(void)`
**Purpose**
Load a tree from a breadth-first saved file.

**Behavior**
- If current tree exists:
  - asks whether to save it first
  - deletes it
- Reads first line as root
- Uses BFS reconstruction:
  - for each node popped, reads two lines for left and right
  - if line equals `filling_string`, treats child as null and pushes `NULL` in queue
  - otherwise allocates a node and sets parent pointers

**Edge cases**
- Empty first line → treated as empty file/tree.

---

### 4.8 End-user interaction (tree traversal Q/A mode)

#### `void decision_tree::interface(void)`
**Purpose**
Interactive “end user” mode where user answers true/false and navigates the decision process.

**Controls**
- `t` → go to right child (true)
- `f` → go to left child (false)
- `b` → back to parent
- `r` → back to root
- `Enter` → return to caller
- `q` → quit interface (returns to caller)

---

## 5) Module: `tree_list` (multiple named trees manager)

### 5.1 Construction / lifetime

#### `tree_list::tree_list()`
**Purpose**
Initialize an empty list.

**State**
- `tail = NULL`
- `traverser = NULL`

---

#### `tree_list::~tree_list()`
**Purpose**
Destroy the list and all trees it owns.

**Behavior**
- Breaks circular link (`tail->next = NULL`)
- Iterates nodes, and for each:
  - prints tree name
  - deletes `ptr1->dtree` (which triggers decision_tree destructor prompts/save)
  - deletes the list node

---

#### `list_node* tree_list::get_list_node(string& tree_name)`
**Purpose**
Allocate a list node + allocate an empty `decision_tree`.

**Returns**
- New node where:
  - `name = tree_name`
  - `dtree = new decision_tree`
  - `next` and `prev` point to itself (single-node circular list)

---

### 5.2 List operations

#### `void tree_list::add(void)`
**Purpose**
Add a new named tree to the list.

**Behavior**
- Prompts for name
- If list empty: `tail = new_node`
- Else inserts after `tail` and updates circular links; sets `tail = new_node`
- Sets `traverser = tail`

---

#### `bool tree_list::move(void)`
**Purpose**
Interactive movement through tree names (as a way to select a tree).

**Controls**
- `a` previous
- `d` next
- `s` print list
- `Enter` select current (success)
- `q` quit (fail)

**Returns**
- `true` if selected with Enter
- `false` if quit or list empty

---

#### `void tree_list::print()`
**Purpose**
Print the names of all trees in the list, in order from `tail->next` around to start.

**Output**
`name1 -> name2 -> name3 ...`

---

#### `string tree_list::get_traverser(void)`
**Purpose**
Return the currently selected tree name.

**Returns**
- `traverser->name`

**Precondition**
- `traverser` must not be null.

---

### 5.3 Searching trees

#### `bool tree_list::search_by_text(void)`
**Purpose**
Find a tree by (substring) matching the name.

**Flow**
- Prompts for name text
- Iterates circular list from `tail->next`
- Uses `is_related(searchText, traverser->name)`

**Returns**
- `true` and leaves `traverser` at found node
- `false` and resets `traverser = tail`

---

#### `bool tree_list::search(void)`
**Purpose**
Top-level search interface.

**Options**
1. search by text → `search_by_text()`
2. search manually → `move()`
3. back → returns false

---

### 5.4 Deleting trees

#### `void tree_list::del_list_node(void)`
**Purpose**
Delete the currently selected tree/list node.

**Behavior**
- If only one node:
  - deletes `tail` (note: current code deletes the list node but does **not** delete its `dtree` in this single-node case)
  - sets `tail = NULL`, `traverser = NULL`
- Else:
  - finds previous node (by walking until `ptr->next == traverser`)
  - prints tree name
  - deletes `temp->dtree`
  - deletes `temp`

**Important note (possible bug)**
- In the single-node case (`traverser->next == traverser`) it does `delete tail;` but does **not** do `delete tail->dtree;` first. In the multi-node case, it deletes `dtree`. So the single-node delete path likely leaks the tree unless `list_node` had its own destructor (it doesn’t).

---

### 5.5 Tree interfaces (delegation to decision_tree)

#### `void tree_list::engineer_interface(void)`
**Purpose**
Privileged interface to create/modify a selected tree.

**Operations (delegates)**
1. `dtree->construct()`
2. `dtree->insert()`
3. `dtree->save()`
4. `dtree->load()`
5. `dtree->update()`
6. `dtree->remove()`
7. `dtree->interface()` (test user interface)
8. `dtree->move()`
9. `dtree->search()`
10. `dtree->print()`
11. `dtree->switch_nodes()`
12. `dtree->switch_answers()`
13. quit

---

#### `void tree_list::user_interface(void)`
**Purpose**
Restricted interface for interacting with a selected tree (less editing).

**Operations**
1. `dtree->interface()`
2. `dtree->load()`
3. `dtree->move()`
4. `dtree->search()`
5. `dtree->print()`
6. quit

---

## 6) CLI program entrypoint (`main.cpp`)

### 6.1 Menu helpers

#### `void create(tree_list& tree)`
**Purpose**
“Engineer Mode” main menu for managing multiple trees.

**Options**
1. Add new tree → `tree.add(); tree.engineer_interface();`
2. Edit existing tree → `tree.search(); tree.engineer_interface();`
3. Show list → `tree.print()`
4. Move through list → `tree.move()`
5. Delete a tree → `tree.search(); confirm; tree.del_list_node();`
6. Back to main menu

---

#### `void user(tree_list& tree)`
**Purpose**
“User Mode” main menu.

**Options**
1. Add new tree → `tree.add(); tree.user_interface();`
2. Search tree → `tree.search(); tree.user_interface();`
3. Show list → `tree.print()`
4. Move through list → `tree.move()`
5. Delete a tree → `tree.search(); confirm; tree.del_list_node();`
6. Back to main menu

---

### 6.2 `int main()`
**Purpose**
Top-level application loop.

**Menu**
1. Engineer Mode → calls `create(tree)`
2. User Mode → calls `user(tree)`
3. Quit → returns 0

---
