# EditContext API Explained
This document proposes a new API for integrating web applications with the input services of the operating system.

## Background and Motivation
Editing on the web has evolved significantly over the last two decades.  The built-in form controls of the browser are no longer sufficient as the editing experience has evolved from filling in form data to rich editing experiences in web applications like Office 365, GMail, and Visual Studio Online.

Contenteditable, the most recent editing innovation developed as part of HTML5, was meant to provide a new editing primitive suitable for building rich editing experiences, but came with a design flaw in that it couples the document model and view.  As a result, contenteditable is only suitable for editing HTML in a WYSYWIG fashion, and doesn't meet the needs of many editing applications.

Despite their shortcomings, contenteditable and the good old textarea element are used by many web-based editors today, as **without a focused, editable element in the DOM, there is no way for an author to leverage the advanced input features of modern operating systems including composition, handwriting recognition, shape-writing, and more**.

### Challenges using contenteditable and textarea to facilitate input
When using a contenteditable element, it is typically a visible part of the editing application's view containing the content to be edited.  This approach limits the app's ability to enhance the view, as the view (i.e. the DOM) is also the authoritative source on the contents of the document being edited.

Below is an image of the Visual Studio editing experience.  This would be difficult to replicate on the web using a contenteditable element as the view contains more information that the plain-text document being edited. Specifically, the grey text shows commit history and dependency information that is not part of the plain-text C# document being edited. Because input methods query for the text of the document nearby the selection for context, e.g. to provide suggestions, the divergence in document content and presentation can negatively impact the editing experience. 

![Visual Studio's rich view of a plain-text document](visual_studio_editing_experience.png)

An additional issue with using contenteditable is that the editing operations built-in to the browser are designed to edit HTML, which produces results that are unrelated to the change in the actual editable document.  For example, typing an 'x' after public in the document shown above when using a contenteditable element would continue with the preceding blue color making "publicx" look like a keyword. To avoid the issue, authors may prevent the default handling of input (e.g. on keydown). This can be done, but only for regular keyboard input and when a composition is not in progress, specifically, **there is no way to prevent modification of the DOM during composition without  disabling composition**.

For the reasons above, many editing applications opt for an alternative approach using a hidden textarea to capture input. The hidden textarea allows the app to decouple its view of the document from the data the browser will interpret as being editable. This provides flexibility in the presentation of the document and works around issues with the previous contenteditable approach.

However, for the hidden textarea approach to work, it must be focused, and it must contain the browser's native selection. These constraints come with the following drawbacks:

1. Native selection cannot be used as part of the view (because its being used in the hidden textarea instead), which adds complexity (since the editing app must now build its own representation of selection and the caret), and (unless rebuilt by the editing app) eliminates specialized experiences for touch where selection handles and other affordances can be supplied for a better editing experience.
1. When the location of selection in the textarea doesn't perfectly match the location of selection in the view, it creates problems when software keyboards attempt to reposition the viewport to where the system thinks editing is occurring.  Input method-specific UI meant to be positioned nearby the selection, for example the UI presenting candidates for phonetically composed text, can also be negatively impacted (in that they will be placed not nearby the composed text in the view).
1. Accessibility is negatively impacted. Assistive technologies may highlight the textarea to visually indicate what content the assisted experience applies to.  Given that the textarea is likely hidden and not part of the view, these visual indicators will likely appear in the wrong location.  Beyond highlighting, the model for accessibility should often match the view and not the portion of the document copied into a textarea. For assistive technology that reads the text of the document, the wrong content may be read as a result.

## A New Solution: EditContext
To avoid the side-effects that come from using editable elements to integrate with input services, we propose using a new object, EditContext, that when created provides a connection to the operating system's input services.

The EditContext is an abstraction over a shared, plain-text input buffer that provides the underlying platform with a view of the content being edited. Creating an EditContext conceptually tells the browser to instantiate the appropriate machinery to create a target for text input operations. In addition to maintaining a shared buffer, the EditContext also has the notion of selection, expressed as offsets into the buffer, state to describe layout of bounds of the view of the editable region, as well as the bounds of the selection. These values are provided in JavaScript to the EditContext and communicated by the browser to the underlying platform to enable rich input experiences.

Having a shared buffer and selection for the underlying platform allows it to provide input methods with context regarding the contents being edited, for example, to enable better suggestions while typing. Because the buffer and selection are stateful, updating the contents of the buffer is a cooperative process between the characters coming from user input and changes to the content that are driven by other events. Cooperation takes place through a series of events dispatched on the EditContext to the web application &mdash; these events are requests from the underlying platform to read or update the text of the web application. The web application can also proactively communicate changes in its text to the underlying platform by using methods on the EditContext.

A web application is free to create multiple EditContexts if there are multiple distinct editable areas in the application.  Only the focused EditContext (designated by calling the focus method on the EditContext object) receives updates from the system's input services.  Note that the concept of the EditContext being focused is separate from that of the document's activeElement which will continue to determine the target for dispatching keyboard events.

While an EditContext is active, the text services framework may read the following state:
* Text content
* Selection offsets into the text content
* The location (on the screen) of selection
* The location (on the screen) of the content this EditContext represents

The text services framework can also request that the buffer or view of the application be modified by requesting that:
* The text contents be updated
* The selection of be relocated
* The text contents be marked over a particular range, for example to indicate visually where composition is occurring

The web application is free to communicate before, after or during a request from the underlying platform that its:
* Text content has changed
* Selection offsets have changed
* The location (on the screen) of selection or content has changed
* The preferred mode of input has changed, for example, to provide software keyboard specialization 

### Example 1

Create an EditContext and have it start receiving events when its associated container gets focus. After creating an EditContext object, the web application should initialize the text and selection (unless the state of the web application is correctly represented by the empty defaults) via a dictionary passed to the constructor.  Additionally, the layout bounds of selection and conceptual location of the EditContext in the view should be provided by calling ```layoutChanged()```.

```javascript
let editContainer = document.querySelector("#editContainer");

let editContextDict = {
    mode: "text",
    text: "Hello world",
    selection: { start: 11, end: 11 }
};
let editContext = new EditContext(editContextDict);

let model = new EditModel(editContext, editContextDict.text, editContextDict.selection);
let view = new EditableView(editContext, model, editContainer);

editContainer.addEventListener("focus", () => editContext.focus());
window.requestAnimationFrame(() => {
    editContext.layoutChanged(editContainer.getBoundingClientRect(), computeSelectionBoundingRect());
});

editContainer.focus();
```

Assuming ```model``` represents the document model for the editable content, and ```view``` represents an object that produces an HTML view of the document (see [Code Appendix](#code-appendix) for more details on example implementations), register for textupdate and keyboard related events (note that keydown/keyup are still delivered to the edit container, i.e. the activeElement):

```javascript
editContainer.addEventListener("keydown", e => {
    // Handle control keys that don't result in characters being inserted
    switch (e.key) {
        case "Home":
            model.updateSelection(...);
            view.queueUpdate();
            break;
        case "Backspace":
            model.deleteCharacters(Direction.BACK);
            view.queueUpdate();
            break;
        ...
    }
});

editContext.addEventListener("textupdate", e => {
    model.updateText(e.newText, e.updateRange);

    // Do not call textChanged on editContext, as we're accepting
    // the incoming input.

    view.queueUpdate();
});

editContext.addEventListener("selectionupdate", e => {
    model.setSelection(e.start, e.end);

    // Do not call selectionChanged on editContext, as we're accepting
    // the incoming event.

    // Update the view to render the new selection
    view.queueUpdate();
});

editContext.addEventListener("textformatupdate", e => {
    view.addFormattedRange(e.formatRange)
});
```

## Basic scenarios

![shared text](shared_text_basic.png)

The typical flow of text input comes from the user pressing keys on the keyboard. These are delivered to the browser, which opted-in to using the system's text services framework in order to integrate with the IMEs installed on the system. This will cause input to be forwarded to the active IME. The IME is then able to query the text services to read contextual information related to the underlying editable text in order to provide suggestions, and potentially modify which character(s) should be written to the shared buffer. These modifications are typically performed based on the current selection, which is also communicated through the text services framework. When the shared buffer is updated, the web application will be notified of this via the ```textupdate``` event.

When an EditContext has focus, this sequence of events is fired when a key is pressed and an IME is not active:

|  Event        | EventTarget        |
| ------------- | ------------------ |
|  keydown      | focused element    |
|  textupdate   | active EditContext |
|  keyup        | focused element    |

Because the web page has opted in to the EditContext having focus, keypress is not delivered, as it is redundant with the `textupdate` event for editing scenarios.

Now consider the scenario where an IME is active, the user types in two characters, then commits to the first IME candidate by hitting 'Space'.

|  Event                | EventTarget        |  Related key in sequence
| -------------         | -----------------  | -------------------
|  keydown              | focused element    |  Key 1
|  compositionstart     | active EditContext |  ...
|  textupdate           | active EditContext |  ...
|  keyup                | focused element    |  ...
|  keydown              | focused element    |  Key 2
|  textupdate           | active EditContext |  ...
|  keyup                | focused element    |  ...
|  keydown              | focused element    |  Space
|  textupdate           | active EditContext |  (committed IME characters available in event.updateText)
|  keyup                | focused element    |  ...
|  compositioncomplete  | active EditContext |

Note that the composition events are also not fired on the focused element as the composition is operating on the shared buffer that is represented by the EditContext.

### Externally triggered changes

Changes to the editable contents can also come from external events, such as collaboration scenarios. In this case, the web editing framework may get some XHR completion that notifies it of some pending collaboartive change that another user has committed. The framework is then responsible for writing to the shared buffer, via the ```textChanged()``` method.

![external input](external_input.png)

## API Details

The ```textupdate``` event will be fired on the EditContext when user input has resulted in characters being applied to the editable region. The event signals the fact that the software keyboard or IME updated the text (and as such that state is reflected in the shared buffer at the time the event is fired). This can be a single character update, in the case of typical typing scenarios, or multiple-character insertion based on the user changing composition candidates. Even though text updates are the results of the software keyboard modifying the buffer, the creator of the EditContext is ultimately responsible for keeping its underlying model up-to-date with the content that is being edited as well as telling the EditContext about such changes. These could get out of sync, for example, when updates to the editable content come in through other means (the backspace key is a canonical example &mdash; no ```textupdate``` is fired in this case, and the consumer of the EditContext should detect the keydown event and remove characters as appropriate).

Updates to the shared buffer driven by the webpage/javascript are performed by calling the ```textChanged()``` method on the EditContext. ```textChanged()``` accepts a range (start and end offsets over the underlying buffer) and the characters to insert at that range. ```textChanged()``` should be called anytime the editable contents have been updated. However, in general this should be avoided during the firing of ```textupdate``` as it will result in a canceled composition.

The ```selectionupdate``` event may be fired by the browser when the IME wants a specific region selected, generally in response to an operation like IME reconversion.

```selectionChanged()``` should be called by the web page in order to communicated whenever the selection has changed. It takes as parameters a start and end character offsets, which are based on the underlying flat text buffer held by the EditContext. This would need to be called in the event that a combination of control keys (e.g. Shift + Arrow) or mouse events result in a change to the selection on the edited document.

The ```layoutChanged()``` method must be called whenever the [client coordinates](https://drafts.csswg.org/cssom-view/#dom-mouseevent-clientx) (i.e. relative to the origin of the viewport) of the view of the EditContext have changed. This includes if the viewport is scrolled or the position of the editable contents changes in response to other updates to the view. The arguments to this method describe a bounding box in client coordinates for both the editable region and also the current selection. 

The ```textformatupdate``` event is fired when the input method desires a specific region to be styled in a certain fashion, limited to the style properties that correspond with the properties that are exposed on TextFormatUpdateEvent (e.g. backgroundColor, textDecoration, etc.). The consumer of the EditContext should update their view accordingly to provide the user with visual feedback as prescribed by the software keyboard. Note that this may have accessibility implications, as the IME may not be aware of the color scheme of the editable contents (i.e. may be requesting blue highlight on text that was already blue).

```compositionstart``` and ```compositioncompleted``` fire when IME composition begins and ends. It does not provide any other contextual information, as the ```textupdate``` events will let the application know the text that the user chose to insert.

There can be multiple EditContext's per document, and they each have a notion of focused state. Because there is no implicit representation of the EditContext in the HTML view, focus must be managed by the web developer, most likely by forwarding focus calls from the DOM element that contains the editable view. ```focus``` and ```blur``` events are fired on the EditContext in reponse to changes in the focused state. EditContext focus is bound to the element that was focused when the EditContext became active, that is, if the focused element changes, the EditContext will also lose focus.

The ```mode``` property on the EditContext (also can be passed in a dictionary to the constructor) denotes what type of input the EditContext is associated with. This information is typically provided to the underlying system as a hint for which software keyboard to load (e.g. keyboard for phone numbers may be a numpad instead of the default keyboard). This defaults to 'text'.

## Implementation notes

In a browser where the document thread is separate from the input thread, there is some synchronization that needs to take place so that the web developer can provide a consistent and reliable editing experience to the user. Because the threads are separate, there must be a copy of the shared buffer to avoid synchronous communication between the two threads. With a single buffer, synchronous commuincation would be necessary to provide synchronous responses as required by operating system queries about the contents of the document. The copies of the shared buffer are then managed by a component that lives on the input thread, and a component that lives in the web platform component. The copies can then be synchronized by converting updates to asynchronous notifications with ACKs, where the updates are not committed until it has been confirmed as received by the other thread.

As in the previous section the basic flow of input in this model could look like this:
![threaded buffer flow](thread_basic.png)
![threaded buffer flow external](thread_external.png)

### Resolving conflicts

It is possible for conflicts to occur between the input thread and script thread updating the shared buffer. These can be resolved in such a way that the users input is not dropped and is consistently applied in the expected manner.

Let's say there is an EditContext that starts with a shared buffer of ```"abc|"``` with the selection/caret being at the end of the buffer. The user types ```d``` and approximately the same time, there is a collaborative update (perhaps triggered/detected by a completed XHR) to the document that prepends ```x``` &mdash; these are delivered independently to each thread.
1. The input thread sees the insertion of ```d``` at position 3, the shared buffer is updated to ```"abcd|```, and the input thread component keeps a record of this pending action. It then sends a textupdate notification to the document thread. 
2. Meanwhile, prior to receiving that notification, the document thread processes the prepending of ```x``` and sends a notification to the input thread of this text change, keeping track of the fact that it too has a pending operation. 
3. The input thread receives the text change notification prior to the ACK for its pending textupdate. To resolve this conflict, it undoes the pending insertion of ```d``` and applies the text change. It is then determined that the previous insertion location of ```d``` was not modified* by the text change, so it replays the insertion of ```d```, but at position 4 instead and keeps this as a pending update. This leaves the shared buffer as ```"xabcd|"```. The ACK of the text change is sent to the document thread.
4. The document thread then yields and receives the text update of ```d``` at position 3. It determines that it has a pending operation outstanding, so runs through the same algorithm as the input thread &mdash; the ```x``` is already prepended but the text update is determined to not have been modified by the pending operations. The text update is then adjusted and applied as ```d``` at position 4. The text update is then ACK'd back to the input thread.
5. The ACK of the text change is received on the document thread and the pending operation is removed (committed)
6. The ACK of the text update is received on the input thread and its pending operation is also removed (committed)

\* An operation is only affected by a change if the range on which it was originally intended to apply to has been modified.

![thread conflict](thread_conflict.png)

The layout position of the EditContext is also reported to the input thread component, which caches the values and lets the text services know that the position has changed. In turn, it uses the cached values to respond to any read requests from the text services.

## Code Appendix

Example of a user-defined EditModel class that contains the underlying model for the editable content
```javascript
// User defined class 
class EditModel {
    constructor(editContext, text, selection) {
        // This specific model uses the underlying buffer directly so doesn't
        // store model directly.
        this.editContext = editContext;
        this.text = text;
        this.selection = new Selection();
        this.setSelection(selection.start, selection.end);
    }

    updateText(text, updateRange, newSelection) {
        this.text = this.text.slice(0, updateRange.start) +
            text + this.text.slice(updateRange.end, this.text.length);
    }

    setSelection(start, end) {
        this.selection.start = start;
        this.selection.end = end;
    }

    updateSelection(...) {
        // Compute new selection, based on shift/ctrl state
        let newSelection = computeSelection(this.editContext.currentSelection, ...);
        this.setSelection(newSelection.start, newSelection.end);
        this.editContext.selectionChanged(newSelection.start, newSelection.end);
    }


    deleteCharacters(direction) {
        if (this.selection.start !== this.selection.end) {
            // Notify EditContext that things are changing.
            this.editContext.textChanged(this.selection.start, this.selection.end, "");
            this.editContext.selectionChanged(this.selection.start, this.selection.start);

            // Update internal model state
            this.text = text.slice(0, this.selection.start) +
                text.slice(this.selection.end, this.text.length)
            this.setSelection(this.selection.start, this.selection.start);
        } else {
            // Delete a single character, based on direction (forward or back).
            // Notify editContext of changes
            ...
        }
    }
}
```

Example of a user defined class that can compute an HTML view, based on the text model
```javascript
class EditableView {
    constructor(editContext, editModel, editRegionElement) {
        this.editContext = editContext;
        this.editModel = editModel;
        this.editRegionElement = editRegionElement;

        // When the webpage scrolls, the layout position of the editable view
        // may change - we must tell the EditContext about this.
        window.addEventListener("scroll", this.notifyLayoutChanged.bind(this));

        // Same response is needed when the window is resized.
        window.addEventListener("resize", this.notifyLayoutChanged.bind(this));
    }

    queueUpdate() {
        if (!this.updateQueued) {
            requestAnimationFrame(this.renderView.bind(this));
            this.updateQueued = true;
        }
    }

    addFormattedRange(formatRange) {
        // Replace any previous formatted range by overwriting - there
        // should only ever be one (specific to the current composition).
        this.formattedRange = formatRange;
        this.queueUpdate();
    }

    renderView() {
        this.editRegionElement.innerHTML = this.convertTextToHTML(
            this.editModel.text, this.editModel.selection);

        notifyLayoutChanged();

        this.updateQueued = false;
    }

    notifyLayoutChanged() {
        this.editContext.layoutChanged(this.computeBoundingBox(), this.computeSelectionBoundingBox());
    }

    convertTextToHTML(text, selection) {
        // compute the view (code omitted for brevity):
        // - if there is no selection, return a string with the text contents
        // - surround the selection by a <span> that has the
        //   appropriate background/foreground colors.
        // - surround the characters represented by this.formatRange
        //   with a <span> whose style has properties as specified by
        //   the properties on 'this.formattedRange': color
        //   backgroundColor, textDecorationColor, textUnderlineStyle
    }
}
```
## Open Issues

How to deal EditContext focus when the focused element itself is editable? In the current proposed model, the focused element doesn't receive things like composition events &mdash; should an editable element receive these? It feels like we should treat these the same as when the text input operations are redirected and not deliver those events to the editable element.

Is there a reason we might want to fire keypress on the focused element for non-IME input to EditContext. I couldn't think of one and this is generally a synthesized event anyways.

How does EditContext integrate with accessibility [Accessibility Object Model?](http://wicg.github.io/aom/explainer.html) so that screen readers also have context as to where the caret/selection is placed as well as the surrounding contents. This is another major complaint about implementing editors today - without a contenteditable with a full fidelity view, the default accessibility implementations report incorrect information.

Additionally, how can we provide better guidance around accessibility w.r.t. to the `textformatupdate` event.


It feels like we may need a mechanism by which ```layoutChanged()``` is more easily integrated. Currently there is no single point that the web developer knows it may need to report updated bounds, and the current model may encourage layout thrashing by computing bounds early in the process of producing a frame. Instead we may need to provide a callback during the rendering steps where the EditContext owner can set the updated layout bounds themselves. Perhaps IntersectionObservers is a good model where we can queue a microtask that will fire after the frame has been committed and layout has been computed &mdash; the layout update may be delayed by a frame, but the update is asynchronous anyways.


# Backup Material

#### Loss of Native Selection
Below the two animated gifs contrast the experience touching the screen to place a caret for Visual Studio Online (uses a hidden textarea for input and recreates its own selection and caret), versus placing a caret in a contenteditable div in Chrome (grippers shown).

|  No grippers  | Native Grippers    |
| ------------- | ------------------ |
|  ![Missing Grippers in Visual Studio Online](NOGrippersGif.gif) | ![With Grippers in Visual Studio Online](withGrippers.gif) |

#### Existing Proposals:
Multiple approaches have been discussed during the meetings and through online discussions. The group has [considered](https://w3c.github.io/editing/contentEditable.html) adding new attribute values to contenteditable (events, caret, typing) that in would allow web authors to prevent certain input types or to modify some input before it has made it into the markup. This approach hasn’t gotten much traction since browsers would still be building these behaviors on top of content editable thus, inheriting existing limitations.
 
Another approach taken was to introduce “beforeInput” event. While sounding great in concept, It eventually diverged into two different specs, [Level 1](https://www.w3.org/TR/input-events-1/) (Blink implementation) and [Level 2](https://www.w3.org/TR/input-events-2/) (Webkit implementation). The idea behind this event was to allow developer to preventDefault user input (except for IME cases) and provide information about the type of the input. Due to Android IME constraints, Blink made most of the beforeInput event types non-cancelable except for a few formatting input types. This divergence would only get worse over time and since it only solves a small subset of problems for the web, it can’t be considered as a long-term solution.

As an alternative to the failed beforeInput Google has proposed a roadmap in [Google Chrome Roadmap Proposal](https://docs.google.com/document/d/10qltJUVg1-Rlnbjc6RH8WnngpJptMEj-tyrvIZBPSfY/edit) where it was proposed to use existing browser primitives solving CE problems with textarea buffer approach, similar to what developers have already been doing. While we agree with it in concept, we don't think there is a clean way to solve this with existing primitives. Hence, we are proposing EditContext API.

#### What services do editable elements provide?
* Integrate with focus so keyboard, composition, clipboard and other ambient input events have a sensible place to be routed
* Integrate with the browser's native selection so the user can express where editing operations should occur
    * This includes displaying a caret to mark an insertion point for text
    * Includes implementing boundaries for selection so that it doesn't extend across the boundary of an editable element.
* Editable elements participate in the view such that they have a size and position known to the browser for themselves and their contents
* Provide editing operations that are specific to the type of editable element.
* Describe themselves to the OS input services:
    * To indicate if a specialized software keyboard could be used to facilitate input.
    * To enable composition and other forms of input like handwriting recognition and shape-writing.
    * To communicate position information so specialized UI for input can be displayed nearby editable regions and software keyboards can scroll the viewport to avoid occluding the editable area.
    * To provide a plain-text view of the document for context so that suggestions for auto-completion, spell-checking, and other services can be effectively provided.
* Handle specialized input requests from the OS
    * To highlight text to, for example, indicate where composition is occurring
    * Replace arbitrary runs of text to, for example, facilitate composition updates, provide auto-correction, and other services.
    * Change the location of selection or the caret.
    * Blur to lose focus
* Describe themselves to accessibility services in a special way to indicate that they are editable regions of the document.
* Enable clipboard operations
* Automatically become a drop target

#### If we build an editor without editable elements, i.e. using the DOM to render the view of the editable document, what are we missing?
* APIs to manage focus exist and can be applied to elements that are not editable.
* Size and position can be computed for elements in the view that represent the editable document.  APIs exist so this information can be queried and fulfill requests for accessibility and the OS input services if new APIs were created to communicate with those services.
* We lose the ability receive OS input-oriented requests.  An API is needed to replace this.
* We lose edit pattern support for accessibility.  An API is needed to replace this.
* To compensate for the loss of caret the editing app must provide its own and may also provide its own selection.
* APIs exist to register parts of the view as a drop target
* Clipboard events will still fire on paste even when the area is not editable, but the editing app must take actions on its own.

#### Lower-level APIs provided by modern operating systems
* To facilitate input using a variety of modalities, OSX, iOS, Android, Windows, and others have developed a stateful intermediary that sits between input clients (e.g. IMEs) and input consumers (i.e. an editing app).
* This intermediary asks the editing app to produce an array-like, plain-text view of its document and allows various input clients to query for that text, for example, to increase the accuracy of suggestions while typing.  It also can request that regions of the document be highlighted or updated to facilitate composition.  It also can request the location of arbitrary parts of the document so that the UI can be augmented with input-client specific UI.
* Browsers take advantage of these OS input services whenever an editable element is focused by registering for callbacks to handle the requests to highlight or update the content of the DOM and to fulfill the queries mentioned above.

#### Links to Relevant APIs
| Operating System |    |
| ---------------- | -- |
| OS X | [Implementing Text Input Support](https://developer.apple.com/library/archive/documentation/TextFonts/Conceptual/CocoaTextArchitecture/TextEditing/TextEditing.html#//apple_ref/doc/uid/TP40009459-CH3-SW25) |
| iOS | [Communicating with the Text Input System](https://developer.apple.com/library/archive/documentation/StringsTextFonts/Conceptual/TextAndWebiPhoneOS/LowerLevelText-HandlingTechnologies/LowerLevelText-HandlingTechnologies.html#//apple_ref/doc/uid/TP40009542-CH15-SW16) |
| Windows | [Text Services Framework](https://docs.microsoft.com/en-us/windows/desktop/TSF/text-services-framework) |