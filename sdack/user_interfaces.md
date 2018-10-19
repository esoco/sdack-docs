# User Interfaces

In SDACK user interfaces are created from interactive process steps. If a process encounters such a step during execution it will stop, display data to the application user, and wait for input. If the user provides the input the process continues to run, either until the next interaction or to it's end. On a process stop the framework queries all process parameters that have been declared as interactive by the causing step. Then it generates the corresponding user interface components and displays them to the user. Any input will be collected into the corresponding parameters and the process step will be notified when the process continues to run. Based on the input it can then decide whether to continue the execution or to request additional feedback from the user.

## `InteractionFragment`

The base for all process interactions is the class `InteractionFragment`. It is not a subclass of `ProcessStep` but they share a common base class \(`ProcessFragment`\). Although it is possible to have interactive process steps it is desirable to have re-usable user interface structures. Interaction fragments provide exactly that, allowing to combine multiple fragments in an interactive step. The root of an interaction always is a process step but that is handled transparently by the framework.

Most subclasses of `InteractionFragment` just need to implement the method `init()`. There they declare which process parameters shall be displayed to the user and which of these can receive input from the user. As process parameters are instances of `RelationType` \(from the ObjectRelations framework\) an interaction is defined by a list of parameter relation types. If fragments contain other fragments they just add the parameter lists of the sub-fragments into parameters that have a collection of relation types as their datatype. That means a complex interaction is defined by nested collections of process parameter relation types.

To modify the default presentation of interactive parameters a fragment can set corresponding properties as meta-data on the parameters. Most of these properties are defined in the class `UserInterfaceProperties`.

## Fragment Layout

The layout of a fragment defines how it's parameters are arranged in the generated UI. Because interactions are basically defined as lists of parameters the property to set the fragment layout is called `ListDisplayMode`. It is defined in the class `DataElementList`, a subclass of `DataElement`. Data elements are the data transport objects for the communication between server and client \(i.e. process and UI\). To set the layout of a fragment use the fluent parameter interface:

```text
fragmentParam().layout(ListDisplayMode.GRID);
```

Fragment layouts can be divided into several categories: simple, segmented, exclusive, and structured. Simple layouts just displays one or more parameters without further constraints. Segmented layouts place parameters in specific areas of the available space. Exclusive layouts display multiple parameters separately so that only one parameter \(which may be a sub-fragment\) is visible at a time. And structured layouts expect certain layout properties to be set on parameters to place them accordingly.

### Simple Layouts

* `FILL`: This mode simply fills the display area available to a fragment with a single parameter. Placing multiple parameters in this layout is not supported and may result in rendering problems or exception. \(HTML: `div` with width and height set to 100%\)
* `FLOW`: Parameters are arranged in a natural flow as defined by the UI context. For web applications that means that components will be placed by the web browsers layout engine. This can often be controlled by providing CSS styling.  \(HTML: `div`\)
* `MENU`: Adds all parameters to a structure that represents a menu or other navigation element. \(HTML: `nav`\)

### Segmented Layouts

* `DOCK`: Up to three parameters \(or sub-fragments\) can be arranged as left, right, and center elements. The left and/or right parameters must have an explicit size set so that the center element will fill the remaining space. To arrange the parameters vertically instead of horizontally the flag `VERTICAL` can be set on the fragment parameter.
* `SPLIT`: Similar to `DOCK` but the preset sizes of the border areas can be modified interactively by the user afterwards.

### Exclusive Layouts

* `TABS`: All fragment parameter will be placed in distinct horizontal tabs of which only one can be visible at a time. The user can select the visible tab interactively but the fragment that also query and set the index of the currently active tab through the property `CURRENT_SELECTION`.
* `STACK`: Similar to `TABS` but the parameters are arranged in a vertical stack instead.
* `DECK`: This mode doesn't provide a interface for the user. Therefore the currently visible parameter can only be set by the fragment through `CURRENT_SELECTION`.

### Structured Layouts

* `GRID`: Places parameter in distinct rows depending on the property flag `SAME_ROW`. If set on a parameter it will be placed in the same row as the previous parameters. For each parameter without this flag a new row will be created in the UI. In HTML the rows and their elements will become corresponding style names so that a CSS grid system can be applied to the fragment. \(HTML: hierarchy of `div` elements\)
* `FORM`: An input form. Works like `GRID` but uses a different UI element. \(HTML: `form` with `div` children\)
* `GROUP`: Groups the parameter visually and can have a title by adding a text parameter with the `LABEL_STYLE` set to `TITLE`. Typically used to group elements in a form. \(HTML: `fieldset` with `div` children\)
* `TABLE`: Arranges parameters in a table structure. In the case of HTML this will be a `table` element. Because this is the oldest layout mode it is also the default if no explicit mode is set on a fragment. But because the layout of nested tables often leads to problems and rendering tables responsively is difficult it is recommended to use one of the other layouts whenever possible. Table layouts should only be used if a \(HTML\) table is explicitly needed and only if no further nesting of fragments is necessary.

