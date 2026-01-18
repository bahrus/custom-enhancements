## Author

Bruce B. Anderson

PR's, Issues [welcome](https://github.com/bahrus/custom-enhancements)

Last update: Jan 17, 2026

This is [one](https://github.com/whatwg/html/issues/2271) [of](https://eisenbergeffect.medium.com/2023-state-of-web-components-c8feb21d4f16) [a](https://github.com/WICG/webcomponents/issues/1029) [number](https://github.com/WICG/webcomponents/issues/727) of interesting proposals, one of which (or some combination?) can hopefully get buy-in from all three browser vendors.  This proposal borrows heavily from the others.

# Custom Attributes For [Simple Enhancements](https://www.w3.org/TR/design-principles/#simplicity)

Say all you need to do is to create an isolated behavior/enhancement/hook/whatever associated with an attribute -- say "log-to-console".  It enhances elements adorned with that attribute, logging the value of the attribute to the console when the element is clicked.  Here's how that would be done with this proposal.  It could be done more simply, with hard coded values, and without the commentary noise, so please allow for that when weighing the complexity.


```JS
customElementRegistry.mount({
    baseAttr: 'log-to-console', //canonical name of our (base) custom attribute.
    do: function(el, {mountInfo}){
        const {baseAttr} = mountInfo;
        // in this example, base will simply equal 'log-to-console', 
        // but this code is demonstrating how to code defensively, so that
        // the party (or parties) responsible for registering the enhancement 
        // could choose to modify the name(s), either globally, 
        // or inside a scoped registry in a different file.
        el.addEventListener('click', e => {
            const {target} = e;
            console.log(
                target.getAttribute(`enh-${baseAttr}`) 
                || target.getAttribute(`${baseAttr}`));
        })
    }
});
```

```HTML
<svg log-to-console="clicked on an svg"></svg>
    ...
<div log-to-console="clicked on a div"></div>

...

<some-custom-element 
    enh-log-to-console="clicked on some custom element">
</some-custom-element>
```

Done!

> [!NOTE]  
> What follows is a ridiculously large proposal.  At the risk of stating the obvious, I think it would make sense to roll it out in stages, starting with the most pressing, least controversial needs. Flaws in obscure features that I may have missed shouldn't jettison other asks, I hope.


## Why a function, and not a class or a class mixin?

Classes are also supported, as described below.  But for this simple example, a class would appear to be overkill.  Note that event listeners do *not* in [themselves cause a memory leak](https://github.com/whatwg/dom/issues/1396).

## Why "mount"?

Just a suggestion.  It ties in with this [additional / primitive proposal](https://github.com/WICG/webcomponents/issues/896) which this proposal might be considered to be extending (and vice versa).  Other suggestions are "inject", "enhance", "enhanceWith", "decorate", "decorateWith", the sky is the limit.  The specific suggestion of "inject" will become clearer in what follows, hopefully.

## Why the long attribute names?

It would be great if we could use a short attribute name, like "log".  That can be done for custom elements, why not custom enhancements?  This is especially important to consider because, as we will see, this proposal supports multiple attributes "owned" by an enhancement, so allowing for small names would help reduce carpal syndrome.

While it is a bit dicey to be supporting these single word attributes for custom elements, attributes that could conflict with future global attributes, that ship has sailed, and I view it as similar to key words in JavaScript, just a risk we have agreed is acceptable.

This proposal views the risks of following suit as being too high when we move on to enhancing higher-order components, especially as the platform is happily introducing more of them (🥳).  There is an informal understanding that built-in attributes won't have dashes in them (e.g. onclick, etc), [except once in a blue moon](https://github.com/webplatformco/project-custom-attributes/?tab=readme-ov-file#naming) (aria-*), so insisting on dashes (or maybe another short character like "_" ) in this context seems prudent.

The extra enh- is there to avoid conflicting with attributes that a custom element author may be using, so one of the aspects of this proposal is to suggest that the platform reserve "enh-" prefix similar to how it reserved "data-".

However, I've become aware that there is another informal understanding -- that the platform will only use ASCII characters for future attributes.

So developers wanting to capitalize on that and benefit from shorter names could define, under this proposal, an alternative mapping.  For example:

```JS
export const mountInfo = {
    baseAttr: '🪵'
}
```

```HTML
<svg 🪵="clicked on an svg"></svg>
    ...
<div 🪵="clicked on a div"></div>

...
```

Or developers could use single words using the small latin characters.  Or an emoji followed by ascii characters.  Whatever character sets the platform says it will never tap into.

Some risks to doing this:

1.  It may break xml (like svg tags)
2.  Programmatically setting such attributes seems to be currently difficult. It can be done by updating the value property of the .attributes[i]. This is a bit of a hurdle to overcome if the developer wishes to update the value of said attribute via client side scripting.
3.  It is even farther away from being "HTML5 compliant"
4.  Clashes between different libraries are extremely likely to occur (the shorter the name, the less the ability to "reserve" the name in npm or some other package manager), which is why we posit that a solution to scoped registry should ideally be shipping and proven before shipping this problem space.

It doesn't seem to me that any of these concerns would "block" the platform from doing its thing, so this proposal opts to empower the developer to take these risks.

> [!NOTE]
> I agree 100% with others that scoped registry being fully settled before some combination of these proposals get rolled out into production would appear to be the wise course of action.  Now that Safari has rolled out scoped registries, this proposal is incorporating the concepts.

## Support for classes

If the requirements for an enhancement would benefit from a stateful class that can be accessed publicly, use "spawn" instead of "do":

```JS
customElementRegistry.mount({
    baseAttr: 'log-to-console', //canonical name of our (base) custom attribute.
    spawn: class {
        constructor(enhancedElement, {mountInfo}){
            const {baseAttr} = mountInfo;
            enhancedElement.addEventListener('click', e => {
                const {target} = e;
                console.log(
                       target.getAttribute(`enh-${baseAttr}`)
                    || target.getAttribute(base)
                ); 
            });
        }
    }
});
```

A function prototype can also be used.  The word "spawn" as opposed to "do" indicates a number of significant differences in behavior:

1.  Spawn will use "new ..." before "invoking" the class constructor or function signature
2.  Unlike "do", "spawn" will cause a weak reference keyed off the passed in mount info object (more on that later), in order to provide a way for other parties to gain access to the instance.


The do function only gets called the first time all the criteria contained in mountInfo is met.  Likewise, the spawn instance only gets created once, unless the developer disposes it, as described below.


## How do I, or my users, access my class instance, and/or public properties / methods therein?

In lots of ways, which we will discuss far below.  We want to make this as convenient for all parties as possible, which we will get into later.  Chillax!

> [!NOTE]
> Adding public properties and methods to the class instance will *not* be accessible directly from the top level of the element being enhanced.


## Mount Registry API Shape

The structure below is optimized to allow for renaming things with as little pain as possible.

```TypeScript

export const isHello = Symbol.for('o8u9z9so50iLU_WwKk6O7Q');
const mountInfo: MountInfo = {
    //optional
    do(el: Element, {info, signal}: {info: MountInfo, signal: AbortSignal}){}
    //optional.
    //Can point directly to an already loaded Class constructor, or
    //as shown below, it can point to an async loader that allows
    //for lazy loading on demand.
    // either "do" or "spawn" is required.  Both is also fine.
    spawn: async () => {
        return MyEnhancementClassConstructor
    },
    //optional -- this is one place we can optionally find the instance of the spawned class that gets created:
    // oElement.enh[enhKey], e.g. oElement.enh.greetings
    // Don't be afraid to use, but you can avoid possible name clashes by not using if there's no need
    // to publicly expose the api outside of tightly constrained JavaScript
    /** @type {string | symbol | undefined} **/
    enhKey: 'greetings',
    //optional
    baseAttr: 'my-greetings',
    //optional -- only applicable if baseAttr is present
    attrTree:[
        //allow for standalone base attribute
        '', 
        {
            'hello': ['', 'how-are-you', 'hows-it-going']
        },
        {
            'goodbye': ['', 'last-words', 'ps':['', 'pps']]
        }
    ]
    //optional
    map: {
        //base attribute my-greetings
        '0': {
            instanceOf: 'Object',
            mapsTo: '.'
        },
        //my-greetings-hello
        '1': {
            instanceOf: 'Boolean',
            mapsTo: 'isHello'
        },
        //my-greetings-hello-how-are-you
        '1.1': {
            instanceOf: 'String',
            mapsTo: 'firstHelloGreeting'
        },
        //optional
        //this is for property binding using "assignGingerly".  
        //See https://github.com/bahrus/custom-enhancements?tab=readme-ov-file#symbolic-prop-shortcuts-and-support-for-dependency-injection 
        [isHello]: 'isHello'
    },
    //optional
    whereInstanceOf: [            
        HTMLInputElement, 
        HTMLTextArea, 
        SomeAlreadyLoadedCustomElementClass
    ],
    //optional
    whereElementMatches: 'input[type="text], textarea',
    //optional
    //can only enhance element if base attribute is present
    //very low priority requirement
    baseRequired: false,
    //optional -- if need to prevent memory leaks, mutation observers, etc
    //specify name of method of class instance (or function prototype) to use
    //this would be called before the enhanced element is about to be
    //purged from memory, assuming it is possible to prevent
    //these enhancements from being referenced counted as far as garbage collection.
    //only applicable if spawn has a value
    disposeKey: 'dispose'
    //optional -- only applicable if spawn has a value
    // if scheduling enhancements in sequence is needed, 
    // class must extend EventTarget (mixin someday?), 
    // and have a property with name specified below 
    // and dispatch event name when property switches to true:
    resolvedKey: 'resolved'
    //optional -- name of method that handles attribute changes
    //though personally I would rather we go with better mapping support
    //it is quite easy for developers to add mutation observers for this
    attributeChangeKey: 'attributeChangedCallback'
};
type branchitude = number;
type leafitude = number;
// in this example, we can "hard code"
// the names "branches" and "leaves"
// but this api will allow n-levels deep
type AttrCoordinates = 
    | `{branchitude}` 
    | `{branchitude}.{leafitude}`;
class MyEnhancement<
    HTMLInputElement, 
    HTMLTextArea, 
    SomeAlreadyLoadedCustomElementClass, 
    SVGElement, 
    HTMLMarqueeElement> {

    constructor(enhancedElement: TSupportedElements, enhanceInfo: MountInfo, initVals?: unknown){}

    dispose(enhancedElement: TSupportedElements, enhanceInfo: MountInfo){
        //prior to garbage collection
        //assuming it is possible to prevent
        //these enhancements from being referenced counted.
    }
	
    //maybe we don't need this, 
    // given the better mapping support
    // mentioned above?
	attributeChangedCallback(
        coordinates: AttrCoordinates,
        oldValue: string, 
        newValue: string,
        attrNode: Node,
        ) { 
        ...
    }

    //  Entirely optional filtering conditions for when the enhancement should be
    // allowed to be spawned.
    // maybe typescript could be enhanced to check for consistency?
    static supportedInstanceTypes = 
        [
            HTMLInputElement, 
            HTMLTextArea, 
            SomeAlreadyLoadedCustomElementClass, 
            SVGElement,
            HTMLMarqueeElement
        ]; //For example
    

    //Entirely optional
    static supportedCSSMatches = 'textarea[type="text"], input';

}
```

At the risk of overwhelming the reader, I want to amend the api above with a little completely optional nuance to allow for different attribute delimiters at different levels of the hierarchy:

```JS
const mountInfo: MountInfo = {
    baseAttr: '[_]greetings',
    attrTree:[
        '', 
        {
            '[:]hello': ['', '[--]how-are-you', '[--]hows-it-going']
        },
        {
            '[::]goodbye': ['', '[---]last-words', '[-]ps']
        }
    ]


};

```

If no prefix is specified, '-' is used by default.

This would allow for more readable syntax:

```html
<your-custom-element 
    enh_my-greetings="courtesy of hallmark" 
    enh_my-greetings:hello="select from gloomy section"
    enh_my-greetings:hello--how-are-you="one day closer to death"
    enh_my-greetings:good-bye="select from funny section"
    enh_my-greetings:good-bye---last-words="smell you later"
>
...
</your-custom-element>
```

### Filter support with supportedInstanceTypes, supportedCSSMatches

Having filtering support is there to benefit the developer first and foremost -- the developer is essentially publishing a "contract" of what kinds of elements they can support.  

Another key reason for adding this filtering capability is performance -- there is a cost to instantiating an enhancement class, adding it to the enhancements gateway, invoking the callback, and holding on to the class instance in memory, so anything we can do to declaratively prevent that seems like a win for all involved.

The idea for using supportedInstanceTypes, proposed [here](https://github.com/WICG/webcomponents/issues/1029) seems like it has some quite positive benefits:

1.  I think it could help avoid some timing issues of attempting to start enhancing an unknown element, by essentially enforcing a loading sequence of dependencies. 
2.  In some cases, especially with custom elements, it could group a bunch of custom elements together based on the base class.  CSS currently isn't so good at selecting elements based on a common prefix.
3.  The names can be validated by TypeScript.


>[!NOTE]
>Bear in mind that if no "whereElementMatches/whereInstanceTypes" is specified (the default), and if the "base/attrTree" option is also not specified or is empty, the platform will *not* automatically enhance every element.  The platform will only act when it finds a matching attribute pattern and/or css match and/or instanceType.  But it will **allow** enhancements to be programmatically spawned by the developer on all element types in that scenario.  In fact, the platform will **ignore** the base/branches/leaves criteria altogether when the developer programmatically spawns an enhancement, only using the "allowed*" value(s) (combined with the static supported* values specified by the enhancement author) to prevent unauthorized enhancements. 

###  What, if any, are the benefits of having a "has" (or some other equivalent) attribute?

> [!NOTE]
> To my great (temporary) relief, the main advocate of the "has" proposal and I seemed, for a short while at least, to have found common ground somewhere in the middle, based on observed attributes (which I recently discovered, was there all along with the has proposal, I missed it because I was so puzzled by the purpose of the "has" attribute). I should also point out that this proposal no longer considers the flat "observedAttributes" array to be the right model for this problem space, meaning a consensus appears even more precarious than before.

From a "developer advocacy" point of view, as the simple example I opened with demonstrates, there doesn't seem to be any benefit to having an extra "has" attribute -- that would just be clumsy and provide more opportunities for conflicts between different teams of developers.

I amended this proposal, though, to support multiple attributes for a single enhancement, in order to accommodate, as best I can, the [apparent appeal, which I can definitely relate to](https://github.com/WICG/webcomponents/issues/1029#issuecomment-1719996635) that the "has" attribute seemingly provides, kind of a way of grouping related attributes together.  I actually do believe there are very strong use cases where we *do* want one enhancement to be able to break down the "aspects" of the enhancement/behavior into multiple attributes.  Benefits are:

1.  The values can be simple strings / numbers / boolean, vs JSON.  
2.  Some frameworks may prefer to modify state via attributes instead of properties.
3.  Styling may benefit as well.

However, I think by supporting multiple attributes, requiring that they have dashes (or at least one underscore or a prefix like enh-?) or at least one non ascii character, and knowing that developers will go out of their way to avoid clashing with other libraries, we can achieve the same effect without telling the entire IT industry that their way of doing things is wrong.  **Almost no one is using a "has" attribute, so we should, I think, bend over backwards to not impose a new requirement in order to utilize the platform, without an extremely strong reason**.  So with this proposal, we can have attributes that naturally group together.  To take one very practical example where this makes sense:  Suppose we want to provide a userland implementation of [this proposal](https://github.com/whatwg/html/issues/2404).  We could define it like this, which this proposal supports:

```html
<time lang="ar-EG" 
    datetime=2011-11-18T14:54:39.929Z 
    be-intl-weekday=long be-intl-year=numeric be-intl-month=long
    be-intl-day=numeric>
</time>
```

Or perhaps there's a desire to be even more like the has solution and provide for the base attribute as well:

```html
<time lang="ar-EG" 
    datetime=2011-11-18T14:54:39.929Z
    be-intl 
    be-intl-weekday=long be-intl-year=numeric be-intl-month=long
    be-intl-day=numeric>
</time>
```

which this proposal also supports.

Even better, this proposal supports emoji's, which allows for quite short attribute names:

```html
<time lang="ar-EG" 
    datetime=2011-11-18T14:54:39.929Z 
    🌐-weekday=long 🌐-year=numeric 🌐-month=long
    🌐-day=numeric>
</time>
```

So what would make much more sense to me is rather than having a "has" requirement, to instead insist that all the attributes that a single enhancement "observes" begin with the same base (be-intl or 🌐 in this case), presumably tied to the package of the enhancement.  This proposal is now advocating enforcing such a rule, at least if the developer wishes to receive help from the platform with parsing and automated spawning/attachment.

The reason that the flat observedAttributes approach used for custom elements doesn't quite fit the bill, is that I think it will be quite natural for developers to start by doing something with the base attribute, like supporting a JSON structure for all the properties, then decide "you know, it would be helpful to provide a more semantic vocabulary where the aspects of the enhancement can be specified individually" and kind of slap it on.  Such is not the case with custom elements. 

### Better ergonomics for managing attribute changes

This proposal is focusing somewhat on managing attributes, similar to custom elements.   

I agree with others that the support that the platform currently provides for managing attributes with custom elements is insufficient.  So further compounding that shortcoming by creating a whole new api without additional support doesn't seem right. 

I like the promising ideas presented [here](https://github.com/WICG/webcomponents/issues/1029) as far as providing declarative support for managing properties and attributes.  Based on the reasoning above, I think it makes sense to consider such [improvements to custom elements themselves](https://github.com/WICG/webcomponents/issues/1045), and I see no reason not to carry over such ideas to custom enhancements, which this proposal does in fact do (with some variations where it makes sense).  

Or maybe it would make more sense to "pilot" such ideas on custom enhancements, and then apply to custom elements.  I think those ideas are 100% compatible with this proposal, and shouldn't break it in any way. 


## Backdrop

The WebKit team has raised a number of valid concerns about extending built-in elements.  I think one of the most compelling is the concern that, since the class extension is linked to the top level of the component, it will be natural for the developer to add properties and methods directly to that component.  Private properties and methods probably are of no concern.  It's the public ones which are.  Why? 

Because that could limit the ability for the platform to add properties without a high probability of breaking some component extensions in userland, thus significantly constraining their ability to allow the platform to evolve.  The same would apply to extending third party custom elements.  

Now why would a developer want to add public properties and methods onto a built-in element?  For the simple reason that the developer expects external components to find it beneficial to pass values to these properties, or call the methods.  I doubt the WebKit team would have raised this issue, unless they were quite sure there would be a demand for doing just that, and I believe they were right.

So for this reason (and others), the customized built-in standard has essentially been blocked.

And yet the need to be able to enhance [existing](https://aurelia.io/docs/templating/custom-attributes#simple-custom-attribute) [elements](https://dojotoolkit.org/reference-guide/1.10/quickstart/writingWidgets.html) [in](https://medium.com/@ignatovich.dm/creating-custom-directives-in-angular-a-beginner-friendly-guide-048596893a89) [cross-cutting](https://svelte.dev/docs#template-syntax-element-directives) [ways](https://mavo.io/docs/plugins) [has](https://knockoutjs.com/documentation/custom-bindings.html) [been](https://medium.com/@_edhuang/add-a-custom-attribute-to-an-ember-component-81f485f8d997) [demonstrated](https://alpinejs.dev/) [by](https://github.com/bahrus?tab=repositories&q=be-&type=&language=&sort=) [countless](https://htmx.org/docs/) [frameworks](https://vuejs.org/v2/guide/custom-directive.html), [old](https://jqueryui.com/about/) [and](https://riot.js.org/documentation/#html-elements-as-components) [new](https://make.wordpress.org/core/2023/03/30/proposal-the-interactivity-api-a-better-developer-experience-in-building-interactive-blocks/).  As the latter link indicates, there are great synergies that can be achieved between the client and the server with these declarative blocks of settings.  And making such solutions work across frameworks would be as profound as custom elements themselves.  The only alternative, working with nested custom elements, is [deeply](https://sitebulb.com/hints/performance/avoid-excessive-dom-depth/) [problematic](https://opensource.com/article/19/12/zen-python-flat-sparse#:~:text=If%20the%20Zen%20was%20designed%20to%20be%20a,obvious%20than%20in%20Python%27s%20strong%20insistence%20on%20indentation.).  And quite critically, some built-in elements **can't** be wrapped inside a custom element without breaking functionality and proper HTML decorum.

A close examination of these solutions usually indicates that the problem WebKit is concerned about is only percolating under the surface, pushed (or remaining) underground by a lack of an alternative solution.  One finds plenty of custom objects attached to the element being enhanced.  Just to take one example:  "_x_dataStack" is used by Alpine.js.  

Another example:  Currently if I go to https://walmart.com and right click and inspect their tile elements, I see some "react fiber" objects attached (__reactFiber$...), full of properties like memoizedProps, refs (a function) etc.  And reactProps (__reactProps$...), also a function prototype containing properties and methods.   [Preact does as well](https://www.nevermoreacademy.com/).

Other examples include closure, wiz, knockout.js, JQueryUI, HTMX, also using names that typically start with an underscore (HTMX uses dashes in the property name).

Clearly, they don't want to "break the web" with these naming conventions, but combine two such libraries together, and chances arise of a conflict.  And such naming conventions don't lend themselves to a very attractive api when being passed values from externally (such as via a framework).

It has [been argued](https://www.youtube.com/watch?v=uygxJ8Wxotc&t=319s) by the browser vendors that really, attaching such objects onto DOM elements makes optimizing the memory footprint of DOM elements problematic.  I'm hoping that providing this standard approach to allow what a huge percent of web sites are already doing would make that memory footprint problem surmountable.

## Custom Property Name-spacing

So, for an alternative to custom built-in extensions to be worthwhile, I strongly believe the alternative solution must first and foremost:

1.  Provide an avenue for developers to be able to safely add properties to their class without trampling on any other developer's classes, or the platform's, and 
2.  Just as critically, make those properties and methods public in a way that is (almost) as easy to access as the top level properties and methods themselves.

So the bottom-line is that the crux of this proposal is to allow developers to do this (with a little tender loving care):

```JavaScript
oInput.enh.myEnhancement.foo = bar;
oMyCustomElement.enh.yourEnhancement.bar = foo;
```

in a way that is recognized by the platform.

The most minimal solution, then, is for the web platform to simply announce that no built-in element will ever use a property with name "enhancements", push the message to web component developers not to use that name, that it is a reserved property, similar to dataset,  only to be used by third-party enhancement libraries.  Of course, the final name would need to be agreed to.  This is just my suggestion.  Some analysis would be needed to make sure that "enhancements" isn't already in heavy use by any web component library in common usage.

I think that would be a great start.  But the rest of this proposal outlines some ways the platform could assist third parties in implementing their enhancements in a more orderly fashion, so they can work together, and with the platform, in harmony.

## Justification for enh-

The next thing beyond that announcement would be what many (including myself) are clamoring for:  Safely adding custom attributes.

The platform informs web component developers to not use any attributes with a prefix that pairs up with the property gateway name, "enh"; that that prefix is only to be used by third parties to match up with the sub-property of "enh" they claim ownership of.  My suggestion is enh-*.  Continuing to use data- seems fundamentally flawed from a semantic point of view, and would also result in more overlapping uses between these two very different attribute meanings. 

So if server-rendered HTML looks as follows:

```html
<input my-enhancement='{"foo": "bar"}'>
<my-custom-element enh-your-enhancement='{"bar": "foo"}'>
```

... we can expect (but not guarantee) to see a class instance associated with each of those attributes, accessible via oInput.enh.myEnhancement and oMyCustomElement.enh.yourEnhancement, typically.

The requirement for the prefix can be dropped only if built-in elements are targeted, in which case the only requirement is that the attribute(s) contain (a) dash(es) or non ascii characters.  

Another aspect of this proposal that I think should be considered is that as the template instantiation proposal gels, looking for opportunities for these enhancements to play a role in the template instantiation process would be great. Many of the most popular such libraries do provide similar binding support as what template instantiation aims to support.  Basically, look for opportunities to make custom element enhancements serve the dual purpose of making template instantiation extendable, especially if that adds even a small benefit to performance.

## A note about naming

> [!NOTE]
> The use of the term "enhancement" has been greatly reduced as this proposal has become increasingly "unopinionated", hence the importance of the name has likely greatly diminished.

I started this journey placing great emphasis on the HTML attribute aspect of this, but as the concepts have marinated over time, I think it is a mistake to over emphasize that aspect.  The fundamental thing we are trying to do is to enhance existing elements, not attach strings to them.  

When we enhance existing elements during template instantiation, the attributes (can) go away, in order to optimize performance.  It is much faster and flexible to pass data through a common gateway property, not through attributes.  For similar reasons, when one big enhancement needs to cobble smaller enhancements together, again, the best gateway is not through attributes, which again would be inefficient, and would result in big-time cluttering of the DOM, but rather through the same common property gateway through which all these enhancements would be linked. 

### Why "enh", and not "behaviors"?

Granted, the majority of enhancements would likely fit our common idea of what constitutes a ["behavior"](https://www.brainbell.com/tutors/XML/XML_Book_B/DHTML_Behaviors.htm#:~:text=DHTML%20Behaviors%20are%20lightweight%20components%20that%20extend%20the,referenced%20in%20Internet%20Explorer%205%20by%20using%20styles.).

I think it is quite fine to use the term "behaviors" informally, just as we use "web components" to describe things informally, even though "customElements" is the formal api name.

But enhancements could also include specifying some common theme onto a white label web component, and contorting the language to make those sound like behaviors doesn't sound right:  "Be Picasso blue-period looking" for example.  I actually think this objection touches on a fairly important concern I have with the term "behavior" as the main term.  "Don't judge a book by its cover" seems to run afoul with the idea that affecting how an element looks should be called a "behavior."

The word "behavior" makes what we are doing quite adjacent to things that tie in closely with humans (and other living beings).  In fact, I've personally found it somewhat amusing to search for [names](https://github.com/bahrus?tab=repositories&q=be-&type=&language=&sort=) that apply both to DOM elements as well as (human) behavior, where it makes sense, and where it is appropriate.  But this close connection should give us pause.  We should acknowledge this connection, especially as it will likely affect our subconscious, including our dreams (or possibly nightmares).  

Yes, we should welcome the ability to explain complex things in terms we can relate to as human beings, so use of the term "behaviors" should certainly not be out of bounds.  But we should remember that the things we are talking about, even if they have children, siblings and parents, are not in fact human beings, and using a more generic term (enhancements) when it comes to more formal settings (like the actual API) would help with that.  It also expands our range of analogies we can reach for.  Think of the analogy of attaching a wing to a plane.  Does it make sense to refer to a wing as a "behavior"?

Some enhancements could be adding some common paragraph containing copyright text.  The dictionary defines behaviors as something associated with actions, so does that apply here?

Many are adding binding support to elements, which may or not resonate with developers as being a "behavior".

So "enhancements" seems to cover all bases.

Another reason to consider:  I think it would be wonderful if built-in elements started providing structured, namespaced paths to various features of the element.  Like what was done with styles from the get-go. As it is, having all the key properties at the top level, sometimes splitting up related properties like command and commandFor, has made the api rather unwieldy.    I think "behaviors" would be a great property name for built-in elements to use to indicate these are platform behaviors.  If so, use of "enhancements" for third party, well, enhancements, makes a lot of sense, I think.  So developers could access these built in behaviors via:

```JavaScript
oButton.assignGingerly({ 
    '?.behaviors?.command': {
        name:'doSomething', 
        for: oDialog
    }
});
```

Even in our current day when the platform provides no such structure, it would not be unreasonable for developers to assume that "behaviors" are built in, not third party enhancements, thus causing confusion due to this naming.

![In which I discover my keyboard has no support for print screen.  Screenshot of what happens if you type $0. on a random HTML Element.  The scrollbar can't even get past the D's.  Fortunately, "command" starts with c, so it can reach it for now.](https://github.com/bahrus/custom-enhancements/blob/baseline/20251101_182222.jpg?raw=true)

> In which I discover my keyboard has no support for print screen.  Screenshot of what happens if you type "$0." on a random HTML Element.  The scrollbar can't even get past the D's.  Fortunately, "command" starts with c, so it can reach it for now.

Granted, some frameworks might not support the ability to tap into this at first, but I suspect would accommodate it if the platform went in this direction.

Others prefer "behaviors" (but the others who do seem to think it is of zero consequence, whereas I think there is some substantial consequence to the decision, if that counts for anything). I'm open to both, maybe my reasoning above is wrong (but no one has yet to address my concerns head on).

Choosing the right name seems important, as it ought to align somewhat with the reserved sub-property of the element, as well as the reserved prefix for attributes (think data- / dataset).

## Should use of enh-* prefix for server-rendered (progressive) enhancement of custom elements be required (or even strongly suggested?)

The reason I think it would be reasonable for the prefix enh-* to be required, or at least strongly suggested is this:

1.  If enh-* is only encouraged the way data-* is encouraged, at least we could still count on custom element authors likely avoiding that prefix when defining their custom attributes associated with their element, to avoid confusion, making the "ownership" clear.
2.  But should a custom enhancement author choose a name that happens to coincide with one of the attribute names of another author's custom element, which seems quite likely to happen frequently, it still leaves the messy situation that the custom element's attribute gets improperly flagged as an enhancement.
3.  However, it could be argued, depending on how smoothly working with scoped registry proves to be in this context, that such catastrophes could be averted using the scoped registry.  This proposal provides out-of-the-box support for renaming any and all the attributes associated with an enhancement.  So maybe it shouldn't be required, and may seem silly for developers working in a closed environment, with enhancements they have no interest in publishing for general consumption.  But even so, I think it would be quite useful for the platform to at a minimum provide for a key prefix that developers can use to help avoid having to be always on the watch out for such collisions (which might not become immediately apparent until some user discovers it in production).

#  When should the spawn and/or do mount operations be supported by the platform?

## Spawning/referencing methods of the enhancements property

Unlike dataset, the enhancements property, added to the Element prototype, would have several methods available, making it easy for developers / frameworks to reference, and even spawning enhancements imperatively (without the need for attributes), for example during template instantiation (or later).

```JavaScript
// use this if "spawn" points to an already imported class
const enhancementInstance = oElement.enh.get(mountInfo);
//use this if "spawn" points to an an async loader or you aren't sure.
//In the asynchronous case, get should throw an error
const lazyLoadedInstance = await oElement.enh.await(mountInfo);
//await is definitely necessary here
const resolvedInstance = await oElement.enh.whenResolved(mountInfo);
```

All three of these methods would see if the enhancement has already been instantiated for the element, and if so, pass that back.  If not, the method will spawn an instance of the class constructor returned by the *spawn* option. This assumes the element passes all the "supports/matches" criteria.

I'm a little uncertain how important it is to provide for both ".get" and ".await".  I *think* using await automatically yields a microtask, even if there is no actual async call.  If that is not the case, I think we only need one of the two ("get") and just use the await keyword to be safe.

The whenResolved promise will only resolve when:

1.  The registering party specifies the name of the resolveKey in mountInfo
2.  The enhancement author defines a class that extends EventTaget, and does:

```JavaScript
const {resolvedKey} = mountInfo
this[resolvedKey] = true;
this.dispatchEvent(new Event(resolvedKey))
```

The whenResolved method would throw an error (catchable via try/catch with await or .catch() if using the more traditional promise approach) when the developer sets this.resolved = false;

The purpose of having this "whenResolved" feature is explained towards the end of this proposal.

> [!NOTE]
> I think it would be quite reasonable for these methods to accept an additional parameter where the registry and enhancement info can be passed in, and call the "inject" (subject to change) method on that registry when applicable, and throw an error if a conflicting enhancement has already been registered.  Not at all needed for day one.

## Reducing developer guilt and allowing for a nice API by formally endorsing attaching the spawned instance to the element's "enhancements" property gateway

A key config setting in the enhacementInfo, "enhKey," would cause the spawned instance to be attached at that name to the new, proposed "enhancements" property gateway that would be added to the Element prototype.

Use of the enhKey means that the developer will be responsible for avoiding name-spacing conflicts with other enhancements registered in the same registry. 

For example:

```JS
customElements.mount({
    //name of our "custom prop", accessible via oElement.enhancements[enhKey], 
    //which is where we will find an instance of the class defined below.
    enhKey: 'logger',
    baseAttr: 'log-to-console', //canonical name of our (base) custom attribute.
    spawn: class {
        constructor(el, {mountInfo}){
            const {baseAttr} = mountInfo;
            // in this example, base will simply equal 'log-to-console', 
            // but this code is demonstrating how to code defensively, so that
            // the party (or parties) responsible for registering the enhancement 
            // could choose to modify the name(s), either globally, 
            // or inside a scoped registry in a different file.
            enhancedElement.addEventListener('click', e => {
                console.log(
                       enhancedElement.getAttribute(`enh-${baseAttr}`)
                    || enhancedElement.getAttribute(base)
                ); 
            });
        }
    }
});
```

In the above example, we have two strings that we need to consider from the point of view of colliding with other enhancements (and with attributes of the (custom) elements themselves):  The name of the enhancement - "logger" - and the attribute(s) tied to it, if any:  'log-to-console'.  This proposal holds that the attributes for a single enhancement must share the same base, if help from the platform is desired.  Other enhancements can share that same base, and even share the entire  base-branch-leaf-prefix combo between different enhancements. It would result in multiple enhancements getting spawned. 

[TODO] Give a good example of two enhancements that would want to share the same 

> [!NOTE]
> Each specified enhKey must be unique within a registry.

There are some very strong use cases for the developer to go ahead and opt to name the enhancement, essentially making it more "public", even if it incurs a bit of a "burden" due to the registry uniqueness requirement:

1.   The name will be useful anytime we are outside the domain of JavaScript -- in particular referencing enhancement properties from declarative HTML (server-rendered) Markup.
2.  Accessing the properties value / methods of the instance is more natural to the developer using traditional dot (".") nested access, and feels less clunky.  
3.  Some protocols for distinguishing between "safe", declarative, side-effect-free code versus imperative code may use the existence of parenthesis as the defining characteristic for separating the two.

If no enhKey is specified by the parties registering the enhancement in the registry, I think the platform should still provide a less elegant mechanism to access the spawned instance, and in fact was already provided above:

```JavaScript
const enhancementInstance = oElement.enh.get(mountInfo);
```

### Does it make sense to define a spawn mount with no enhKey and no base attribute and no whereElementMatches?

I think it does.  In this case, the platform would be providing something like JQuery's [data](https://api.jquery.com/data/) feature, but more powerful (supporting one per enhancement).
 
## A helper property to make setting properties easier.

In addition to the three methods above, the enhancements property would contain a lazy property, "set", which would return/instantiate a proxy if invoked/retrieved, which can then dynamically return an instance of the enhancement, if the enhancement has already attached.  If it hasn't attached yet, it will return either an empty object, or whatever value has been placed there previously.

> [!Note]
> This will only work for enhancements where the enhKey property is specified.

This would allow consumers of the enhancement to pass property values (and only property values) ahead of the upgrade (or after the upgrade), so that no "await" is necessary, nor any imperative looking code:

```JavaScript
oElement.enh.set.steelEnhancer.carbonPercent = 0.2;
```

These value settings would either get applied directly to oElement.enh.steelEnhancer if it has already been attached.  Or, if it hasn't been attached yet, the browser would set (or merge) the value into the property, and begin attaching the enhancement in the background:

```JavaScript
if(oElement.enh.steelEnhancer=== undefined) {
    //get enhancement info for property "steelEnhancer"
    //if found:
    {
        //is steelEnhancerMountInfo.spawn a class constructor?
        {

        }
        //else it must be an async lazy dynamic loader to a class constructor
        {
            oElement.enh.steelEnhancer = {};
            //attach asynchronously in the background
        }
    }
    //else
    {
        oElement.enh.steelEnhancer = {};
    }
    
} 
oElement.enh.steelEnhancer.carbonPercent = 0.2;

```

In the case of an async attach definition, the property value object would sit there, ready to be absorbed into the enhancement in the constructor, which could happen right away if already loaded, or whenever the customElements.whenMounted is resolved for this enhancement.

The attaching in the background convenience would only be possible if the developer has already registered enhKey = "steelEnhancer" in an applicable registry.

Due to this lazy property, set, being a proxy, the convenience of this approach likely comes at a cost.  Proxies do impose a bit of a performance penalty, so a framework or library that uses this feature would be well-advised to add a little bit of nuance to the code, to set properties directly to the enhancement once it is known that the enhancement has attached.  For example, use this property the first time setting a property value, and then more directly for subsequent times.  Or, alternatively, implement the identical logic described above within the library code, thus avoiding the use of this special property altogether.

## Symbolic Prop Shortcuts and support for dependency injection

We can skip the step of either passing in mountInfo, as well as referencing the optional enhKey, which may not be totally stable when mixing together multiple third party libraries.  If instead, we want to "jump to the chase" and set (presumably) stable properties of the instance, we can do so as follows:

```JavaScript
export const isHappy = Symbol.for('TFWsx0YH5E6eSfhE7zfLxA');
class MyEnhancement extends ElementEnhancement(EventTarget){
    get isHappy(){}
    set isHappy(nv){}
}

export const isMellow = Symbol.for('BqnnTPWRHkWdVGWcGQoAiw');
class YourEnhancement extends ElementEnhancement(EventTarget){
    get isMellow(){}
    set isMellow(nv){}
    get madAboutFourteen(){}
    set madAboutFourteen(nv){}
}

//Here's where the dependency injection mapping takes place
const customEnhancementRegistry = new CustomEnhancementRegistry;
customEnhancementRegistry.define([
    {
        map: {
            [isHappy]: 'isHappy'
        },
        spawn: MyEnhancement
    },{
       enhKey: 'mellowYellow',
       map: {
           [isMellow]: 'isMellow'
       },
       spawn: YourEnhancement
    }
]);
//end of dependency injection

const divContainer = document.createElement('div', {customEnhancementRegistry});
const inputEl = document.createElement('input');
inputEl.assignGingerly({
    [isHappy]: true,
    [isMellow]: true,
    '?.style.height': '40px',
    '?.enh?.mellowYellow?.madAboutFourteen': true
});
inputEl.set[isMellow] = false;
divContainer.appendChild(inputEl);
document.body.appendChild(divContainer);
```

The platform would search the registry for any enhancements that has a mapping with a matching symbol of isHappy and isMellow, and if found, instantiate the instance if needed, then set the property value.

The suggestion to use Symbol.for with a guid, as opposed to just Symbol(), is based on some negative experiences I've had with multiple versions of the same library being referenced, but is not required.  Regular symbols could also be used when that risk can be avoided.

## Spawning/Attaching based on presence of attributes and/or whereElementMatches

If any one of the  (enh-*) attributes matching the pattern of base/branch/leaf is found on an element in the live DOM tree, this would cause the platform to instantiate an instance of the corresponding class, assuming other conditions are also met (whereElementMatches, whereInstanceOf).

If no base is specified (and thus attrTree is not applicable), but "whereElementMatches" is specified, this would also cause the spawn and/or do reactions.  Likewise with "whereInstanceOf".  Perhaps in the latter case, the prototype can be modified, assuming no additional conditions (low, low priority).

I also suggest that it would be great if, during template instantiation supported natively by the platform, the platform can do whatever helps in achieving the most efficient outcome as far as recognizing these custom attributes.  One key feature this would provide is a way to extend the template instantiation process -- plug-ins essentially.  Especially if this means things could be done in "one-pass".  I don't claim any expertise in this area.  If the experts find little to no performance gain from this kind of integration, perhaps it is asking too much.  Doing this in userland would be quite straightforward (on a second pass, after the built-in instantiation has completed).

My suspicion is that the best performing solution would be to do a "template compilation step", and convert all the attributes (with an opt-out capability) to a set of JavaScript instructions keyed off of the "coordinates" of the node, as implemented [here](https://github.com/bahrus/spawning).

Another integration nicety I would like to see supported by built-in template instantiation is to be able to bind sub objects from the host to the enhancements gateway.  So for example:

```html
<input :enhancements.steelEnhancer.carbonPercent={{carbonPercent}} >
```

would work (using FAST web component syntax here.  Lit uses a . instead).


What follows is going out into uncharted territories, discussing how this proposal might integrate into a work-in-progress spec (template instantiation) that hasn't been fully fleshed out.


##  Mapping elements contained in the template to enhancement classes during template instantiation.

Suppose we have a template that we want to use for repeated template instantiation:

For example:

```html
<template>
    <div>
        <span></span>
        <button></button>
    </div>
    <section>
        <span></span>
        <button></button>
    </section>
<template>
```

Now the developer defines a class that provides the ability to keep track of how many times a button has been clicked, and that can broadcast that count to other elements near-by.  The class extends ElementEnhancement.

An example, in concept, of such a class, used in a POC for this proposal, can be [seen here](https://github.com/bahrus/be-counted), just to make the concept less abstract (the POC will not exactly follow what this proposal will outline as far as defining and registering the class), but basically, for server-rendered progressive enhancement, the **server-rendered** HTML this class expects would look as follows:

```html
<template>
    <div generateids>
        <span #></span>
        <button be-counted='{
            "transform": {
                "# span": "value"
            }
        }'></button>
    </div>
    <section generateids>
        <span #></span>
        <button be-counted='{
            "transform": {
                "#{{span}}": "value"
            }
        }'></button>
    </section>
<template>
```
 

Note that the enhancement class corresponding to this attribute may specify a default count, so that the span would need to be mutated with the initial value,  either while it is being instantiated, if the custom enhancement has already been imported, or in the live DOM tree.  The decision of whether the enhancement should render-block is, when relevant, up to the developer.  If the developer chooses to import the enhancing class synchronously, before invoking the template instantiation, then it will render block, but the span's text will be already set when it is added to the DOM tree.  If the developer imports the class asynchronously, then, depending on what is in cache and other things that could impact timing, the modification could occur before or after getting appended to the live DOM tree.  Ideally before, but often it's better to let the user see something than nothing.

The problem with using this inline binding in our template, which we might want to repeat hundreds or thousands of times in the document, is that each time we clone the template, we would be copying that attribute along with it, and we would need to parse the values.

Because this proposal is advocating that the MountInfo interface that is passed into the define method has enough information to map from the attribute to the parsed properties, it's my view that this would allow template instantiation supported by the platform (or userland implementations) to avoid unnecessary string parsing, by making judicious use of caching.

Again, I think [this](https://github.com/bahrus/spawning) is probably optimal way.

## Support for connected/disconnected callback of the element being enhanced?

When should the enhancement be purged from memory?


This is an area likely to require some critical feedback from browser vendors, but I will nevertheless express some thoughts on the matter.

This proposal is hoping that the browser engineers can figure out a way to not count these spawned enhancements when doing reference counting, and when the reference count minus spawned enhancements reaches 0, call the (configurable) dispose method of the function prototype / class and purge.


One time it definitely would **not** be purged is if the (enh-*) attributes, if present, are removed from the enhanced element, since as we've discussed, the custom attribute aspect is only one way to attach an enhancement.  A developer may want to remove the attributes to reduce clutter, optimize for template instantiation, or before transferring to another Shadow DOM realm to avoid unexpected side effects of being transported in.

I could see scenarios where the enhancement would want to know that its host has been disconnected and (re) connected.  So the custom enhancement should have a way of being notified that this transfer took place.

One way to do this is if the platform adds an event that can be subscribed to for elements:  Elements currently have a built-in property, "isConnected".  It would be great if the elements also emitted a standard event when the element becomes [connected and (possibly another)](https://github.com/whatwg/dom/issues/533) [event](https://twitter.com/jaffathecake/status/1521023821003767808) or [signal](https://github.com/whatwg/dom/issues/1296) when it becomes disconnected.

Another way would be to allow developers to specify a key name for the connected and disconnected callbacks.  I prefer the previous solution.

## How to programmatically dispose of an enhancement

I'm encountering a small number of use cases where we want enhancements to "do its thing", and then opt for early retirement.  The use cases I've encountered this with is primarily focused around an enhancement that does something with server-rendered HTML, which then goes idle afterwards, possibly to be replaced by a different kind of enhancement during template instantiation.  So I think it should be possible to do this via:

```JavaScript
const detachedEnhancement = await oElement.enh.forget(mountInfo);
```

I think we would want this to remove the associated attribute(s) also, if applicable (which is a little messy, because other enhancements may share the base or even base/branch/leaf combos as stated above, so maybe not).

## How an enhancement class indicates it has hydrated 

Earlier in this document, I mentioned a feature built in to the base class, that indicates a state of "resolved".  Here's the explanation for one use case:

In many cases, multiple enhancements are so loosely coupled, they can be run in parallel.

However, suppose we want to apply three enhancements to an input element, each of which adds a button:

1.  One that opens a dialog window allowing us to specify what type of input we want it to be (number / date, etc).
2.  One that allows us to clone the input element.
3.  One that allows us to delete the input element.

If the three enhancements run in parallel, the order of the buttons will vary, which could confuse the user.

In order to avoid that, we need to schedule them in sequence.  This means that we need a common way each enhancement class instance can signify it either succeeded, or failed, either way you can proceed.  That is why we should have this ability to specify whether the hydration has completed. 

I have a heavy suspicion that as the platform builds out template instantiation and (hopefully) includes something close to this solution as far as plug-in's, there will arise other reasons to support this feature.

But for now, the way this feature can be used is with a bespoke custom enhancement, such as [be-promising](https://github.com/bahrus/be-promising#be-promising).

## Support for "defer-[base]"

Along the lines of the discussion above about loading enhancements in a predictable sequence, I've encountered some compelling use cases where we want to "defer" loading of a server-rendered attribute-based enhancement.  For example, a peer web component may want to tap into events that an enhancement fires, and not miss any events it may fire prior to the peer web component becoming upgraded.

A related requirement has been identified by the web component community, called [defer-hydration](https://github.com/webcomponents-cg/community-protocols/blob/main/proposals/defer-hydration.md).  While the use case is a bit different, the end requirement is quite similar.

To support this important use case, we propose the pattern:  defer-[base].  Until the attribute is removed from the element, the get / await / whenResolved methods discussed above cannot proceed:


```html
<form
    defer-be-reformable
    be-reformable='{
        "baseLink": "newton-microservice",
        "path": "api/v2/:operation/:expression",
    }'
>
    <label for=operation>
        Operation:
        <input :operation value=integrate>
    </label>
    
    <label for=expression>
        Expression:
        <input :expression value="x^2">
    </label>
    
    <noscript>
        <button type=submit>Submit</button>
    </noscript>
</form>

<for-fetch></for-fetch>
```

The attribute could also be used for purposes of disabling functionality within the element.

## Namespacing events

Because custom enhancements extend the EventTarget, it is quite possible (and probably optimal) to subscribe to events directly from the enhancement, as we've seen above with the "resolved" event.

However, I've encountered quite a few use cases where we want the enhancement to dispatch an event from the element it adorns.

To be able to distinguish that:

1.  The event was initiated by an enhancement
2.  Uniquely identify which enhancement issued the event within a ShadowDOM realm

I propose:

The base Event object gets an additional built in property:  "mountInfo", which is where we pass in the mountInfo registry definition.


## How custom elements can opt in

So far we've been discussing using enhancements to enhance *third party* elements, built-in or custom.  Some of these enhancements/behaviors would provide functionality that is quite perpendicular to what the custom element provides -- e.g. logging, persistence, binding.  Others will be more aligned with the functionality the element provides.

But I think some of the infrastructure behind this proposal could be useful to [first party developers](https://github.com/WICG/webcomponents/issues/814#issuecomment-3392840225) as well.  In particular, it would be great if we empower custom element authors with:

1. ...the ability to break down one behavior into various aspects via quite semantic attributes, and roll them up into one behavior/enhancement.
2. ...dynamically name-spacing support for property names within registries, or, in contrast... 
3. ...leveraging the support of enhancements, but with more locked down prototype based properties
3. ...supporting lazy-loading of such functionality as needed.
4. ...declarative mapping of functionality similar to dependency injection.

... while leveraging the exact same class definition as above.

### Use cases?

One of the "slam dunk" use cases that come to mind is applying behaviors/enhancements that have proven really useful when applied to built-in elements, but now apply these same libraries to custom elements that aim to emulate the same built-in abilities.  The ability to acquire the traits of built-in elements may be becoming more achievable as the platform provides said behaviors [via internals](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/ElementInternalsType/explainer.md).

Another use case is for robust web components that do far more than just being a cool button or tab control -- components that provide business functionality, including managing domain objects.  These components would benefit from specializing, and breaking down the large component into smaller sub units.  Some sub units could utilize store libraries, like MobX stores, for example.

And another important use case is providing a nicely structured API to implement built-in behaviors provided by the platform, and orchestrating the handoff of access to internals and private data.

One way a first-party component could adopt a first-party or third-party behavior/enhancement would be to do this by simply spawning and/or attaching the enhancement as discussed above, "at arms length".

But suppose a behavior/enhancement's functionality is core to a custom element's mission, or close enough for government work? Suppose the custom element wants to provide key information that is not accessible from outside, like private data and/or the internals?  And/or suppose the custom element wants to nail down the name of the "custom prop" directly onto its namespaced object / prototype chain, so dependencies can leverage TypeScript and not have to be so vigilant about collisions between different (versioned) libraries that use the same name (beyond vigilance towards the shadow scoped name of the element itself).  As well as pinning down the (base) attribute(s) tied to the enhancement?

I propose a significant amendment to this proposal, support for...:

# Custom Element Features

## Dynamically, imperatively attaching a feature

Here, no "enh-" prefix is required for attributes.  In fact, enh- prefixed attributes will be ignored.

```TypeScript
interface PhotoTaker{}
class MyPhotoTaker extends CustomElementFeature(EventTarget) implements PhotoTaker{
    constructor(customElement: HTMLElement, {info}: {info: MyPhotoTakerFeatureInfo}){
        super();
        ...
    }
}
interface BadgeMaker{}
class YourBadgeMaker extends CustomElementFeature(EventTarget) implements BadgeMaker{
    constructor(customElement: HTMLElement, {info}: {info: YourPhotoTakerFeatureInfo}){
        super();
        ...
    }
}

interface ClubMemberProps {
    photoTaker:  PhotoTaker | undefined;
    badgeMaker:  BadgeMaker | undefined;
}

class ClubMember extends HTMLElement implements ClubMemberProps{
    constructor(){ 
        super();
        this.addEventListener('feature-added', e => {
            ...
        });
        this.attachInternals()
            .attachFeature<PhotoTaker, ClubMember>(MyPhotoTakerMountInfo)
            .toInstance(this)  //toInstance should expect an instance of ClubMember, TypeScript definers
            .atProp('photoTaker') // atProp should expect a keyof ClubMember for its parameter, TypeScript definers
            .attachFeature<BadgeMaker, ClubMember>(YourBadgeMakerMountInfo)
            .toInstance(this)
            .atProp('badgeMaker');
    }

    photoTaker: PhotoTaker;

    badgeMaker: BadgeMaker;

}

```

I think this would allow for testable Mock Objects, especially if these methods (especially .attachFeature) is/are made overridable by a super class.

This code can be run at any time, not just in the constructor.

The type definition for FeatureInfo would closely resemble that of MountInfo, but some fields of MountInfo don't quite make sense in this context, and other fields may make more sense in the context of features, like the last two:

```TypeScript
type Feature = {new(): CustomElementFeature} | () => Promise<{new(): CustomElementFeature}>
interface FeatureInfo {
    spawn: Feature
    base?: Base
    branches?: Branches
    leaves?: Leaves
    map: {key: AttrCoordinates: AttrHandlerInfo}
    //only attach the feature if the base attribute is present on the element
    attachOnBase?: boolean
    //Instantiate the feature immediately when the custom element is created.
    //Applicable to the declarative support described below.
    loadEagerly?: boolean
}
```

No support for supportInstanceTypes, supportedCssMatches is needed, for example. 

## Support for adding features to the custom element prototype declaratively with dependency injection

In addition, we should provide for a way to declaratively attach when we don't need to be so dynamic:

```TypeScript
class ClubMember extends HTMLElement {
    
    photoTaker: PhotoTaker | undefined;
    badgeMaker: BadgeMaker | undefined;
}

customElementRegistry.define('club-member', ClubMember, {
    features: {
        photoTaker: MyPhotoTakerFeatureInfo,
        badgeMaker: YourBadgeMakerFeatureInfo
    }
});

```

Here the platform would attach the feature in the base HTML class (preferably in the constructor, I think) using the afore mentioned methods, if loadEagerly is set to true.  If loadEagerly is false (the default), the platform will only instantiate it when it finds a matching attribute or detects property access in ways that have been described above.

Both ways of attaching the enhancement would result in dispatching an event, '"featureadded", allowing the userland code to pass in such things as private data and element internals to the enhancement.

## Serving dual roles

I think it should (almost?) always be possible to use the same class to support both ElementEnhancements *and* CustomElementFeatures -- simply wrap both mixins:

```JavaScript
class MyPhotoTaker extends ElementEnhancement(CustomElementFeature(EventTarget))
```

## Differences between the two mixins

I think it makes sense for the CustomElementFeature to have a standard, reserved setter (similar to connectedCallback) for passing in the custom element internals:

```TypeScript
interface CustomElementFeature{
    resolved?: boolean
    set internals?(elementInternals: HTMLElementInternals)
    channelEvent(event: Event, options: EventInitOptions)
}
```

# But wait, there's more!!!

[Support for private features](https://github.com/bahrus/custom-enhancements?tab=readme-ov-file#support-for-private-features)

[Support for nested features](https://github.com/bahrus/custom-enhancements?tab=readme-ov-file#support-for-nested-features)

[Support for Prop-Passthrough's to custom element features](https://github.com/bahrus/custom-enhancements?tab=readme-ov-file#support-for-prop-passthroughs-to-custom-element-features)

[Support for a view model DOM fragment manager tied to the itemscope attribute.](https://github.com/bahrus/custom-enhancements?tab=readme-ov-file#support-for-a-view-model-dom-fragment-manager-tied-to-the-itemscope-attribute)

## Support for private features

What if the feature need not expose any public interface?

Simply prefix the key with a #.  This will cause the FeatureAdded event to fire, without attempting to attach the feature, allowing the custom element to assign the feature to a private property if needed.

```TypeScript
class ClubMember extends HTMLElement {
    
    photoTaker: MyPhotoTaker | undefined;
    badgeMaker: YourBadgeMaker | undefined;
}

customElementRegistry.define('club-member', ClubMember, {
    features: {
        '#photoTaker': MyPhotoTakerFeatureInfo,
        '#badgeMaker': YourBadgeMakerFeatureInfo
    }
});

```

Done!

The next two asks are probably the lowest in the priority list, as I can see it being a hard sell.  They've also not yet been vetted with an actual implementation anywhere that I know of.

## Support for nested features

For really large components that make use of many features / behaviors / enhancements / whatever, it would be nice to be able to group them into "property bag" categories, so that the api becomes more scalable and manageable (unlike the platform).

I think given that these categories would tend to be fairly stable over time, and not have much custom logic if any,  we could leave much up to the developer to "hard code" these property bag classes without the benefit of dependency injection:

```TypeScript
class MyPhotoTaker extends CustomElementFeature(EventTarget) implements MyPhotoTaker{}
class MyBadgeMaker extends CustomElementFeature(EventTarget) implements MyBadgeMaker{}

// Properties could be added to the prototype, probably producing better performance
class RegistrationFeatures {
    myPhotoTaker: MyPhotoTaker | undefined;
    myBadgeMaker: MyBadgeMaker | undefined;
}

class ClubMember extends HTMLElement{
    #registrationFeatures: Features;
    get registrationFeatures(){
        return this.#registrationFeatures;
    }
    
    constructor(){
        this.#registrationFeatures = new RegistrationFeatures();
    }
}

customElementRegistry.define('club-member', ClubMember, {
    features: {
        '?.registrationFeatures?.photoTaker': MyPhotoTakerFeatureInfo,
        '?.registrationFeatures?.badgeMaker': YourPhotoTakerFeatureInfo,
    }
})
```

So the platform should be able to assume that whenever it comes time to attach the features, the RegistrationFeatures instance will already be assigned to the registrationFeatures property, so there is no ambiguity about how to instantiate it.  I.e. we limit the dependency injection to that last property path (photoTaker/badgeMaker).


## Support for Prop-Passthrough's to custom element features

Unfortunately, some of the questionable design decisions made long ago behind DOM API's are likely to percolate to questionable design decisions to custom elements, with the advent of exposing platform behaviors.  I think the platform would have benefited from structuring functionality a little more, as it did with styles (and unlike aria, for example).  In particular, to emulate the built-in button, developers will want to add [properties to the top level](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/ElementInternalsType/explainer.md):

```TypeScript
class CustomButton extends HTMLElement {
    static buttonActivationBehaviors = true;

    constructor() {
        super();
        this.internals_ = this.attachInternals();
    }

    get commandForElement() {
        return this.internals_.commandForElement ?? null;
    }

    set commandForElement(element) {
        this.internals_.commandForElement = element;
    }

    get command() {
        return this.internals_.command ?? '';
    }

    set command(value) {
        this.internals_.command = value;
    }
}
customElements.define('custom-button', CustomButton);
```

Because there may be a growing number of such built-in behaviors that the developer will want to emulate, developers will naturally and understandably flock toward the mixin model, to avoid unnecessary clutter, versus more compositional approaches, just to seem more "native-like", indistinguishable from built-in buttons.  This could, in my view, encourage [problematic dependency anti-patterns](https://legacy.reactjs.org/blog/2016/07/13/mixins-considered-harmful.html).

I harbor no illusions that developers will be unanimously abandoning brittle mixins in favor of this platform nicety described below.  The amount of custom code that the developer would inject to achieve this built-in functionality in the property getters / setters would start out, at least, to be quite small. Thus the benefits of DI (testing, loose coupling, etc) would be small.  Still, I think it is worthwhile considering offering this ability, because I could see interest growing in this ability in scenarios where the amount of custom code that gets embedded in the getters / setters increases, so the benefits begin to outweigh the "costs".

### Scenario I - Static, declarative approach

In the above example, the "internals_" property is actually made publicly accessible, which is how MDN documents this feature.  Safari makes [it private](https://webkit.org/blog/13711/elementinternals-and-form-associated-custom-elements/) in their demonstrations.  Going with the latter approach, and adopting the more static, declarative way of attaching features, suppose we did this:


```TypeScript
class Behaviors {
    Command: ICommand
}
class CustomButton extends HTMLElement {
    static buttonActivationBehaviors = true;
    #internals;

    #behaviors: Behaviors;
    get behaviors() {
        return this.#behaviors;
    }

    constructor() {
        super();
        this.#behaviors = new Behaviors;
        this.#internals = this.attachInternals();
        //optional -- can skip this is theres no need to be untrusting
        //of how your users register your component.
        this.addEventListener('feature-added', e => {
            //optional, if there's any reason to be wary of trusting how
            //the custom was registered (I don't see a scenario where that would be the case)
            if(!(e.target instanceof CommandCustomElementFeature)) return; 

            //alternatively, this can be done by the platform, 
            //based on a static autoInjectInternals setting
            //as shown below
            e.target.internals = this.#internals;
        })
    }

    //optional
    static autoInjectInternals: true

}

interface CustomButton {
    behaviors: {
        command: CommandInterface
    }
}

//or a mixin would work well also, I think
class CommandCustomElementFeature extends CustomElementFeature(EventTarget) implements CommandInterface{
    #internals;
    constructor(customElement: HTMLElement, commandFeatureInfo: FeatureInfo){
        super();
        this.channelEvent(new FeatureAddedEvent()); //FeatureAddedEvent would be a platform provided event
    }

    get commandForElement() {
        return this.#internals.commandForElement ?? null;
    }

    set commandForElement(element) {
        this.#internals.commandForElement = element;
    }

    get command() {
        return this.#internals.command ?? '';
    }

    set command(value) {
        this.#internals.command = value;
    }

    //standard name recognized by the platform
    set internals(nv){
        this.#internals = nv;
    }
}
customElements.define('custom-button', CustomButton, {
    features: {
        '?.behaviors?.command': {
            spawn: CommandCustomElementFeature,
            baseAttr: 'command',
            map: {
                '0.0': {
                    instanceOf: String,
                    mapsTo: 'command'
                },
            },
            passThrough: ['command', 'commandForElement']
            
        }
    }
});
```

What the platform would do with the passThrough setting:
   
Add simple pass-through properties 'command' and 'commandForElement' to the top level of the CustomButton class, with  setters which would spawn the CommandCustomElementFeature if needed, and pass the values through to the same named property of the custom element feature class instance, however deeply we specify as far as the path.

I think it's okay to use the "behaviors" property here, for custom elements only, since "HTMLElement" is kind of like "Object" in this context and I can't see it interfering with future built in behaviors added to higher order elements, nor cause developer confusion.

## See what we did there?

Other than the entirely optional double-checking in the feature-added event handler, the top level custom element has fully, 100% delegated implementation of the command behavior.  It doesn't even need to add "command" to the list of observed attributes, because the feature is taking care of watching for that attribute.  It's really "clean", and can focus on whatever top level functionality it needs to focus on that makes the custom button "custom".


# Support for a view model DOM fragment manager tied to the itemscope attribute.

There are many scenarios where it makes sense to have one "central" element manager, that frameworks / libraries can expect, that manages the data and/or view model and/or binding and/or event handling for a DOM element and its children, plus extensions of that element that are linked to it via the itemref attribute:

- Scenarios where we can't wrap the element inside a custom element.  
- Scenarios where we need to manage the light children of a web component that uses ShadowDOM
- Scenarios where we want to manage a DOM fragment that was generated from a looping library api. 


Unlike the other enhancements that this proposal supports, these element managers would not be enhancing the behavior of the element it provides, but rather focused squarely on binding and hydrating the light children of the element it adorns.  

This proposal is advocating enhancing the itemscope attribute, so that it can optionally specify the name of a registered class, instances of which frameworks could then easily pass values to, or invoke methods, or dispatch events to.  These classes would need very little in terms of integration with the DOM API's, as their focus is meant to be on "business domain logic" -- no support for owned attributes is needed, for example.  Nor specifying any restrictions of which types of elements that we are targeting.  These classes would be so generic in manner that the element type is largely immaterial.

So I am advocating no fewer than three "registries", as far as categories of classes / function prototypes:

1.  Custom Elements, that extends HTMLElement (already built into the browser)
2.  Custom Enhancements, that extends ElementEnhancement (the bulk of this proposal)
3.  Itemscope managers, that extends ItemscopeManager (the addendum to this proposal we are discussing now).  

So, just to provide a sample API to make things less abstract, suppose the API for registering the Itemscope managers looks like this:

```JavaScript
document.body.registerItemScopeManager('my-item', class extends ItemscopeManager(Object){
    get ssn(){
        ...
    }
    set ssn(val){
        ...
    }
    get name(){
        ...
    }
    set name(val){
        ...
    }
});
```


Then libraries could integrate with these managers.  For example, with lit-html:

```JavaScript
html`
<table>
    <thead><th>Name</th><th>SSN Number</thead>
    <tbody>
${myList.map(item => html`
    <tr itemscope=my-item .tbd=${item}>
        <td itemprop=name>
            ${item.name}
        </td>
        <td itemprop=ssn>${item.ssn}</td>
    </tr>
`)}
    </tbody>
</table>
`
```

What this would do:  

1.  Before instantiating the "tbd" property of the tr element, merge whatever properties were passed to the "tbd" placeholder.
2.  After the instantiation, "setting" the property to an object would **not** replace the class or function prototype instance with the object, but rather the setter for "tbd" would interject, and do an Object.assign of the passed in object into the custom element/enhancement instance.

What the real name of "tbd" should be is completely open in my mind.  Nothing jumps out at me as the "correct" answer.  Names that would make sense to me are:  "host", "vm", "viewModel", "scope", "ism" or "ish" -- short for itemscope host.  I guess I'm leaning towards the latter -- it is short, and is kind of a play on "is".

## Support for lists and the itemscope manager

In many cases, what we need to bind the view to is not an "expando" type object, but rather an array of objects (i.e. lists).  In some cases we would want both to be supported ("expando" type properties but also an iterator interface). The discussion above doesn't make much sense --  merging in an object (via object.assign or something more powerful than object.assign) -- in this scenario where we have a list of objects or primitives to "pass in".  

So it seems reasonable to amend the  discussion above so that if the data that needs be passed into the tbd property is an array, to automatically upgrade the class or function prototype in the registration method:

```JavaScript
ctr.prototype[Symbol.iterator] = function () {
    var index = -1;
    var data = this[secretKey];
    return {
        next: function () {
            return {
                value: data === undefined ? undefined : data[++index],
                done: data === undefined || !(index in data)
            };
        }
    };
};
```

## Channeling events

I initially thought these itemscope managers could be function prototypes, or classes, and leave that decision up to the developer, not inheriting anything particular from the platform.  But I think the case for supporting the same "channelingEvent" method as above, that dispatches a distinguishable event from the adorned element that can bubble up the tree, is strong enough to advocate for a base class.




