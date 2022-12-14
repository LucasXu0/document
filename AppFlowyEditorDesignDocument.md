# AppFlowy Editor Design Document

> This document describes the technical design of the AppFlowy Editor.

[AppFlowy](https://github.com/AppFlowy-IO/AppFlowy) is an open-source alternative to Notion built with [Rust](https://www.rust-lang.org/) and [Flutter](https://flutter.dev/). 

An *editor* is a core component in AppFlowy. In many scenarios, such as Document, Grid, and Board, the current editor component is unable to support certain business requirements. 

We have therefore been seeking an editor that will support all of AppFlowy's use cases and we have concluded that we need to design and develop our own editor for AppFlowy that we will call the **AppFlowy Editor**.

## Issues With the Editor Component

In the previous version of AppFlowy, we used [flutter_quill](https://pub.dev/packages/flutter_quill) as our editor component. 

In the process of using this library, we have encountered problems with extensibility, consistency, and code coverage. 

Although these problems may be addressed in future versions of flutter_quill, we do not wish to rely on updates this component in order to move forward with our development.

### Issues with Extensibility

We have encountered difficulty with quickly extending new *components* (aka plug-ins) and *shortcuts*. 

When it comes to components, an example issue is our requirement to insert Grid and Board into existing documents. We have defined a data structure for our new AppFlowy Editor that simplifies this process. We only need to define a new node with a new type and define a corresponding `NodeWidgetBuilder` to render these components in the AppFlowy Editor.

We also need additional *shortcuts* extensibility, such as markdown syntax support and shortcuts for key combinations (`meta + shift + option + key`).  The new AppFlowy Editor supports [customizing more shortcuts](https://github.com/AppFlowy-IO/AppFlowy/blob/main/frontend/app_flowy/packages/appflowy_editor/documentation/customizing.md#customize-a-shortcut-event).

### Issues With Self-Consistent Production Process

We have been unable to support self-consistent context production processes, such as inserting new components via slash command or floating toolbar.

The new AppFlowy Editor supports customizing toolbar items and slash menus.

### Issues with Code Coverage and Stability

The previous editor component lacked stability and sufficient code coverage.

To date, the code coverage of the AppFlowy Editor is stable at 79 to 80%. Meanwhile, we try to make sure to fix known issues and add new test cases to cover them.

## Replacement Approach

We have been actively looking for alternatives in the open-source community, such as [super_editor](https://pub.dev/packages/super_editor). 

During our research, we found that super_editor allows for extending new components in a way that can also support customized shortcuts. 

However, the underlying data structure of super_editor is a list that does not support nesting. We feel this data structure is not appropriate for nodes with parent-child relationships. For example, in the case of multi-level lists the form of each level is inconsistent. 

Other important considerations that have factored into our search is our need for an editor that is highly customizable and can keep up with the times such that it can continue to support the evolving functionality of AppFlowy.

To date, we still haven't found a solution that suits our needs.

For the above reasons, we have decided to design and develop the new AppFlowy Editor component ourselves.

---

## Solution Overview

Before starting a new editor project, we'll examine some existing editor implementations. There are not many editor projects based on Flutter, so we'll refer to well-known front-end editor implementations, such as Quill.js and Slate.js.

We believe that the foundation of the editor lies in *the design of the data structure*. 

**Quill.js** uses [Delta](https://quilljs.com/docs/delta/) as the data structure, while **Slate.js** uses tree nodes as the data structure. Ultimately we have elected to use a tree node like Slate.js to assemble the documents, while continuing to use Delta for the data storage of text nodes.

### Why Use a Combination of Node Tree and Delta?

Why do we use a node tree?
 - The entirety of the document data is described using a single Delta data which does not allow us to easily describe complex nested scenarios.
  - When there is an issue with a paragraph or document, restoring the document becomes relatively difficult.

So our preference is to use a node tree like `Slate.js` to describe the document in chunks, where each chunk’s additions, deletions, and modifications only affect the changes to the current node.

Why do we still use Delta for the text node?
 - If text with different styles continues to be split into different nodes, it will increase the complexity of the tree node structure.
 - The ability to export a text change delta is already supported in Flutter, so it is easy to substitute the Flutter text change delta to `Delta`.
 - Considering that our previous version is using flutter-quill as the editor component, it is simpler to keep Delta for text nodes in doing a data migration.

### Code Example 
The following JSON will be used to describe the above-combined data structure.

![Combined Data Structure Example](https://raw.githubusercontent.com/LucasXu0/document/main/AppFlowyEditorDesignDocument/0.png)

- For the text node (with a type equal to `text`), the editor will use Delta to store the data.
- For the others (non-text nodes), the editor will use Attributes to store the data.

---

## Detailed Design for AppFlowy Editor

We will state the design of AppFlowy Editor through the following three aspects.

1. What is the data made of? (keywords: Node, Delta, Document)
2. How to update the data? (keywords: Position, Path, Operation, Transaction, EditorState, Apply)
3. How to render widgets through the data? (keywords: Render Plugins)

### Editor Data Structure

AppFlowy Editor treats a document as a collection of nodes. For example, a paragraph is a `TextNode` and an image is an `ImageNode`.

We use `LinkedList` to organize the relationship between nodes, which provides a relatively efficient way to insert and delete nodes.

Each node uses a normalized description, so we can easily describe those nodes in JSON.

#### Required Node Fields
A node must contain the fields listed below.

##### Type
The `Type` field is used to find the renderer and control how to serialize and deserialize the current node

##### Attributes
The `Atttributes` field indicates what data should be presented and synced. An `ImageNode`, for example, uses the `image_src` in its attributes to describe the link where to load the image.

##### Children
The `Children` field indicates the children nodes, such as the embedded bulleted list or the block in the table component.

##### Delta
The [Delta](https://quilljs.com/docs/delta/) field will only be used for instances of `TextNode`.

As mentioned above, AppFlowy Editor will use Delta to describe the information of the text node, which is not repeated here.

It should be noted that certain styles are described using `Attributes` instead of `Delta`. Rather than make them a part of the text, we treat these styles are descriptions of paragraphs. These styles include headings, references, lists of text nodes, as well as the overall paragraph style.

### Example Node Definitions

Below is the definition of a `Node` in Dart.

```dart
class Node extends ChangeNotifier with LinkedListEntry<Node> {
  Node({
    required this.type,
    Attributes? attributes,
    this.parent,
    LinkedList<Node>? children,
  })
}
```

While this is an example definition of a `TextNode` in Dart.

```dart
class TextNode extends Node {
  TextNode({
    required Delta delta,
    LinkedList<Node>? children,
    Attributes? attributes,
  })
}
```

#### Image and Text Node Example

In the following figure, there is an image node and a text node in the document.

![Image and Text Node Example](https://raw.githubusercontent.com/LucasXu0/document/main/AppFlowyEditorDesignDocument/1.png)

The JSON representation of `ImageNode`'s data is

```json
{
    "type": "image",
    "attributes": {
        "image_src": "https://i.ibb.co/WKQwVDn/Xnip2022-09-02-15-49-51.jpg",
        "align": "left",
        "width": 285
    }
}
```

And the JSON representation of `TextNode`'s data is

```json
{
    "type": "text",
    "attributes": { "subtype": "heading", "heading": "h1" },
    "delta": [{ "insert": "🌟 Welcome to AppFlowy!" }]
}
```

#### Unordered List Example

In the following figure you can see an example of an embedded unordered list in the document

![Unordered List Example](https://raw.githubusercontent.com/LucasXu0/document/main/AppFlowyEditorDesignDocument/2.png)

And the JSON representation for the document is

```json
{
  "document": {
    "type": "editor",
    "children": [
      {
        "type": "text",
        "attributes": { "subtype": "heading", "heading": "h3" },
        "delta": [{ "insert": "Bulleted List" }]
      },
      {
        "type": "text",
        "children": [
          {
            "type": "text",
            "attributes": { "subtype": "bulleted-list" },
            "delta": [{ "insert": "A1" }]
          },
          {
            "type": "text",
            "attributes": { "subtype": "bulleted-list" },
            "delta": [{ "insert": "A2" }]
          }
        ],
        "attributes": { "subtype": "bulleted-list" },
        "delta": [{ "insert": "A" }]
      }
    ]
  }
}
```

### Updating Data in the Editor

Before we update the data, we must know which part of the data need to be updated. In other words, we need to locate the position of a node.

#### Locating Nodes

Nodes may be located in a variety of manners including:
 * [Path](#path)
 * [Position](#position)
 * [Selection](#selection)

##### Path

AppFlowy Editor uses `Path` to locate the position of a node. Path is an integer array consisting of its position in its ancestor's node and the position of its ancestors. All data change operations are performed based on the Path.

```dart
typedef Path = List<int>;
```

There is an example below.

![Nexted List Path Example](https://raw.githubusercontent.com/LucasXu0/document/main/AppFlowyEditorDesignDocument/3.png)

The path of the first node A is `[0]`, then the path of the next node A1 is `[0, 0]`, and so on ...

##### Position

AppFlowy Editor uses position to locate the offset of a node. It consists of a path and an offset.

```dart
class Position {
  final Path path;
  final int offset;
}
```

Position is usually used for text editing and cursor locating. For example, if we need to locate a caret in the middle of A and 1 in node A1, then the Position is

```dart
Position(path: [0, 0], offset: 1)
```

##### Selection

AppFlowy Editor uses `Selection` to represent the range of the selection.

The cursor is also a special kind of selection, except that start and end coincide. It consists of two Positions.

```dart
class Selection {
	final Position start;
  final Position end;
}
```

For example, We need to locate the selection range as shown below.

![Selecting a Range](https://raw.githubusercontent.com/LucasXu0/document/main/AppFlowyEditorDesignDocument/4.png)

Then the selection is:

```dart
Selection(
	start: Position(path: [1], offset: 0), 
	end: Position(path:[3], offset: 1)
)
```

Note that selection is directional.

For example, in the case of top-down selection, the selection is

```dart
Selection(
	start: Position(path: [1], offset: 0), 
	end: Position(path:[3], offset: 1),
)
```

And the down-top selection is

```dart
Selection(
	start: Position(path:[3], offset: 1),
	end: Position(path: [1], offset: 0), 
)
```

#### Operation Types

AppFlowy Editor uses `Operation` objects to manipulate the document data instead of changing the node data directly. All changes to the document are triggered by an `Operation`.

The operations defined in AppFlowy Editor include

- [Insert](#insert)
- [Delete](#delete)
- [Update](#update)
- [UpdateText](#updatetext)

Each operation has a corresponding reverse operation that is applied to undo and redo.

##### Insert

`Insert` represents inserting a list of nodes into the document at a given path. Its reverse operation is `Delete`.

```dart
class InsertOperation extends Operation {
	final Path path;
  final Iterable<Node> nodes;
}
```

Take node A1 in the above figure as an example. Inserting a node with the style Bulleted List under the node A1, then the operation is

```json
{
   "op":"insert",
   "path":[0, 1],
   "nodes":[
      {
         "type":"text",
         "attributes":{"subtype":"bulleted-list"},
         "delta":[]
      }
   ]
}
```

##### Delete

`Delete` represents deleting a list of nodes into the document at a given path. Its reverse operation is `Insert`.

```dart
class DeleteOperation extends Operation {
  final Path path;
  final Iterable<Node> nodes;
}
```

Take the node D in the above figure as an example. Deleting the node D, then the operation is

```json
{
   "op":"delete",
   "path":[3],
   "nodes":[
      {
         "type":"text",
         "delta":[]
      }
   ]
}
```

In addition, the node data assigned in the delete operation is for the logic of recovery.

##### Update

`Update` represents updating a node’s attributes at the given path. Its reverse operation is itself.

```dart
class UpdateOperation extends Operation {
  final Path path;
	final Attributes attributes;
  final Attributes oldAttributes;
}
```

Take the node C in the above figure as an example. Converting the type of the node C from numbered list to bulleted list, then the operation is

```json
{
   "op":"update",
   "path":[2],
   "attributes":{"subtype":"bulleted-list"},
   "oldAttributes":{"subtype":"number-list", "number":1}
}
```

##### UpdateText

`UpdateText` represents updating text delta in the text node, which is consistent with the `Delta` logic.

For more information, see: [https://github.com/quilljs/delta](https://github.com/quilljs/delta)

#### Transactions

The AppFlowy Editor uses a `Transaction` to describe an set of changes to the document which must be treated as atomic.  It consists of a collection of `Operation`s and changes to the selection before and after.

```dart
class Transaction {
  final List<Operation> operations = [];
  Selection? afterSelection;
  Selection? beforeSelection;
}
```

The purpose of using `transaction` is to apply a collection of sequential operations that cannot be split apart. For example, in the following case:

![Transaction Example 1](https://raw.githubusercontent.com/LucasXu0/document/main/AppFlowyEditorDesignDocument/5.png)

![Transaction Example 2](https://raw.githubusercontent.com/LucasXu0/document/main/AppFlowyEditorDesignDocument/6.png)

Pressing the enter key in front of `AppFlowy!` will actually produce two consecutive operations.

- operation 1. Insert a new `TextNode` at path `[1]`, and set the delta to insert `AppFlowy!`
- operation 2. Delete `AppFlowy!` at path `[0]`.

It can be described in JSON

```dart
{
   "operations":[
      {
         "op":"insert",
         "path":[1],
         "nodes":[{"type":"text","delta":[{"insert":"AppFlowy!"},]}]
      },
      {
         "op":"update_text",
         "path":[0],
         "delta":[{"retain":11},{"delete":9}],
         "inverted":[{"retain":11},{"insert":"AppFlowy!"}]
      }
   ],
   "after_selection":{
			"start":{"path":[1],"offset":0},
      "end":{"path":[1],"offset":0}
   },
   "before_selection":{
			"start":{"path":[0],"offset":11},
      "end":{"path":[0],"offset":11}
   }
}
```

#### EditorState and Apply

`EditorState` is responsible for managing the state of the document. It holds the `Document`, and updates the document data through the `apply` function given a `Transaction`.

```dart
class EditorState {
	void apply(Transaction transaction);
}
```

#### Summary of How Data Changes

1. `EditorState` holds the `Document`, and `Document` is a collection of `Node` objects.
2. The end-user manipulates a `Node` to generate a `Selection` and `Operations`, which form a `Transaction`.
3. Apply `Transaction` to `EditorState` to refresh the `Document`.

![Editor Comonponent Diagram](https://raw.githubusercontent.com/LucasXu0/document/main/AppFlowyEditorDesignDocument/7.png)

### Rendering Widgets Using the Data

`NodeWidgetBuilder` is an abstract protocol, responsible for converting a `Node` to a `Widget`. 

```dart
typedef NodeWidgetBuilders = Map<String, NodeWidgetBuilder>;

typedef NodeValidator<T extends Node> = bool Function(T node);

abstract class NodeWidgetBuilder<T extends Node> {
  NodeValidator get nodeValidator;

  Widget build(NodeWidgetContext<T> context);
}
```

Each node owns its corresponding `NodeWidgetBuilder`.

Before initializing AppFlowy Editor, we need to inject the mapping relationship between `Node` and `NodeWidgetBuilder`.

For now, AppFlowy Editor’s built-in `NodeWidgetBuilder` includes the following

```dart
NodeWidgetBuilders defaultBuilders = {
  'editor': EditorEntryWidgetBuilder(),
  'text': RichTextNodeWidgetBuilder(),
  'text/checkbox': CheckboxNodeWidgetBuilder(),
  'text/heading': HeadingTextNodeWidgetBuilder(),
  'text/bulleted-list': BulletedListTextNodeWidgetBuilder(),
  'text/number-list': NumberListTextNodeWidgetBuilder(),
  'text/quote': QuotedTextNodeWidgetBuilder(),
  'image': ImageNodeBuilder(),
};
```

When AppFlowy Editor starts to render the `Node`s, it will first recursively traverse the `Document`.

For each `Node` it encounters, the editor will find the corresponding `NodeWidgetBuilder` from the mapping relationship according to the nodes’ type and then call the `build` function to generate a `Widget`.

![Node Builder Widget Infrastructure](https://raw.githubusercontent.com/LucasXu0/document/main/AppFlowyEditorDesignDocument/8.png)

Meanwhile, each `NodeWidgetBuilder` is bound to `Node` through `ChangeNotifierProvider`. Combined with the above-mentioned logic of `Document` data change, whenever the data of a certain node changes, AppFlowy Editor will notify `NodeWidgetBuilder` to refresh in real-time.


## Questionnaire

* Which part of the editor do you want to know more about?
* Do you have more suggestions for our design?

…
