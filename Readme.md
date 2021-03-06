# Understanding pieces of Inheritance in Lightning Components #
=
The inheritance of components and the usage of interfaces is usually not explained at a greater detail 
when it comes to the majority of the samples we find as the google results are focussed on giving a basic idea.
may be I am searching wrong. So am summarizing it here, please feel free to comment or clarify and I will update this doc accordingly.

References :
* [Reusable lightning components by AutomaTom](https://www.slideshare.net/thomaswaud/advanced-designs-for-reusable-lightning-components)
* [Github repo for Scheduler](https://github.com/AutomaTom/scheduler)
* [Reusable lightning components by John Belo](https://developer.salesforce.com/events/webinars/AdvLightning?d=7010M000001yCik)
* [Interface reference](https://developer.salesforce.com/docs/atlas.en-us.lightning.meta/lightning/ref_interfaces.htm)
* [Aura Components not necessarily in lightning components](https://github.com/forcedotcom/aura/tree/master/aura-components/src/main/components/ui)

understand usage of {!v.body}
=

cmp1.cmp
```html
<aura:component>    
    {!v.body} 
</aura:component>
```

cmp2.cmp
```html
<aura:component>
    This is the dummy text inside component2
</aura:component>
```

TestApp.app
```html
<aura:application>
    <c:cmp1>
        <c:cmp2 />
    </c:cmp1>    
</aura:application>
```

> To see the definition of cmp2 , the cmp1 definiton should have a line at the end 
{!v.body}

body attribute refers to the content which is wrapped inside a component.
ex: 
```html
<cmp1>
    Everything that is here is part of body attribute.
</cmp1>
```

so in my component definition if i dont have the line 
```html
<aura:component>
    <!-- {!v.body} -->
</aura:component>
```
Anything wrapped inside will not be displayed.

understanding aura:method
=

```html
<aura:component>
    <aura:method action="{!c.controllerMethod}" name="doControllerMethod">
        <aura:attribute name="name" type="String" />
        <aura:attribute name="arguments" type="Object[]" />
    </aura:method>
</aura:component>
```

This is a publicly accessible method as part of the component.
at any point am able to get the component definition , I will be able to access the controller method directly.


extensible="true" abstract="true"
=

I am gonna have 3 components
* baseSearch
    * has searchstring and searchResults
    * orchestrates how the helper methods are called.    
* advSearch 
    * has one text box for searching
    * implements the missing method from baseSearch
* moreAdvSearch
    * has one button to submit for searching
    * tries to call the search method which is shared with helpers.

baseSearch.cmp
```html
<aura:component extensible="true" abstract="true">
    <aura:attribute type="String" name="searchString" access="public" />
    <aura:attribute type="Object[]" name="searchResults" access="public" />
    <aura:method name="doSearch" action="{!c.doSearch}" access="public">
        <aura:attribute name="name" type="String" />
        <aura:attribute name="arguments" type="Object[]" />
    </aura:method>
    {!v.body}
</aura:component>
```

>This is the place we should expand our understanding of the "helper" inside lightning components.

> [reference for javascript "apply"](https://msdn.microsoft.com/en-us/library/4zc42wh1(v=vs.94).aspx) 

baseSearchController.js
```javascript
({
    doSearch : function(component, event, helper) {
		var params = event.getParam("arguments");        
        /*
        we need a detailed understanding of apply function of javascript
        it was confusing for me until i saw this example.        
        */

        helper[params.name].apply(helper, params.arguments);
	}
})
```

baseSearchHelper.js
```javascript
({
    test:function(){
        //since the component is abstract, 
        //I can define my helper methods in one of the extended ones.
        //but its important that helper object is not empty , it has atleast 1 function at the base component.
    }
})
```

advSearch.cmp
```html
<aura:component extends="c:baseSearch" extensible="true">   
    <ui:inputText class="slds-input" value="{!v.searchString}" />
    {!v.body}
</aura:component>
```
advSearchController.js
```javascript
({
    //its empty
})
```

advSearchHelper.js
```javascript
({
    //the name baseHelperMethod is important for us to know which function to call from one of the children.
    // since the baseSearch is a abstract component this is where we implement the definition.
	baseHelperMethod : function(component) {
		//instead of getting results ,am populating some dummy after 5 seconds.
        window.setTimeout(
            $A.getCallback(function(){
                if(component.isValid()){
                   component.set('v.searchResults',[{'a':'b','c':'d'},{'a':'b5','c':'d5'},{'a':'b4','c':'d3'},{'a':'b2','c':'d2'},{'a':'b1','c':'d1'}]) ;
                }
            }),5000
        ); 
	}
})
```

moreAdvSearch.cmp
```html
<aura:component extends="c:advSearch">
	<ui:button label="Search Advanced" press="{! c.doSearchTest }" />
    <aura:iteration items="{!v.searchResults}" var="searchResult">
        {!searchResult.a} - {!searchResult.c} <br />
    </aura:iteration>    
</aura:component>
```

moreAdvSearchController.js
```javascript
({
    doSearchTest : function(component, event, helper) {
        // previously we had access to the helper functions of the parent components directly but 
        // after locker service is enabled we have to go through the workaround of
        // calling a public method which calls the helper method from base component.
        component.getSuper().doSearch('baseHelperMethod',[component.getSuper()]);
	}
})
```

Interfaces
=
Interface at a high level helps us define a specific group of functionality that every component can chose to implement in its own way.

ex: a quick action, if a component is implementing a force:lightningQuickAction or force:lightningQuickActionWithoutHeader , this enables us to configure custom quick actions. 

```javascript
$A.get("e.force:closeQuickAction");
$A.get("e.force:closeQuickAction");
```

are available directly on the component controller/helper without registering them separately.

These are advantages of using platform specific quick actions , but we also create our own interfaces.
now hold on , you may ask why do we need to do that ?
>They dont have any additional functionality other than attributes and registered events , we can as well have that code in our component.One of the biggest temptations of procedural programming.

Lets dive into an example.
