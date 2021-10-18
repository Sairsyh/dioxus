# Custom Renderer

Dioxus is an incredibly portable framework for UI development. The lessons, knowledge, hooks, and components you acquire over time can always be used for future projects. However, sometimes those projects cannot leverage a supported renderer or you need to implement your own better renderer.

Great news: the design of the renderer is entirely up to you! We provide suggestions and inspiration with the 1st party renderers, but only really require implementation of the `RealDom` trait for things to function properly.

## The specifics:

Implementing the renderer is fairly straightforward. The renderer needs to:

1. Handle the stream of edits generated by updates to the virtual DOM
2. Register listeners and pass events into the virtual DOM's event system
3. Progress the virtual DOM with an async executor (or disable the suspense API and use `progress_sync`)

Essentially, your renderer needs to implement the `RealDom` trait and generate `EventTrigger` objects to update the VirtualDOM. From there, you'll have everything needed to render the VirtualDOM to the screen.

Internally, Dioxus handles the tree relationship, diffing, memory management, and the event system, leaving as little as possible required for renderers to implement themselves.

For reference, check out the WebSys renderer as a starting point for your custom renderer.

## Trait implementation and DomEdits

The current `RealDom` trait lives in `dioxus_core/diff`. A version of it is provided here (but might not be up-to-date):

```rust
pub trait RealDom<'a> {
    fn handle_edit(&mut self, edit: DomEdit);
    fn request_available_node(&mut self) -> ElementId;
    fn raw_node_as_any(&self) -> &mut dyn Any;
}
```

For reference, the "DomEdit" type is a serialized enum that represents an atomic operation occurring on the RealDom. The variants roughly follow this set:

```rust
enum DomEdit {
    PushRoot,
    AppendChildren,
    ReplaceWith,
    CreateTextNode,
    CreateElement,
    CreateElementNs,
    CreatePlaceholder,
    NewEventListener,
    RemoveEventListener,
    SetText,
    SetAttribute,
    RemoveAttribute,
}
```

The Dioxus diffing mechanism operates as a [stack machine](https://en.wikipedia.org/wiki/Stack_machine) where the "push_root" method pushes a new "real" DOM node onto the stack and "append_child" and "replace_with" both remove nodes from the stack. 


### An example

For the sake of understanding, lets consider this example - a very simple UI declaration:

```rust
rsx!( h1 {"hello world"} )
```

To get things started, Dioxus must first navigate to the container of this h1 tag. To "navigate" here, the internal diffing algorithm generates the DomEdit `PushRoot` where the ID of the root is the container. 

When the renderer receives this instruction, it pushes the actual Node onto its own stack. The real renderer's stack will look like this:

```rust
instructions: [
    PushRoot(Container)
]
stack: [
    ContainerNode,
]
```

Next, Dioxus will encounter the h1 node. The diff algorithm decides that this node needs to be created, so Dioxus will generate the DomEdit `CreateElement`. When the renderer receives this instruction, it will create an unmounted node and push into its own stack:

```rust
instructions: [
    PushRoot(Container),
    CreateElement(h1),
]
stack: [
    ContainerNode,
    h1,
]
```
Next, Dioxus sees the text node, and generates the `CreateTextNode` DomEdit:
```rust
instructions: [
    PushRoot(Container),
    CreateElement(h1),
    CreateTextNode("hello world")
]
stack: [
    ContainerNode,
    h1,
    "hello world"
]
```
Remember, the text node is not attached to anything (it is unmounted) so Dioxus needs to generate an Edit that connects the text node to the h1 element. It depends on the situation, but in this case we use `AppendChildren`. This pops the text node off the stack, leaving the h1 element as the next element in line.

```rust
instructions: [
    PushRoot(Container),
    CreateElement(h1),
    CreateTextNode("hello world"),
    AppendChildren(1)
]
stack: [
    ContainerNode,
    h1
]
```
We call `AppendChildren` again, popping off the h1 node and attaching it to the parent:
```rust
instructions: [
    PushRoot(Container),
    CreateElement(h1),
    CreateTextNode("hello world"),
    AppendChildren(1),
    AppendChildren(1)
]
stack: [
    ContainerNode,
]
```
Finally, the container is popped since we don't need it anymore.
```rust
instructions: [
    PushRoot(Container),
    CreateElement(h1),
    CreateTextNode("hello world"),
    AppendChildren(1),
    AppendChildren(1),
    Pop(1)
]
stack: []
```
Over time, our stack looked like this:
```rust
[]
[Container]
[Container, h1]
[Container, h1, "hello world"]
[Container, h1]
[Container]
[]
```

Notice how our stack is empty once UI has been mounted. Conveniently, this approach completely separates the VirtualDOM and the Real DOM. Additionally, these edits are serializable, meaning we can even manage UIs across a network connection. This little stack machine and serialized edits makes Dioxus independent of platform specifics.

Dioxus is also really fast. Because Dioxus splits the diff and patch phase, it's able to make all the edits to the RealDOM in a very short amount of time (less than a single frame) making rendering very snappy. It also allows Dioxus to cancel large diffing operations if higher priority work comes in while it's diffing.

It's important to note that there _is_ one layer of connectedness between Dioxus and the renderer. Dioxus saves and loads elements (the PushRoot edit) with an ID. Inside the VirtualDOM, this is just tracked as a u64.

Whenever a `CreateElement` edit is generated during diffing, Dioxus increments its node counter and assigns that new element its current NodeCount. The RealDom is responsible for remembering this ID and pushing the correct node when PushRoot(ID) is generated. Dioxus doesn't reclaim IDs of elements its removed, but your renderer probably will want to. To do this, we suggest using a `SecondarySlotMap` if implementing the renderer in Rust, or just deferring to a HashMap-type approach.

This little demo serves to show exactly how a Renderer would need to process an edit stream to build UIs. A set of serialized EditStreams for various demos is available for you to test your custom renderer against.

## Event loop

Like most GUIs, Dioxus relies on an event loop to progress the VirtualDOM. The VirtualDOM itself can produce events as well, so it's important that your custom renderer can handle those too.

The code for the WebSys implementation is straightforward, so we'll add it here to demonstrate how simple an event loop is:

```rust
pub async fn run(&mut self) -> dioxus_core::error::Result<()> {
    // Push the body element onto the WebsysDom's stack machine
    let mut websys_dom = crate::new::WebsysDom::new(prepare_websys_dom());
    websys_dom.stack.push(root_node);

    // Rebuild or hydrate the virtualdom
    self.internal_dom.rebuild(&mut websys_dom)?;

    // Wait for updates from the real dom and progress the virtual dom
    loop {
        let user_input_future = websys_dom.wait_for_event();
        let internal_event_future = self.internal_dom.wait_for_event();

        match select(user_input_future, internal_event_future).await {
            Either::Left((trigger, _)) => trigger,
            Either::Right((trigger, _)) => trigger,
        }
    }

    while let Some(trigger) =  {
        websys_dom.stack.push(body_element.first_child().unwrap());
        self.internal_dom
            .progress_with_event(&mut websys_dom, trigger)?;
    }
}
```

It's important that you decode the real events from your event system into Dioxus' synthetic event system (synthetic meaning abstracted). This simply means matching your event type and creating a Dioxus `VirtualEvent` type. Your custom event must implement the corresponding event trait. Right now, the VirtualEvent system is modeled almost entirely around the HTML spec, but we are interested in slimming it down.

```rust
fn virtual_event_from_websys_event(event: &web_sys::Event) -> VirtualEvent {
    match event.type_().as_str() {
        "keydown" | "keypress" | "keyup" => {
            struct CustomKeyboardEvent(web_sys::KeyboardEvent);
            impl dioxus::events::KeyboardEvent for CustomKeyboardEvent {
                fn char_code(&self) -> usize { self.0.char_code() }
                fn ctrl_key(&self) -> bool { self.0.ctrl_key() }
                fn key(&self) -> String { self.0.key() }
                fn key_code(&self) -> usize { self.0.key_code() }
                fn locale(&self) -> String { self.0.locale() }
                fn location(&self) -> usize { self.0.location() }
                fn meta_key(&self) -> bool { self.0.meta_key() }
                fn repeat(&self) -> bool { self.0.repeat() }
                fn shift_key(&self) -> bool { self.0.shift_key() }
                fn which(&self) -> usize { self.0.which() }
                fn get_modifier_state(&self, key_code: usize) -> bool { self.0.get_modifier_state() }
            }
            VirtualEvent::KeyboardEvent(Rc::new(event.clone().dyn_into().unwrap()))
        }
        _ => todo!()
```

## Custom raw elements

If you need to go as far as relying on custom elements for your renderer - you totally can. This still enables you to use Dioxus' reactive nature, component system, shared state, and other features, but will ultimately generate different nodes. All attributes and listeners for the HTML and SVG namespace are shuttled through helper structs that essentially compile away (pose no runtime overhead). You can drop in your own elements any time you want, with little hassle. However, you must be absolutely sure your renderer can handle the new type, or it will crash and burn.

These custom elements are defined as unit structs with trait implementations.

For example, the `div` element is (approximately!) defined as such:

```rust
struct div;
impl div {
    /// Some glorious documentaiton about the class proeprty.
    #[inline]
    fn class<'a>(&self, cx: NodeFactory<'a>, val: Arguments) -> Attribute<'a> {
        cx.attr("class", val, None, false)
    }
    // more attributes
}
```
You've probably noticed that many elements in the `rsx!` and `html!` macros support on-hover documentation. The approach we take to custom elements means that the unit struct is created immediately where the element is used in the macro. When the macro is expanded, the doc comments still apply to the unit struct, giving tons of in-editor feedback, even inside a proc macro.


## Compatibility

Forewarning: not every hook and service will work on your platform. Dioxus wraps things that need to be cross-platform in "synthetic" types. However, downcasting to a native type might fail if the types don't match.

There are three opportunities for platform incompatibilities to break your program:

1. When downcasting elements via `Ref.downcast_ref<T>()`
2. When downcasting events via `Event.downcast_ref<T>()`
3. Calling platform-specific APIs that don't exist

The best hooks will properly detect the target platform and still provide functionality, failing gracefully when a platform is not supported. We encourage - and provide - an indication to the user on what platforms a hook supports. For issues 1 and 2, these return a result as to not cause panics on unsupported platforms. When designing your hooks, we recommend propagating this error upwards into user facing code, making it obvious that this particular service is not supported.

This particular code _will panic_ due to the unwrap on downcast_ref. Try to avoid these types of patterns. 

```rust
let div_ref = use_node_ref(cx);

cx.render(rsx!{
    div { ref: div_ref, class: "custom class",
        button { "click me to see my parent's class"
            onclick: move |_| if let Some(div_ref) = div_ref {
                log::info!("Div class is {}", div_ref.downcast_ref::<web_sys::Element>().unwrap().class())
            }
        }
    }
})

```

## Conclusion

That should be it! You should have nearly all the knowledge required on how to implement your own renderer. We're super interested in seeing Dioxus apps brought to custom desktop renderers, mobile renderer, video game UI, and even augmented reality! If you're interesting in contributing to any of the these projects, don't be afraid to reach out or join the community.