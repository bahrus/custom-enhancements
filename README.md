# Mounting built-in enhancements, custom element features, itemscope managers

## Author

Bruce B. Anderson

PR's, Issues [welcome](https://github.com/bahrus/custom-enhancements)

Last update: May 2026

This is [one](https://github.com/whatwg/html/issues/2271) [of](https://eisenbergeffect.medium.com/2023-state-of-web-components-c8feb21d4f16) [a](https://github.com/WICG/webcomponents/issues/1029) [number](https://github.com/WICG/webcomponents/issues/727) of interesting proposals, one of which (or some combination?) can hopefully get buy-in from all three browser vendors. This proposal borrows heavily from the others.

A working polyfill of this proposal is available at [assign-gingerly](https://github.com/bahrus/assign-gingerly) (for the property assignment, dependency injection, and registry APIs) and [mount-observer](https://github.com/bahrus/mount-observer) (for automatic DOM discovery and enhancement attachment).  These polyfills is hevily influenced, and  work best within the current constraints of what is easily available to developers, and could probably be streamlined when implementing directly in a browser setting.

# Custom Attributes For [Simple Enhancements](https://www.w3.org/TR/design-principles/#simplicity)

Say all you need to do is to create an isolated behavior/enhancement associated with an attribute — say "log-to-console". It enhances elements adorned with that attribute, logging the value of the attribute to the console when the element is clicked:

Informal, without registering anything:

```JS
document.mount({
    matching: '[log-to-console]',
    do: (el) => {
        el.addEventListener('click', e => {
            console.log(e.target.getAttribute('log-to-console'));
        });
    }
});
```

```HTML
<svg log-to-console="clicked on an svg"></svg>
<div log-to-console="clicked on a div"></div>
<some-custom-element enh-log-to-console="clicked on some custom element"></some-custom-element>
```

Formally registering the enhancement declaratively in custom element registry:

```html
<script type="emc">
{
    "matching": "button",
    "enhConfig": {
        "spawn": "./button-enhancement.js",
        "enhKey": "fancyButton",
        "withAttrs": {
            "base": "variant"
        }
    }
}
</script>
```

emc stands for Element Mount Configuration.

The enhConfig ends up getting registered in customElementRegistry.enhancementRegistry

More programmatic ways of registering enhancements are documented in the assign-gingerly polyfill.

## Why the custom element registry?

The platform indicates this is the best way to organize such things. The `CustomElementRegistry` already provides scoping via Shadow DOM and even just a parent DOM element, which enhancements would greatly benefit from as far as avoiding namespace collisions.

## Why "mount"?

It ties in with [mount-observer](https://github.com/bahrus/mount-observer), which provides the DOM discovery primitive. The platform would automatically detect elements matching the enhancement criteria and invoke the spawn.

## Custom Property Name-spacing via `enh`

This proposal adds a reserved property `enh` to the Element prototype — a namespace gateway for enhancement instances, similar to `dataset` for data attributes:

```JavaScript
oInput.enh.myEnhancement.foo = bar;
oMyCustomElement.enh.yourEnhancement.bar = foo;
```

The `enh-*` attribute prefix is reserved for enhancement attributes on custom elements (to avoid conflicting with the custom element's own attributes). For built-in elements, dashed attribute names without the prefix are sufficient.

For full details on the `enh` gateway API, see [assign-gingerly: Enhancement Registry](https://github.com/bahrus/assign-gingerly#enhancement-registry-addendum-to-the-custom-element-registry).

## API Shape


```TypeScript
interface EnhancementConfig<T = any> {
    // The class to instantiate for this enhancement
    spawn: { new(element: Element, ctx: SpawnContext, initVals?: Partial<T>): T };
    
    // Optional: attribute patterns for declarative initialization
    withAttrs?: AttrPatterns<T>;
    
    // Optional: property name on element.enh for public access
    enhKey?: string | symbol;
    
    // Optional: symbol-to-property mappings for dependency injection
    symlinks?: { [key: symbol]: keyof T };
    
    // Optional: restrict to specific element types
    whereInstanceOf?: Function[];
    
    // Optional: restrict to elements matching a CSS selector
    whereElementMatches?: string;
    
    // Optional: lifecycle method names
    lifecycleKeys?: true | { dispose?: string; resolved?: string };
    
    // Optional: allow unprefixed attributes for custom elements
    allowUnprefixed?: string | RegExp;
}
```

For the full type definitions, see [assign-gingerly types](https://github.com/bahrus/assign-gingerly/blob/baseline/types/assign-gingerly/types.d.ts).

## Attribute Parsing with `withAttrs`

Enhancements can declaratively map element attributes to constructor `initVals`. The `withAttrs` configuration defines a base attribute name and maps sub-attributes to typed properties:

```TypeScript
customElementRegistry.enhancementRegistry.push({
    enhKey: 'intlFormatter',
    spawn: IntlFormatterEnhancement,
    withAttrs: {
        base: 'be-intl',
        weekday: '${base}-weekday',
        //unnecessary:  this is what happens by default
        _weekday: { instanceOf: 'String', mapsTo: 'weekday' },
        year: '${base}-year',
        month: '${base}-mount',
        day: '${base}-day'
    }
});
```

```html
<time lang="ar-EG" datetime="2011-11-18T14:54:39.929Z"
    be-intl-weekday="long" be-intl-year="numeric" 
    be-intl-month="long" be-intl-day="numeric">
</time>
```

Note that by using template substitution of previously defined attribute stems, we can easily define a tree like structure of attributes that can make to a single class with matching (optionally nested) properties.

The platform parses these attributes into an `initVals` object passed to the constructor:
```javascript
// initVals = { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' }
```

For full attribute parsing documentation, see [assign-gingerly: withAttrs](https://github.com/bahrus/assign-gingerly#custom-element-parsing-with-withattrs).

## Symbolic Dependency Injection

Enhancement properties can be set via Symbols, enabling type-safe dependency injection without string-based naming conflicts:

```JavaScript
import { isHappy } from './my-enhancement.js';
import { isMellow } from './your-enhancement.js';

const registry = customElements.enhancementRegistry;
registry.push([
    {
        symlinks: { [isHappy]: 'isHappy' },
        spawn: MyEnhancement
    },
    {
        enhKey: 'mellowYellow',
        symlinks: { [isMellow]: 'isMellow' },
        spawn: YourEnhancement
    }
]);

// assignGingerly resolves symbols through the registry
inputEl.assignGingerly({
    [isHappy]: true,
    [isMellow]: true,
    '?.enh?.mellowYellow?.madAboutFourteen': true
});
```

For full dependency injection documentation, see [assign-gingerly: Dependency Injection](https://github.com/bahrus/assign-gingerly#dependency-injection-based-on-a-registry-object-and-a-symbolic-reference-mapping).

## The `enh` Gateway

The `enh` property on Element provides methods for programmatic enhancement management:

```JavaScript
// Get or spawn an enhancement instance
const instance = oElement.enh.get(enhancementConfig);

// Dispose of an enhancement
oElement.enh.dispose(enhancementConfig);

// Wait for an async enhancement to resolve
const resolved = await oElement.enh.whenResolved(enhancementConfig);
```

Enhancements can also be accessed by `enhKey`:

```JavaScript
const instance = oElement.enh.get('logger'); // looks up by enhKey in registry
```

For full `enh` gateway documentation, see [assign-gingerly: Element Enhancement Gateway](https://github.com/bahrus/assign-gingerly#element-enhancement-gateway-enh).

## Automatic DOM Discovery (mount-observer)

The platform automatically detects elements matching enhancement criteria (attributes, CSS selectors, instance types) and spawns enhancements. This is handled by [mount-observer](https://github.com/bahrus/mount-observer), which provides:

- Efficient DOM observation via `MutationObserver`
- Lazy loading of enhancement code on demand
- Scoped registry support
- Conditional spawning based on `whereElementMatches` / `whereInstanceOf`

## Scoped Registries

Enhancements respect scoped custom element registries. Each Shadow DOM scope can have its own enhancement registry, preventing naming conflicts between different component libraries:

```JavaScript
const scopedRegistry = new CustomElementRegistry();
scopedRegistry.enhancementRegistry.push(myEnhancementConfig);

// Elements within this scope use the scoped registry
// via element.customElementRegistry (Chrome 146+)
```

## Backdrop: Why Enhancements?

The WebKit team raised valid concerns about extending built-in elements: public properties added by extensions could conflict with future platform additions. Yet the need to enhance existing elements in cross-cutting ways has been [demonstrated](https://aurelia.io/docs/templating/custom-attributes#simple-custom-attribute) [by](https://htmx.org/docs/) [countless](https://vuejs.org/v2/guide/custom-directive.html) [frameworks](https://alpinejs.dev/).

This proposal provides a structured alternative:

1. **Namespaced properties** via the `enh` gateway — no top-level property pollution.
2. **Registry-based discovery** — enhancements are scoped and don't conflict.
3. **Declarative attributes** — server-renderable, progressive enhancement friendly.
4. **Dependency injection** — via symbols and the enhancement registry.

# Custom Element Features

Custom Element Features extend the enhancement concept to first-party custom element development. They provide dependency injection for composable feature classes that are lazily instantiated on the custom element's prototype.

This enables:
- Breaking large components into smaller, testable units
- Dynamic dependency injection (swap implementations for testing)
- Lazy loading of feature code
- Sharing private data (ElementInternals) with features via `getSharedContext`
- Async feature loading with placeholder objects
- Property forwarding from the element to nested features

For full documentation, see [assign-gingerly: Custom Element Features](https://github.com/bahrus/assign-gingerly#custom-element-features).

### Quick Example

```JavaScript
import 'assign-gingerly/assignFeatures.js';

class PhotoTakerImpl {
    constructor(host, ctx, initVals) {
        this.host = host;
        if (initVals) Object.assign(this, initVals);
    }
    takePicture() { return '📸'; }
}

class ClubMember extends HTMLElement {
    static supportedFeatures = {
        photoTaker: {
            fallbackSpawn: PhotoTakerImpl,
            getSharedContext(instance) {
                return { internals: instance.#internals };
            }
        }
    }
    static featuresConfig = { lifecycleKeys: true }
    #internals;
    constructor() { super(); this.#internals = this.attachInternals(); }
}

customElements.assignFeatures(ClubMember, {
    photoTaker: { spawn: PhotoTakerImpl }
});
customElements.define('club-member', ClubMember);

const el = document.createElement('club-member');
el.photoTaker.takePicture(); // '📸' — lazily spawned on first access
```

# Itemscope Managers

Itemscope Managers provide a way to associate a class instance with elements that have the `itemscope` attribute. This enables frameworks and libraries to manage DOM fragments, looping constructs, and scenarios where custom element wrapping is not feasible.

For full documentation, see [assign-gingerly: Itemscope Managers](https://github.com/bahrus/assign-gingerly#itemscope-managers-chrome-146).

### Quick Example

```JavaScript
import 'assign-gingerly/object-extension.js';

class TodoItem {
    constructor(element, initVals) {
        if (initVals) Object.assign(this, initVals);
    }
    title = '';
    completed = false;
}

customElements.itemscopeRegistry.define('todo-item', { manager: TodoItem });
```

```html
<li itemscope="todo-item">
    <span itemprop="title"></span>
</li>
```

# Summary

This proposal advocates for three registry-based systems on `CustomElementRegistry`:

| Registry | Purpose | Polyfill |
|----------|---------|----------|
| `enhancementRegistry` | Third-party element enhancements with namespaced properties | [assign-gingerly](https://github.com/bahrus/assign-gingerly#enhancement-registry-addendum-to-the-custom-element-registry) |
| `featuresRegistry` | First-party custom element dependency injection | [assign-gingerly](https://github.com/bahrus/assign-gingerly#custom-element-features) |
| `itemscopeRegistry` | DOM fragment / view model managers | [assign-gingerly](https://github.com/bahrus/assign-gingerly#itemscope-managers-chrome-146) |

All three leverage scoped registries, lazy instantiation, and the `assignGingerly` utility for property merging and path-based assignment.
