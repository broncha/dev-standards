# [Standard] React Native performance

## Owner: [Louis Lagrange](https://github.com/Minishlink)

## Checks

In order to have an efficient application from the get-go, respect the following standards.

## Why

Because a few seconds of page render, a laggy chart display or a buggy animation makes you lose customers, those tips will help you avoid that before it even happens.
In order to have an efficient application from the get-go, respect the following standards.

### General

> Side note: You don't need to apply every of these standards right away (that would be premature optimization), but as your technical experience grows, you should adopt them along the way. These best practices are ordered by potential impact on performance.

- Use an up-to-date version of your dependencies, and first and foremost:
  - [React Native](https://github.com/facebook/react-native/releases)
  - [React Navigation](https://github.com/react-navigation/react-navigation/releases)
- Don't use images that are unnecessary big. Dynamic resizing is very inefficient on Android. Resize them to 1x 2x 3x flavors (`img.png`, `img@2x.png`, `img@3x.png`) and use them normally (`require('img.png')`). UX designers can export images easily this way with Sketch.
- Use animations in order to make things more fluid (`animationEnabled` in `TabNavigator`; `LayoutAnimation`)
- Use `shouldComponentUpdate` / `PureComponent`. Test thoroughly your component when using `shouldComponentUpdate` because this is error-prone. It will massively improve your app's performance though.
- Don't create new functions on every render, [bind your functions efficiently](https://github.com/bamlab/dev-standards/blob/master/react-native/react/binding-functions-in-react-component.s.md). Similarly, avoid creating inline styles.
- When using `Animated`, use [`useNativeDriver`](https://facebook.github.io/react-native/docs/animations.html#using-the-native-driver)
- If you have a big view that has a lot of subviews, and these are not always shown to the user, use [`removeClippedSubviews`](https://facebook.github.io/react-native/docs/view.html#removeclippedsubviews)
- When triggering a function after clicking on a button, or at `componentDidMount`, use [`InteractionManager.runAfterInteractions`](https://facebook.github.io/react-native/docs/interactionmanager.html)
- Remove console logs from your production builds, use the [`transform-remove-console`](https://facebook.github.io/react-native/docs/performance.html#using-consolelog-statements) Babel plugin
- When possible, use `Fragment` instead of `View`
- Try to limit the number of data you display in charts, maps and tables. To investigate the potential impact, try to divide this number by 10 and measure the impact with the tools presented in the profiling section
- Do not request your data to often: if it changes every hour, do not perform the same request every minutes, it will trigger useless renders and waste ressources.

### Useless renders

- Use `shouldComponentUpdate` / `PureComponent`. Test thoroughly your component when using `shouldComponentUpdate` because this is error-prone. It will massively improve your app's performance though.
- Don't create new functions on every render, [bind your functions efficiently](https://github.com/bamlab/dev-standards/blob/master/react-native/react/binding-functions-in-react-component.s.md).
- Do not request your data to often: if it changes every hour, do not perform the same request every minutes, it will trigger useless renders and waste ressources.

### Displaying large list of elements

- Try to limit the number of data you display in charts, maps and tables. To investigate the potential impact, try to divide this number by 10 and measure the impact with the tools presented in the profiling section
- If you have a big view that has a lot of subviews, and these are not always shown to the user, use [`removeClippedSubviews`](https://facebook.github.io/react-native/docs/view.html#removeclippedsubviews)

### Technical debt

- Use an up-to-date version of your dependencies, and first and foremost:
  - [React Native](https://github.com/facebook/react-native/releases)
  - [React Navigation](https://github.com/react-navigation/react-navigation/releases)
- Remove console logs from your production builds, use the [`transform-remove-console`](https://facebook.github.io/react-native/docs/performance.html#using-consolelog-statements) Babel plugin
- When possible, use `Fragment` instead of `View`

### Start-up times

If your app takes too much time to initialize, solve the problem incrementally:

1.  Add a splashscreen that [closes when the app is ready](https://github.com/Minishlink/DailyScrum/commit/811cfd57304dbb6f08386bce7b1d9d0b7c7388ae) with [`react-native-splash-screen`](https://github.com/crazycodeboy/react-native-splash-screen)
2.  If the startup time is > 2 seconds, show a full page modal with an animation (in the continuity of your splashscreen)
3.  If the startup time is consistently > 5 seconds (or 7 seconds with an animated splashscreen): if you have a very big app, implement [dynamic imports](https://facebook.github.io/react-native/docs/performance.html#unbundling-inline-requires); if not, look for other clues: aren't you doing some long API calls at startup?

### Maps

A lot of apps seem to be struggling with performance when it comes to maps & displaying pins.
Here are a few tips I suggest using to improve your maps' performance:

1. Use `onRegionChangeComplete` instead of onRegionChange. You will wait until you're done changing the region.
2. Use `debounce` when making HTTP calls to fetch new pins.

> ⚠️ If a call takes longer than your wait time. Another call might triggered in the mean time & go through the callback before the first call. It might cause a laggy feeling & incoherent data. Cancelling debounce will not cancel async callback. You can do something like below :

```javascript
     //Generate a unique id to identify a query
     const queryId = shortid.generate();
     this.queryId = queryId;
     this.setState({ isLoading: true }, () =>
       this.props
         .callForPins(..args)
         .then(({ data: { pins } }) => {
           //There is a newest callback no need to update the state
           if (this.queryId !== queryId) return;
           return this.setState({
             pins,
           });
         })
         .catch(console.warn)
         .finally(() => this.setState({ isLoading: false }))
     );
```

3. Use a `caching technique` to reduce the number of queries

   When calling for new pins, you need to consider what your inputs are. Given that you query pins on the map with the following arguments:
   * Region on which the user is (eg. You're located over Paris. You don't want to load a pin in New York)
   * Types of pins (eg. On a map, you just want the restaurants but you don't want to see the shops)
   * Search field (eg. If you're searching for bistrots you don't want to get Mc Donalds' pins)

   You can create a hash key with the type of pins & the search field :

```javascript
        //callForPins callback
        const key = extractKey({ pinTypes, search });
        this.setState({
            queriedRegion: {
              [key]: boundaries(region),
            },
            pins,
        });
```

   Before querying for new pins on the map check whether the last query's hash key is the same & whether the region of your query is within the previous query's region. If so we already have the pins & there is no need for querying new pins.

```javascript
      _getPins = (region, pinTypes, search) => {
        //Create a uniq key out of the pinTypes & search parameters
        const key = extractKey({ pinTypes, search });
        if (this.state.queriedRegion[key]
            && isRegionWithIn(boundaries(region), this.state.queriedRegion[key])) return;
        //Call for new pins with debounce if the pinTypes or search parameters have changed since last time
        //Or if the region queried is larger than the previous one
        return this._callForPinsDebounce(region, pinTypes, search);
      };
```

> As regards to caching, there still things that need to be done. At the time, I was doing that Apollo did not provide any caching per parameter. It would allow us to have a greater history of calls in cache & only the last one. Also, ideally you could think of caching when zooming out. This involves some dev on the back-end, as it means that you are capable of fetching pins in a ring & only inside a cercle.

4. `componentShouldUpdate` is your friend. Don't re-render the map if your pins haven't updated for instance.
5. Clustering of pins on the front end ([Clustering with react-native-maps](https://github.com/bamlab/react-native-components-collection/tree/master/packages/react-native-component-map-clustering))

## Good examples

> Please andon and/or create an issue if you need one!

## Bad examples

> Please andon and/or create an issue if you need one!
