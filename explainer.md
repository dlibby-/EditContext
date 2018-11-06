# EditContext API Explained

## Overview

The EditContext API provides a way for web developers to create editing experiences that are accessible and integrated with the underlying platform's input modalities (e.g. touch keyboard, IME, shapewriting, etc.) without having to deal with the downsides of contenteditable regions. While contenteditable provides certain desirable functionality, such as caret and IME placement, and accessibility information, it is fundamentally a WYSIWYG editor for *HTML* content. In order to compute the underlying document model (which is rarely HTML) the DOM must be scraped and interpreted in order to derive the correct representation. On the other hand, setting up keyboard and composition events on any non-editable Element doesn't provide a fully integrated editing experience.

## Details

The EditContext API is an abstraction over a shared text input buffer that is a plain text model of the content being edited. It also has the notion of selection, expressed in absolute character positions into the buffer (where an empty selection is effectively a caret/insertion point). The EditContext will report the layout bounds of the view (as provided by the web developer) of the editable region, as well as the selection so that touch keyboards and IME's can be appropriately positioned. Having a shared buffer and selection allows for software keyboards to have context regarding the contents being edited. This enables features such as autocorrection suggestions, composition reconversion, and simplified handling of composition candidate selection.

Because the buffer and selection are stateful, updating the contents of the buffer is a cooperative process between the characters coming from the user and changes to the content that are driven by other events. The ```textupdate``` event will be fired on the EditContext when user input has resulted in characters being applied to the editable region. The event signals the fact that the software keyboard updated the text (and as such that state is reflected in the shared buffer at the time the event is fired). This can be a single character update, in the case of typical typing scenarios, or replacement of a one or more of characters in the case where the current selection spans multiple characters, or the user initiated an autocorrect action. Even though text updates are the results of the software keyboard modifying the buffer, the creator of the EditContext is ultimately responsible for keeping its underlying model up-to-date with the content that is being edited and telling the EditContext about such changes. These could get out of sync, for example, when updates to the editable content come in through other means (the backspace key is a canonical example).

Updates to the shared buffer are done via the ```textChanged()``` method on the EditContext. ```textChanged()``` accepts a range (start and end absolute positions over the underlying buffer) and the characters to insert at that range. ```textChanged()``` should be called anytime the editable contents have been updated. However, this should be avoided during the firing of ```textupdate``` as it will result in a canceled composition.

The ```selectionupdate``` event is fired when the input method wants a specific region selected, generally in response to something like IME reconversion.
```selectionChanged()``` should be called whenever the selection has changed, e.g. from Shift+Arrow or any other control.

The ```textformatupdate``` event is fired when the input method desires a specific region to be styled in a certain fashion, limited to the style properties that correspond with the properties that are exposed on TextFormatUpdateEvent (e.g. backgroundColor, textDecoration, etc.). The consumer of the EditContext should update their view accordingly to provide the user with visual feedback as prescribed by the software keyboard.

```compositionstart``` and ```compositioncompleted``` fire, when IME composition begins and ends. It does not provide any other contextual information, as the ```textupdating``` events will let the application know the text that the user wished to insert.

There can be multiple EditContext's per document, and they each have a notion of focused state. Because there is no implicit representation of the EditContext in the HTML view, focus must be managed by the web developer, most likely by forwarding focus calls from the DOM element that contains the editable view. ```focus``` and ```blur``` events are fired on the EditContext in reponse to changes in the focused state.


## Example usage

```javascript

// User defined class that contains the underlying model for the editable content
class EditModel {
    constructor(editContext) {
        // This specific model uses the underlying buffer directly so doesn't
        // store model directly.
        this.editContext = editContext;
    }

    updateText(text, updateRange, newSelection) {
        // No action needed, since we're directly using the underlying buffer
        // as our model
    }

    updateSelection(...) {
        // Compute new selection, based on shift/ctrl state
        let newSelection = computeSelection(this.editContext.currentSelection, ...);
        this.editContext.selectionChanged(newSelection.start, newSelection.end);
    }

    insertNewline() {
        this.editContext.textChanged(this.selection.start, this.selection.end, "\\n");
    }

    deleteCharacters(direction) {
        if (this.editContext.currentSelection.start === this.editContext.currentSelection.end) {
            // adjust start/end based on direction and whether we're at the beginning or end
        } else {
            this.editContext.textChanged(this.selection.start, this.selection.end, "");
        }
    }
}

// User defined class that can compute an HTML view, based on the text model
class EditableView {
    constructor(editContext, editRegionElement) {
        this.editContext = editContext;
        this.editRegionElement = editRegionElement;
    }

    queueUpdate() {
        if (!this.updateQueued) {
            requestAnimationFrame(this.renderView.bind(this));
            this.updateQueued = true;
        }
    }

    renderView() {
        this.editRegionElement.innerHTML = convertTextToHTML(
            this.editContext.currentTextBuffer, this.editContext.currentSelection);
        this.updateQueued = false;
    }

}


window.addEventListener("load", () => {
    let editContext = new EditContext();
    let model = new EditModel(editContext);
    let view = new EditableView(editContext, document.querySelector("#editregion");

    editContext.addEventListener("keydown", e => {
        // Handle control keys that don't result in characters being inserted
        switch (e.key) {
            case "ArrowLeft":
            case "Home":
                model.updateSelection(...);
                view.queueUpdate();
                break;
            case "Enter":
                model.insertNewline();
                view.queueUpdate();
                break;
            case "Backspace":
                model.deleteCharacters("back");
                view.queueUpdate();
                break;
            case "Control":
            ...
        }
    });

    editContext.addEventListener("keyup", e => {
        // Manage key modifier states
        switch (e.key) {
            case "Control":
            case "Shift":
            ...
        }
    });

    editContext.addEventListener("textupdating", (e => {
        model.updateText(e.text, e.updateRange);

        // Do not call textChanged on editContext, as we're accepting
        // the incoming input.

        view.queueUpdate();
    });

    editContext.addEventListener("focus", (e => {
        // Update view to reflect the focus state
    });

    editContext.addEventListener("blur", (e => {
        // Update view to reflect the focus state
    });


    editContext.focus();
})
```

```javascript

    editContext.addEventListener("textformatupdate", (e => {
        let viewRange = view.convertModelRangeToDOMRange(e.formatRange);
        view.applyStyleToRange(e.color, viewRange);
        ...
    });
```

## WebIDL

```webidl
interface EditContextTextRange {
    readonly attribute unsigned long start;
    readonly attribute unsigned long end;
};

interface EditEvent : Event {
};

interface TextUpdate : EditEvent {
    readonly attribute EditContextTextRange updateRange;
    readonly attribute USVString updateText;
    readonly attribute EditContextTextRange newSelection;
};

interface SelectionUpdateEvent : EditEvent {
    readonly attribute EditContextTextRange updatedSelectionRange;
};

interface TextFormatUpdateEvent : EditEvent {
    readonly attribute EditContextTextRange formatRange;
    readonly attribute DOMString color;
    readonly attribute DOMString backgroundColor;
    readonly attribute DOMString underlineColor;
    readonly attribute DOMString underlineType;
    readonly attribute DOMString reason;
};

/// @event name="keydown", type="KeyboardEvent"
/// @event name="keyup", type="KeyboardEvent"
/// @event name="textupdate", type="TextUpdateEvent"
/// @event name="selectionupdate", type="SelectionUpdateEvent"
/// @event name="textformatupdate", type="TextFormatUpdateEvent"
/// @event name="focus", type="FocusEvent"
/// @event name="blur", type="FocusEvent"
/// @event name="compositionstart", type="CompositionEvent"
/// @event name="compositioncompleted", type="CompositionEvent"
interface EditContext : EventTarget {
    void focus();
    void blur();
    void selectionChanged(unsigned long start, unsigned long end);
    void layoutChanged(DOMRect controlBounds, DOMRect selectionBounds);
    void textChanged(unsigned long start, unsigned long end, USVString updateText);

    readonly attribute USVString currentTextBuffer;
    readonly attribute EditContextTextRange currentSelection;
};
```

## Open issues

Do we need the composition events? I suspect we do, but I don't have an example yet, given that textupdate will fire with the composed candidates.


