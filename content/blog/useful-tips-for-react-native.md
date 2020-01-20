---
path: useful-tips-for-react-native
date: 2016-04-29T18:59:58.236Z
title: "Useful Tips for React Native"
description: "We recently decided to revamp our app using React Native. Other than reading a few articles I was very unfamiliar with React/React Native. From my initial research I knew we wanted to use something…"
---

We recently decided to revamp our app using React Native. Other than reading a few articles I was very unfamiliar with React/React Native. From my initial research I knew we wanted to use something like Alt.js or Redux to manage state. We ended up going with Redux and heavily watched [Dan’s talk on egghead](https://egghead.io/series/getting-started-with-redux). This gave us a glimpse into how to get started, but Todo apps are only useful to base your app on when they have simple data structures. We then decided to look at Facebook’s F8 app, which was [made open-source](http://makeitopen.com/). This gave us a better understanding about more complex data. I think that everyone just starting out with React, React Native, or Redux should look at all of these examples as they come from the people with the best knowledge of the tools. These are useful but I want to show you a few other items that we find helpful or useful to know.

## Remote Debugging

Running your app on a simulator is pretty simple and works pretty well out of the box. If however you want to run your code on a device it requires setting up a tunnel to your computer or connecting directly to your computer using its IP address. At CompanyCam we use [forward](http://forwardhq.com) for a lot of things and this is one of them. We set the location of our JavaScript based on if you we are in development (DEBUG) or production. In our app delegate we choose where to load the code from checking if there is a DEBUG symbol:

```objective-c
_AppDelegate.m_

NSURL *jsLocation;
#if DEBUG
  NSString *url = [[[NSBundle mainBundle] infoDictionary] stringForKey:@”REACT_SERVER_URL”];
  jsLocation = [NSURL URLWithString:url];
#else
  jsLocation = [[NSBundle mainBundle] URLForResource:@”main” withExtension:@”jsbundle”];
#endif
```

The value that we set for the _REACT_SERVER_URL_ value in _Info.plist_ is our url from Forward but with “_react-_” and the user’s name embedded in it:

```
https://react-${USER}-companycam.fwd.wf:8081/
```

The `USER` flag is set by Xcode and matches the output of `whoami`, which is coincidentally what we use in your Procfile. “What is a Procfile used for you?” may be asking. In this case the Procfile is used by [foreman](https://github.com/ddollar/foreman), a tool I use for nearly every project. Foreman allows you to have a set of commands in a given Procfile and it runs each of them. To start running our app we have a single command:

```bash
foreman start
```

This runs every command in our Procfile which is only two in our case:

```
Procfile
```

```bash
react: (JS_DIR=`pwd`/app; cd node_modules/react-native; npm run start — — root $JS_DIR)
forward: forward 8081 react-`whoami`
```

Now launch on your phone and simulator and see them work together.

## Redux

When we started using Redux I was like “ok, this looks cool. How do I use it in a big app though?”. Luckily Facebook kicks ass and open-sourced their [F8 app](http://makeitopen.com/) this year. The also did a nice [write-up](http://makeitopen.com/tutorials/building-the-f8-app/data/) on integrating data with your app. This was useful, but at the end of the day playing around with the tools is what made me comfortable with it. At first I wanted to set an array as the value of a root key in the global app state when using `combineReducers`. After a bit of messing around I realized this wasn’t possible. Once you come to grips with the fact that if you create a reducer for say users, companies, and groups and join them using [`combineReducers`](http://redux.js.org/docs/api/combineReducers.html)_l_, you are going to have a global state with the keys “users”, “companies”, and “groups”. Anything returned from the respective reducer will be a key under that root key. In our case we store a key “all” and a key “filtered” for each leaving our global state looking something like:

```javascript
{
  "users": {
    "all": [_user1, user2, etc..._],
    "filtered": [_id1, id2, etc..._]
  },
  "groups": {
    _...same
_  },
  "companies": {
    _...same
_  }
}
```

Then in components that care about a specific part of the state you can use the _connect_ function provided by [react-redux](https://github.com/reactjs/react-redux) and provide it a function that is called each time the state is updated so it can update the components props.

```javascript
// GroupDetails.js

class GroupDetails extends React.Component {
  // ...code omitted
}

// state is the apps global state object when using Redux
const mapStateToProps = (state, ownProps) => {
  return {
    users: state.users.all,
    group: state.groups.all.filter(group => {
      return group.id === ownProps.groupId
    }),
  }
}

export default connect(mapStateToProps)(GroupDetails)
```

Now whenever the state gets updated this component will be notified and will be able to update it’s _users_ and _group props._

This is just the tip of the interesting things that we have found so far. I would like to hear things that you have learned that makes React/React Native development easier once it clicks. If you have some, leave them in the comments.
