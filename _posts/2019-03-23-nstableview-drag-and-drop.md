---
layout: post
title: Using Drag and Drop with NSTableView
tags: [macOS, Tutorial]
thumbnail: assets/img/blog/2019-03-23-nstableview-drag-and-drop/fruits-hero.jpg
---

I recently tried to implement drag and drop with `NSTableView` in a project and ran into a bunch of issues with finding resources that actually helped. Such is the life of a Cocoa developer. So here's my attempt at the _definitive guide_ to drag and drop with `NSTableView`. (This should also apply to `NSOutlineView`, since they work in a similar way.)
<!-- break -->

![Fruits App screenshot]({{ site.baseurl }}/assets/img/blog/2019-03-23-nstableview-drag-and-drop/fruits-app.png)
In this tutorial, you'll make an app that has two table views, each containing a list of fruits. By the end of the tutorial, you'll be able to:
* drag a fruit name to the other table view to copy it over
* drag and drop to reorder the items within a table view
* drag the fruit names into other apps

To get started, download the template Xcode project [here](https://github.com/thompsonate/NSTableView-Drag-and-Drop/archive/template.zip). You can find the completed tutorial [here](https://github.com/thompsonate/NSTableView-Drag-and-Drop).

<br>
## Table of Contents
---
<ul style="list-style-type: none;">
  <li>{% include link.html name="🍎 Introduction" %}</li>
  <li>{% include link.html name="🍑 Writing to the Pasteboard" %}</li>
  <li>{% include link.html name="🍊Dropping on a Table View to Insert" %}</li>
  <li>{% include link.html name="🥭 Creating a Custom Pasteboard Type" %}</li>
  <li>{% include link.html name="🍓 Writing Multiple Types to the Pasteboard" %}</li>
  <li>{% include link.html name="🍇 Drag and Drop to Reorder Rows" %}</li>
  <li>{% include link.html name="🍍 Dropping Onto the Entire Table View" %}</li>
  <li>{% include link.html name="🥝 Dragging to Trash to Delete" %}</li>
</ul>


{% include section.html name="🍎 Introduction" %}
---
Drag and drop is implemented with a dragging pasteboard. When a drag starts, you write to the pasteboard. When a drag ends, you can read data from the pasteboard. `NSTableView` and `NSOutlineView` have delegate methods that make it (relatively) easy to deal with drag and drop for their rows. 


{% include section.html name="🍑 Writing to the Pasteboard" %}
---
The first delegate method to implement is `tableView(_:pasteboardWriterForRow:)` in `LeftTableViewController`. The function has a return type that conforms to the procotol `NSPasteboardWriting`. `NSString` does, so you can just cast the `String` to `NSString` and return that.
```swift
func tableView(
    _ tableView: NSTableView, 
    pasteboardWriterForRow row: Int)
    -> NSPasteboardWriting?
{
    return FruitManager.leftFruits[row] as NSString
}
```
> For more information on `NSPasteboardWriting` and what Cocoa classes conform to it, check out the [documentation](https://developer.apple.com/documentation/appkit/nspasteboardwriting).

Returning a non-nil value from this function will make the cell draggable. But you can't drop it on anything just yet. In order to be able to drag into other apps, the drag's operation mask needs to be set to work outside the app.

Add a line to `viewDidLoad()` which sets the default operation mask for the table view to `copy` for destinations outside the app:
```swift
tableView.setDraggingSourceOperationMask(.copy, forLocal: false)
```
You can now drag from the left table view into other apps! Try dragging cells into TextEdit or Messages.


{% include section.html name="🍊Dropping on a Table View to Insert" %}
---
Now you need to set up the right table view to accept drops on it. First, add this line to `viewDidLoad()` in `RightTableViewController`:
```swift
tableView.registerForDraggedTypes([.string])
```
This registers the table view to accept drags containing `string` types. Since you returned an `NSString` in the pasteboard writer of the left table view, the pasteboard type is `string`. Simple enough, right?

<br>
### Validate the Drop
Next, implement `tableView(_:validateDrop:proposedRow:proposedDropOperation:)`. This is called when a drag is hovering over the table view, before it has been dropped or canceled.

In this function, you can specify how to respond to a proposed drop operation, which can be either on or above a cell. In this table view, the user will be allowed to drop items between cells to insert them. That means we want to allow proposed drops `above` but not respond to proposed drops `on`.
```swift
func tableView(
    _ tableView: NSTableView,
    validateDrop info: NSDraggingInfo,
    proposedRow row: Int,
    proposedDropOperation dropOperation: NSTableView.DropOperation)
    -> NSDragOperation
{
    if dropOperation == .above {
        return .move
    }
    return []
}
```

<br>
### Accept the Drop
The next function to implement is `tableView(_:acceptDrop:row:dropOperation:)`. This is where you'll handle the data from the dragging pasteboard after it's been dropped on the table view.

First, get the array of pasteboard items from the drop. Because table views support multiple selection, multiple rows could be dropped simultaneously. Each dragged row corresponds to a pasteboard item, with each item containing the string you set for the row.

For each pasteboard item, get the string stored in it with the method `NSPasteboardItem.string(forType: NSPasteboard.PasteboardType)`.
```swift
func tableView(
    _ tableView: NSTableView,
    acceptDrop info: NSDraggingInfo,
    row: Int,
    dropOperation: NSTableView.DropOperation)
    -> Bool
{
    guard let items = info.draggingPasteboard.pasteboardItems
        else { return false }
    
    let fruits = items.compactMap{ $0.string(forType: .string) }
    FruitManager.rightFruits.insert(contentsOf: fruits, at: row)
    tableView.insertRows(at: IndexSet(row...row + newFruits.count - 1),
                         withAnimation: .slideDown)
    return true
}
```

Now try dragging from the left table view to the right table view!
<video autoplay loop playsinline muted src="{{ site.baseurl }}/assets/img/blog/2019-03-23-nstableview-drag-and-drop/insert.mp4" type="video/mp4"></video>

> Bonus: since it's registered for the `string` pasteboard type, you can highlight some text in another app and drag that into the right table view.

<br>
<video autoplay loop playsinline muted src="{{ site.baseurl }}/assets/img/blog/2019-03-23-nstableview-drag-and-drop/dragfest.mp4" type="video/mp4"></video>
<p style="text-align: center">It's a drag fest!</p>


{% include section.html name="🥭 Creating a Custom Pasteboard Type" %}
---
Create an extension of `NSPasteboard.PasteboardType`. This is a UTI string and should be a unique identifier.
```swift
extension NSPasteboard.PasteboardType {
    static let tableViewIndex = NSPasteboard.PasteboardType("io.natethompson.tableViewIndex")
}
```


{% include section.html name="🍓 Writing Multiple Types to the Pasteboard" %}
---
In `PasteboardUtil.swift` there's a class `FruitPasteboardWriter`. Make the class conform to `NSPasteboardWriting` and add the following protocol stubs:
```swift
func writableTypes(
    for pasteboard: NSPasteboard) -> [NSPasteboard.PasteboardType] 
{ }

func pasteboardPropertyList(
    forType type: NSPasteboard.PasteboardType) -> Any? 
{ }
```

In `writableTypes(for:)`, return `[.string, .tableViewIndex]`. These are the pasteboard types that this class can write to the pasteboard.

Now, in `pasteboardPropertyList(forType:)`, if the type is `string`, return the fruit name. If the type is `tableViewIndex`, return the row index. It should now look like this:
```swift
class FruitPasteboardWriter: NSObject, NSPasteboardWriting {
    var fruit: String
    var index: Int
    
    init(fruit: String, at index: Int) {
        self.fruit = fruit
        self.index = index
    }
    
    func writableTypes(
        for pasteboard: NSPasteboard)
        -> [NSPasteboard.PasteboardType] 
    {
        return [.string, .tableViewIndex]
    }

    func pasteboardPropertyList(
        forType type: NSPasteboard.PasteboardType) -> Any?
    {
        switch type {
        case .string:
            return fruit
        case .tableViewIndex:
            return index
        default:
            return nil
        }
    }
}
```
When an instance of this object is written to the pasteboard, you can get the value for either (or both) type, depending on what you want to do. We'll get to that next.


{% include section.html name="🍇 Drag and Drop to Reorder Rows" %}
---
Currently, if you drag a cell from the right table view into the right table view, it'll create a duplicate at the position. Let's change that so you can reorder the cells without duplicating.

<video autoplay loop playsinline muted src="{{ site.baseurl }}/assets/img/blog/2019-03-23-nstableview-drag-and-drop/reorder.mp4" type="video/mp4"></video>
<br>

First, register the right table view for the dragged type you created. Modify the line you wrote in `viewDidLoad()` earlier to:
```swift
tableView.registerForDraggedTypes([.string, .tableViewIndex])
```

Next, implement `tableView(_:pasteboardWriterForRow:)` in `RightTableViewController` using the `FruitPasteboardWriter` you just made:
```swift
func tableView(
    _ tableView: NSTableView,
    pasteboardWriterForRow row: Int)
    -> NSPasteboardWriting?
{
    return FruitPasteboardWriter(fruit: FruitManager.rightFruits[row], at: row)
}
```
<br>
### Destination Feedback Style
Then, you'll want to rewrite `tableView(_:validateDrop:proposedRow:proposedDropOperation:)` and set the `draggingDestinationFeedbackStyle` based on the dragging source. The `gap` style works well when reordering rows in the table view. Otherwise, we'll keep using `regular`, which draws the insertion line if the drop operation is `above`.
```swift
func tableView(
    _ tableView: NSTableView,
    validateDrop info: NSDraggingInfo,
    proposedRow row: Int,
    proposedDropOperation dropOperation: NSTableView.DropOperation)
    -> NSDragOperation
{
    guard let source = info.draggingSource as? NSTableView,
        dropOperation == .above
        else { return [] }

    if source === tableView {
        tableView.draggingDestinationFeedbackStyle = .gap
    } else {
        tableView.draggingDestinationFeedbackStyle = .regular
    }
    return .move
}
```
> There's a bug in `NSTableView` that requires implementing `tableView(_:heightOfRow:)` to get the `gap` style to animate correctly.

<br>
### Move Rows
Now you just need to get the table view indexes when the drag is dropped. Modify `tableView(_:acceptDrop:row:dropOperation)` so it looks like this:
```swift
func tableView(
    _ tableView: NSTableView,
    acceptDrop info: NSDraggingInfo,
    row: Int,
    dropOperation: NSTableView.DropOperation) -> Bool
{
    guard let items = info.draggingPasteboard.pasteboardItems
        else { return false }
    
    let indexes = items.compactMap{ $0.integer(forType: .tableViewIndex)
    if !indexes.isEmpty {
        FruitManager.rightFruits.move(with: IndexSet(indexes), to: row)

        // Reordering with multiple selection enabled is hard:
        // https://stackoverflow.com/a/26855499/7471873
        tableView.beginUpdates()
        var oldIndexOffset = 0
        var newIndexOffset = 0
        for oldIndex in oldIndexes {
            if oldIndex < row {
                tableView.moveRow(at: oldIndex + oldIndexOffset, to: row - 1)
                oldIndexOffset -= 1
            } else {
                tableView.moveRow(at: oldIndex, to: row + newIndexOffset)
                newIndexOffset += 1
            }
        }
        tableView.endUpdates()
        return true
    }
    
    let fruits = items.compactMap{ $0.string(forType: .string) }
    FruitManager.rightFruits.insert(contentsOf: fruits, at: row)
    tableView.insertRows(at: IndexSet(row...row + newFruits.count - 1),
                         withAnimation: .slideDown)
    return true
}
```
This implementation prioritizes `tableViewIndex` over `string`. As it's set up now, if you drag within the right table view, there will be values for `tableViewIndex` and `string`. It uese the row number values for `tableViewIndex` to rearrange the array instead of just inserting at a new index.

> Note: `NSArray.move(with:to:)` and `NSPasteboardItem.integer(forType:)` are implemented in extensions, so check for those before you copy/paste and wonder why it doesn't work.


{% include section.html name="🍍 Dropping Onto the Entire Table View" %}
---
Sometimes you might not want users to have as much control of where the data they're dropping lands. Maybe the table view is sorted or you want cells displayed in the order they were added. There's a way to make the whole table view highlight when items are dragged over it.

<video autoplay loop playsinline muted src="{{ site.baseurl }}/assets/img/blog/2019-03-23-nstableview-drag-and-drop/append.mp4" type="video/mp4"></video>
<br>

Implement `tableView(_:validateDrop:proposedRow:proposedDropOperation:)` in `LeftTableViewController`. The line that does the magic here is this:
```swift
tableView.setDropRow(-1, dropOperation: .on)
```
Passing `-1` and `on` will highlight the entire table view. You can then return `copy` to show the green + icon next to the cursor when a drag is hovering over the table view.

There's one thing left to do in this function. We don't want to allow drags from the left table view onto itself since this table view shouldn't be reordered by the user. Get the dragging source and return `[]` if the source is the left table view.
```swift
func tableView(
    _ tableView: NSTableView,
    validateDrop info: NSDraggingInfo,
    proposedRow row: Int,
    proposedDropOperation dropOperation: NSTableView.DropOperation)
    -> NSDragOperation
{
    guard let source = info.draggingSource as? NSTableView
        else { return [] }
    if source !== tableView {
        tableView.setDropRow(-1, dropOperation: .on)
        return .copy
    }
    return []
}
```
> `info.draggingSource` is `nil` when the source is in a different application. The implementation above disallows drags from other apps, sources within the same app that aren't `NSTableView`s, and drags from the left table view. 

Lastly, implement `tableView(_:acceptDrop:row:dropOperation:)` in `LeftTableViewController` and append the dropped strings to the array.
```swift
func tableView(
    _ tableView: NSTableView,
    acceptDrop info: NSDraggingInfo,
    row: Int,
    dropOperation: NSTableView.DropOperation) -> Bool
{
    guard let items = info.draggingPasteboard.pasteboardItems
        else { return false }
    
    let fruits = items.compactMap{ $0.string(forType: .string) }
    FruitManager.leftFruits.append(contentsOf: fruits)
    
    let oldCount = tableView.numberOfRows
    tableView.insertRows(at: IndexSet(oldCount...oldCount + newFruits.count - 1),
                         withAnimation: .slideDown)
    return true
}
```


{% include section.html name="🥝 Dragging to Trash to Delete" %}
---
I think dragging to Trash is kind of a gimmick unless you're writing a file manager, but it's fun and easy so here's how to do it.

<video autoplay loop playsinline muted src="{{ site.baseurl }}/assets/img/blog/2019-03-23-nstableview-drag-and-drop/delete.mp4" type="video/mp4"></video>
<br>

In `RightTableViewController`, add the following line to `viewDidLoad()`:
```swift
tableView.setDraggingSourceOperationMask([.copy, .delete], forLocal: false)
```
This is the similar to what you did earlier on the left table view, with `delete` in addition to `copy`. Setting the `delete` operation allows the drag to be dropped onto the Trash icon.

Now it's just a matter of implementing `tableView(_:draggingSession:endedAt:operation:)` and checking if the drag operation is `delete`.
```swift
func tableView(
    _ tableView: NSTableView,
    draggingSession session: NSDraggingSession,
    endedAt screenPoint: NSPoint,
    operation: NSDragOperation)
{
    if operation == .delete, 
        let items = session.draggingPasteboard.pasteboardItems
    {
        let indexes = items.compactMap {
            $0.integer(forType: .tableViewIndex)
        }
        for index in indexes.reversed() {
            FruitManager.rightFruits.remove(at: index)
        }
        tableView.removeRows(at: IndexSet(indexes), withAnimation: .slideUp)
    }
}
```

<br>
Woohoo you made it to the end! Go take a break and eat some fruit or something. I'm hungry.
