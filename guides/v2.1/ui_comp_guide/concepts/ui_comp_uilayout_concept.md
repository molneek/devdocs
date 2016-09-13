---
layout: default
group: UI_Components_guide
subgroup: concepts
title: About the uiLayout service module
menu_title: About the uiLayout service module
menu_order: 13
version: 2.1
github_link: ui_comp_guide/concepts/ui_comp_uilayout_concept.md
---

## {{page.menu_title}}  
{:.no_toc}

* TOC
{:toc}

## Overview
The `uiLayout` service module is used to initialize UI components. The source file of the `uiLayout` module is `<UI_Module_dir>/view/base/web/js/core/renderer/layout.js`. 

In a [typical UI component's configuration flow]({{page.baseurl}}ui_comp_guide/concepts/ui_comp_config_flow_concept.html), the `uiLayout` module is called by `app.js` (`<Ui_module_dir>/view/base/web/js/core/app.js`) and receives the component's configuration as a parameter.

In the same way, anyone who wants to create a component dynamically (from other components or the `<script/>` tag) can call `app.js` or `layout.js` and pass as a parameter the configuration of the desired component.

## Implementation details

The `uiLayout` module is a singleton. It returns `function run(nodes, parent, toCache, toMerge)`.

Where arguments are the following:

* `{array} nodes`: array of configurations of the UI components that we want to initialize 
* `{string} parent`: parent component for that UI components
* `{boolean} toCache`: defines whether argument `nodes` will be cached. The key for cache will be constructed from `nodes` object keys
* `{boolean} toMerge`: defines whether the  argument `nodes` should be merged with the same one, that has been cached

Arguments `toCache` and `toMerge` play together when a particular UI component instance need to be updated in runtime.

Each item in `nodes` array is an object with the following properties:

* `{string} name`:the component name (short name). Its value is set to the `index` property of the UI component instance

  Note: The full name of the created UI component is formed by concatenating the `parent.name + '.' + name`, and then set as the `name` property. If an instance with the same full name already exists, `uiLayout` will skip its initialization.

* `{string} parent`: the `name` of the component's parent element (full name). If the parent component is not yet initialized, then `uiLayout` waits for it to appear in the `uiRegistry`. 
* `{string} template`: a path to `.html` template
* `{string} component`: path to the JavaScript class (function-constructor) of the new component
* `{object} config`: the object that contains the properties that you want to see in new UI component. In fact, they can also simply be added directly into item. That means, the following configurations will have the same result:

    var config1 = {name: 'myComp1', config: {myProp: '123123'}}

and

    var config2 = {name: 'myComp1', myProp: '123123'}

* `{array} children` - configuration of child elements, if there are any
* `{boolean} isTemplate` - if the value is set to `true`, the component configuration will be stored by `uiLayout` in the private `templates` variable. Current UI component will be not initialized. New instances can be created dynamically based on this object
* `{string} nodeTemplate` - a full name of component from `templates` private object. Stored component will be used as template object for current one
* `{string} provider`- the full name of the DataSource UI component. If property is absent, then parent's `provider` will be inherited

## Example 1:

Let's consider a case when we want to create a UI component instance dynamically from another UI component.
    
We can put the configuration of the desired UI component directly into `uiLayout`. Below in an example of the component's constructor,  which creates the child UI component on initialization:

Before example can be used special files should be created:

* `app/code/OrangeCo/Sample/view/<area>/web/js/my-new-component.js` - a JavaScript component
* `app/code/OrangeCo/Sample/view/<area>/web/templtes/my-new-component/main-template.html` - a component `.html` template, that will be used on client side
* `app/code/OrangeCo/Sample/view/<area>/web/js/sample.js`

The source of `app/code/OrangeCo/Sample/view/<area>/web/js/sample.js`:

{%highlight js%}
define([
    'uiLayout',
    'uiCollection'
], function (layout, Collection) {
    'use strict';

    return Collection.extend({
        defaults: {
            myNewComponentConfig: {
                name: 'myNewComponent',
                component: 'OrangeCo_Sample/js/my-new-component',
                nodeTemplate: 'OrangeCo_Sample/my-new-component/main-template',
                parent: '${ $.name }'
            }
        },
        initialize: function () {
            this._super();
            layout([this.my_newComponentConfig]); // this call will create a child component myNewComponent

            return this;
        }
    });
});

{%endhighlight%}

## Example 2:

Anther option is to create  UI components from the "template" component. For example, lets call it `myRowTemplate`, has already been configured with `isTemplate: true` property.

Lets have such configuration:

{%highlight xml%}
    <container name="my_table">
        <item name="component">OrangeCo_Sample/js/table</item>
    </container>
{%endhighlight%}

And created files:
* `app/code/OrangeCo/Sample/view/<area>/web/js/table.js`

Lets suppose that `myRowTemplate`'s full name is `myRowTemplateComponentName`.

Then, inside of "my_table" UI component we can add rows using this template configuration. The source `OrangeCo_Sample/js/table.js`:

{%highlight js%}
define([
    'uiLayout',
    'uiCollection',
    'mageUtils'
], function (layout, Collection, utils) {
    'use strict';

    return Collection.extend({
        defaults: {
            myRowTemplateConfig: {
                parent: '${ $.name }',
                nodeTemplate: 'myRowTemplateComponentName'
            }
        },

        addRow: function (rowIndex) {
            var objectFromTemplate,
                myRowTemplateConfig = this.myRowTemplateConfig;

            myRowTemplateConfig.name = rowIndex;
            objectFromTemplate = utils.template(myRowTemplateConfig);
            layout([objectFromTemplate]);
        }
    });
});
{%endhighlight%}

Now `addRow` method could be called from a `.html` template. 