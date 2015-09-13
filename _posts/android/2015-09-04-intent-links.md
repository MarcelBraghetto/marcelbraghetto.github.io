---
layout: page
title: "Launching apps from HTML links"
category: Android
---

## Launching apps from HTML links

[Get the source for this example here](https://github.com/MarcelBraghetto/BlogDemos/tree/master/DeepLinks)

A number of blog posts in this site include sample code that I am writing into an Android app as kind of a kitchen sink sandbox app to demonstrate how the code runs.

I hadn't tried using deep links before and I thought it would be really cool if I could have HTML links in my blog content that could open my Android apps directly to code samples related to my blog posts.

<!-- excerpt -->

A bit of Google foo and some reading up on the [Android docs](https://developer.android.com/training/app-indexing/deep-linking.html) and especially the [Code Labs tutorial](http://search-codelabs.appspot.com/codelabs/android-deep-linking) showed that it was perhaps not as difficult as I imagined.

Simply put, all that is needed is:

- Some intent filters for an activity that we want to link through to.
- The ability for that activity to parse any given data into something meaningful and take an action.
- A link to embed in a web page to initiate the intent.

Let's make a basic example app.

### 1. Create an app and deep link activity

Create a new Android app, with the default *MainActivity* that gets generated for you. Then add a new activity that will be our deep linking entry point. I've called mine *DeepLinkActivity*.

In my example, when the *DeepLinkActivity* is launched via an external intent, it simply restarts the *MainActivity* and passes along the *action* and *data* from the originating intent, along with some intent flags to basically restart the activity stack.

This is the point where you could really do anything you like though based on whatever data the intent was initiated with.

```java
public class DeepLinkActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // Generate a new intent to launch our main activity
        // with the given action and data, and tell it to
        // clear and restart a fresh instance.
        Intent intent = new Intent(this, MainActivity.class);
        intent.setAction(getIntent().getAction());
        intent.setData(getIntent().getData());
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_CLEAR_TASK);
        startActivity(intent);

        // Finish this activity as well.
        finish();
    }
}
```

### 2. Update the manifest for the deep link activity.

Next we need to add the new *DeepLinkActivity* to the manifest, along with some filters to allow it to be resolved when we want to trigger it.

Add the following inside the *application* tag of your manifest.

```xml	
<activity
    android:name=".DeepLinkActivity"
    android:exported="true">

    <intent-filter>
        <data
            android:scheme="deeplink"
            android:host="io.github.marcelbraghetto.deeplinking"
            />

        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
    </intent-filter>
</activity>
```

Note that:

- We are registering the scheme *deeplink*.
- The *host* is defined so we can qualify our links.

### 3. Tweak the MainActivity to do something

In this example, we will simply take the default *MainActivity* and add a bit of extra code to examine the contents of the intent that was used to launch it.

There will be two launch conditions for the *MainActivity* class:

1. Normal launch by the user tapping the app icon.
2. Intent based launch from the *DeepLinkActivity* with some intent data.

We are just going to display the text of the intent data, or a default message if the *MainActivity* was launched by the user:

```java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        TextView textView = (TextView) findViewById(R.id.main_text);
        String action = getIntent().getAction();
        Uri data = getIntent().getData();

        // If we were started by an ACTION_VIEW, then
        // grab the content of the passed in data and
        // display it to the user.
        if(Intent.ACTION_VIEW.equals(action) && data != null) {
            textView.setText(data.getLastPathSegment());
        } else {
            // Otherwise it was a regular launch, just
            // show a default message.
            textView.setText(getString(R.string.default_launch_message));
        }
    }
}
```

That's pretty much it for the app.

### 4. Launching the app intent

The next step is launching the app from somewhere else. There is some good information [here](https://developer.chrome.com/multidevice/android/intents) about launching from a web link.

The idea is to attempt to initiate the intent, and launch the app that can handle it (based on the intent filters). The addition of a *browser\_fallback\_url* property will redirect the user if they don't have an app installed that can handle the intent.

Here is the link that should launch the example app if it is installed, or navigate to Google:

```html
<a href="intent://io.github.marcelbraghetto.deeplinking/Hi%20from%20Marcel%27s%20blog!#Intent;scheme=deeplink;package=io.github.marcelbraghetto.deeplinking;S.browser_fallback_url=https%3A%2F%2Fgoogle.com;end">Tap me to launch directly into the Android app</a>
```

This example link will attempt to resolve to an activity that has:

- The **host** property of *io.github.marcelbraghetto.deeplinking*.
- The **scheme** of *deeplink*.
- The target Android **package** of *io.github.marcelbraghetto.deeplinking*.

Check our sample manifest to see how these properties align to allow the app to be the resolved intent handler.

If you have the example app installed you can [tap here to trigger a deep link](intent://io.github.marcelbraghetto.deeplinking/Hi%20from%20Marcel%27s%20blog!#Intent;scheme=deeplink;package=io.github.marcelbraghetto.deeplinking;S.browser_fallback_url=https%3A%2F%2Fio.github.marcelbraghetto;end) - only works if you are browsing on an Android device!

### Wrap up

This was just a super simple example, but the concept is quite handy for surfacing app content via the web. You can also use *http* schemes and setup your intent filters to recognise particular url patterns and therefore appear in the list of suggested apps that can handle those particular urls (alongside your browser).