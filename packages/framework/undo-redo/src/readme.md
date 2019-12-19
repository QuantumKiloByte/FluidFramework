# Undo Redo

This package provides and implmentation of an in-memory undo redo stack, as well as handlers for the SharedMap, and SharedSegementSequence distributed datastructures.

## Undo Redo Stack Manager

The undo redo stack manager is where undo and redo commands are issued, and it holds the stack of all undoable and redoable operations. The undo redo stack manager is a stack of stacks.

The outer stack contains operations, and the inner stack contains all the IRevertable objects that make up that operation. This allows the consumer of the undo redo stack manager to determine the granularity of what is undone or redone.

For instance, you could defined a text operation at the word level, so as a user types you could close the current operation whenever the user types a space. By doing this when the user issues an undo mid-word the characters typed since the last space would be undone, if they issue another undo the previous word would them be undone.

As mentioned above operations are a stack of IReverable objects. As suggested by the name, these objects have the ability to revert some change which ususally means two things. They must be able to track what was changed, and store enough metadata to revert that change.

In order to create IRevertable object there are provided undo redo handlers for commonly used data structures.

## Shared Map Undo Redo Handler

The SharedMapUndoRedoHandler generates IRevertable objects, SharedMapRevertable for all local changes made to a SharedMap and pushes them to the current operation on the undo redo stack. These objects are created via the valueChanged event of the SharedMap. This handler will never close the current operation on the stack. This is a fairly simple handler, and a good example to look at for understanding how IRevertable objects should work.

## Shared Segment Sequence Undo Redo Handler

The SharedSegementSequenceUndoRedoHandler generates IRevertable objects, SharedSegementSequenceRevertable for any SharedSegementSequence based distributed datastructure like: SharedString, SharedObjectSequence, and SharedNumberSequence.

This handler pushes an SharedSegementSequenceRevertable for every local Insert, Remove, and Annotate operations made to the sequence. The objects are created via the sequenceDelta event of the sequence. Like the SharedMapUndoRedoHandler this handler will never close the current operation on the stack.

This handler is more complex than the SharedMapUndoRedoHandler. The handler itself batches the SharedSegementSequence changes into the smallest number of IRevertable objects it can to minimize the memory and performance overhead on the SharedSegementSequence of tracking changes for revert.

### Shared Segment Sequence Revertable

The SharedSegementSequenceRevertable does the heavy lifting of tracking and reverting changes on the underlying SharedSegementSequence. This is accomplished via TrackingGroup objects. A TrackingGroup creates a bi-direction link between itself and the segment. This link is maintained across segment movement, splits, merges, and removal. When a sequence delta event is fired the segments contained in that event are added to a TrackingGroup. The TrackingGroup is then tracked along with additional metadata, like the delta type and the annotate property changes. From the TrackingGroup's segments we can find the ranges in the current document that were affected by the original change even in the presesene of other changes. The segments also contain the content which can be used. With the ranges, content, and metadata we can revert the original change on the sequence.

As called out above, there is some memory and performance overhead associtated with undo redo. This overhead is from the TrackingGroup. This overhead manifests in a few ways:
- Removed segments in a TrackingGroup will not be garbage collected from the backing tree structure.
- Segments can only be merged if they have all the same TrackingGroups.

This object minimizes the number of TrackingGroups created, so this overhead is very low. This undo redo infrastuture is entirely in-memory so it does not affect other users or sessions. If custom IRevertable objects use TrackingGroups this overhead should be kept in mind to avoid possible performance issues.