---
path: turbolinks-stimulus-and-react-oh-my
date: 2018-04-12T20:10:09.873Z
title: "Turbolinks, Stimulus, and React Oh My!"
description: "CompanyCam’s web app has changed considerably over the years. Starting as a .NET app packed with tons of jQuery plugins 🤮 before moving onto a Backbone Marionette app 🤢. About a year ago I rewrote…"
---

[CompanyCam’s](https://companycam.com) web app has changed considerably over the years. Starting as a .NET app packed with tons of jQuery plugins 🤮 before moving onto a Backbone Marionette app 🤢. About a year ago I rewrote the entire web app as a single-page React application that consumed our REST API. This works _well_, but sometimes React is overkill for simple CRUD actions.

The team at CompanyCam is small, we have to optimize where we can to stay small and still achieve our goals. This is what spurred me thinking about how to optimize our productivity and maintain our happiness 😁.

After a little investigation, I began tinkering with moving the mundane items to regular ole’ Rails ERB templates and re-enabling [Turbolinks](https://github.com/turbolinks/turbolinks). This was simple enough to start with but I found myself needing to add some functionality such as multi-select drop-downs with autocompletion. Using JavaScript plugins with Turbolinks is a bit more difficult than binding to the page load event and initializing the plugin. Turbolinks loads subsequent pages using AJAX and replaces the page body with the response. Wouldn’t it be nice if you could just know when an element was added to the page and initialize any plugins that the element(s) needed? _Queue the mystical harp music 🎶_.

This is when I reached for [Stimulus](https://stimulusjs.org/), a new framework from Basecamp that uses the MutationObserver API to listen to changes in the DOM and connecting it with a JavaScript object. When a new element is inserted, such as a new body from Turbolinks or a div from server-generated JavaScript, Stimulus looks for the elements that have a \`data-controller\` attribute and links the element with the corresponding controller. This may not seem powerful at first, but it allowed me to write a small snippet of code once and use it for all of the multi-selects in the future. This is that code:

This controller takes an existing select tag and converts it into a select2 instance, affording you all of the handy features of select2. The problem is that Turbolinks will cache the current page before navigating to a new page, which we don’t want. That is why we have the event listener for `turbolinks:before-cache` that gets the values from select2 and marks the corresponding option(s) as selected in HTML before unmounting the select2 plugin. The next time you navigate to the page, this controller’s function is called again and select2 is remounted.

We also have one for hiding the flash messages from Rails after a short delay. The controller below is very simplistic, but you get the idea.

As you can see, Stimulus is pretty useful and compliments Turbolinks very well.

Everything so far is fine and dandy, but what about the complex or extremely dynamic features we already have built in React? Well, that was the last piece I needed to figure out. I ended up pulling a bit of inspiration from react-rails and created a helper to render an element on the page containing the name of the React component and any props JSON stringified.

You’ll notice that it is creating an empty`div`, but it also adds a few data tags to hook it up with Stimulus. Stimulus can then take over and mount the React component when the element is detected on the page. The Stimulus controller is as simple as:

When the element is rendered on the page Stimulus will lookup the React component out of a global object, get any props from the element, call `ReactDOM.render`, and then add an event listener for `turbolinks:before-cache` to unmount the component before changing pages. The beauty once again is that you can have as many of these elements on a page as you want and each will be rendered and unmounted separately.

This hasn’t been tested in production yet, but I plan to try it out on beta once I get it a bit further along. Local testing, however, shows positive results and has been quite easy to work with as I migrate the React views back to Rails templates.

If you have any questions or have built something similar to this, let me know in the comments below.
