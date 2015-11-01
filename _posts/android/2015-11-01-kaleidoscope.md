---
layout: page
title: "AIDL - Kaleidescope"
category: Android
---

## AIDL - Kaleidoscope

![alt AIDL Kaleidoscope](/images/aidl/aidl.png)

[Get the source for this example here](https://github.com/MarcelBraghetto/BlogDemos/tree/master/Kaleidoscope)

I wanted to play around with [AIDL - Android Interface Definition Language](http://developer.android.com/guide/components/aidl.html) to test out how it can be used to communicate between two (or more I guess) Android processes at run time.

I found that there is more than one way to do **Inter Process Communication (IPC)** on Android, but I wanted to focus on the scenario of two completely different APKs talking to each other, via a contract defined using **AIDL**.

<!-- excerpt -->

The use of AIDL to define a **contract** for communication within Android processes requires us to use *bound services* through a connection's binder. You can do *IPC* without directly using *AIDL* at all, but that isn't the focus of this demo. You could also use *IPC* mechanisms to communicate between different processes within the **same** Android app.

Key goals for this demo:

- Create a **local** application and a **remote** application which have *AIDL contract* interfaces that can be used to send commands between them.
- Our **local** application will allow us to draw on the screen, and will use the *AIDL contract* to forward the drawing commands to the **remote** application process.
- The **remote** application process will perform its own transformations on the actions it receives and call back via the *AIDL contract* to the **local** process with drawing commands of its own!

The idea is that the **remote** process will take the drawing commands it is given and do something to make some kind of *kaleidoscope* effect (*Warning:* the effect is really lame in this demo!).

The overall loop of communication will be:

- **Local** sends drawing commands via **AIDL** to **Remote**.
- **Remote** translates drawing commands.
- **Remote** sends drawing commands via **AIDL** back to **Local**.

Here is a quick diagram of what we will construct:

![alt AIDL diagram](/images/aidl/aidl_diagram.png)

Onward we go!

## 1. Building the local application

The **local** application is pretty basic, really there are just a couple of new things to do after generating a default Android Studio project:

**1a. Create a new AIDL interface contract**

After creating a new project, (I named this project **Local** - you can refactor the default **app** module name), add a new **AIDL** file and name it **IKaleidoscopeInterface** as so:

![alt Add AIDL file](/images/aidl/aidl_step1a.png)

![alt Add AIDL file](/images/aidl/aidl_step1b.png)

![alt Add AIDL file](/images/aidl/aidl_step1c.png)

Our AIDL interface is the key to communicating between processes. Only the methods in this interface may be invoked and the set of data types you can use is limited. See the [Android Developer](http://developer.android.com/guide/components/aidl.html#Create) resources for more info.

For our interface we want to do a one thing: *Draw a line between two points.*

Edit the AIDL interface to reflect this:

```java
interface IKaleidoscopeInterface {
    /**
    * Request that a line be drawn between the
    * two given coordinates.
    */
    void drawLine(int x1, int y1, int x2, int y2);
}
```

**1b. Create a service that can be invoked by the remote process**

In order for the **remote** process to send commands back to our **local** application, we must provide a service that meets the requirements of the AIDL contract we wrote.

When something tries to *bind* to the service, it will be returned an implementation of our AIDL contract as the *binder*.

**Cool tip:** If your intellisense can't find the *IKaleidoscopeInterface* class, you most likely need to perform a *Gradle Sync* so the AIDL file code generation is triggered.

**LocalKaleidoscopeService.java**

```java
package io.github.marcelbraghetto.kaleidoscope.local;

public class LocalKaleidoscopeService extends Service {
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mKaleidoscopeInterface.asBinder();
    }

    /**
     * This is the secret sauce, our service will create an implementation of
     * the AIDL contract 'stub' which can then be returned as the 'binder'
     * when the 'onBind' method is invoked on the service.
     *
     * When the consumer of the service is connected and invokes methods via
     * AIDL, this is where they will be executed.
     */
    private final IKaleidoscopeInterface mKaleidoscopeInterface = new IKaleidoscopeInterface.Stub() {
        @Override
        public void drawLine(int x1, int y1, int x2, int y2) throws RemoteException {
            // TODO: Do something with the received action!
        }
    };
}
```

The key part of the code above is that when something tries to *bind* to our local service, the *onBind* method will be invoked and we will return an *implementation of the AIDL contract*. This allows a remote process to consume our service and talk via AIDL to us!

You may notice that we are creating an implementation of **IKaleidoscopeInterface.Stub()** - Android generates the **Stub()** method as part of AIDL.

**1c. Expose the new local service**

Our new *LocalKaleidoscopeService* can't be accessed as a service just yet because we haven't put its declaration into the app manifest.

The **local** application needs to expose a service that meets the AIDL contract, so the **remote** process can *bind* to it. There is something to consider when doing this:

- A *service* when marked as *exported = true* or with an accessible *intent-filter* will have no permissions on it. This means anyone could really jack into your service which might be a security issue.

For this demo, we'll add a basic custom permission that the service will require in order for something to bind to it.

**AndroidManifest.xml**

```xml
<permission
    android:name="io.github.marcelbraghetto.kaleidoscope.local.LocalKaleidoscopeServicePermission"
    android:protectionLevel="signature"
    />

<application>
    <!-- Insert your other manifest config -->
    <service
        android:name=".LocalKaleidoscopeService"
        android:exported="true"
        android:permission="io.github.marcelbraghetto.kaleidoscope.local.LocalKaleidoscopeServicePermission">
        <intent-filter>
            <action android:name="LocalKaleidoscopeService" />
        </intent-filter>
    </service>
</application>
```

Note that we have added a custom permission named **LocalKaleidoscopeServicePermission** and that our service definition requires it. You may want to extend the permission implementation but I'll keep it simple for this blog post.

The service definition will use the *intent-filter* with an *action* to allow itself to be accessed remotely. When we construct the **remote** application, it will use this service.

##2. Building the remote application

Next we'll create the remote application, which will contain the *remote service* which our *local* application will bind and send AIDL commands to.

**2a. Create remote module**

Start by creating a new application module in Android Studio named **Remote**.

![alt Add AIDL file](/images/aidl/aidl_step2a.png)

**2b. Copy AIDL definition**

We will be writing two way communication via the same AIDL interface contract, so the new *remote* app needs a copy of the **exact same** AIDL definition file that we used in the *local* app.

A simple way to do this is to go into *Finder* (or whatever file manager you use) and copy the *src/main/aidl* folder from the *local* app into the *src/main* folder in the *remote* app.

After doing this, trigger a *Gradle Sync* and you should see something similar to this:

![alt Add AIDL file](/images/aidl/aidl_step2b.png)

**2c. Create remote service**

The *remote* application will implement a service that implements our AIDL interface contract just like our *local* application did. The actual behaviour of the service implementation will be a bit different but the steps to follow are almost identical.

First, create the Java service in the *remote* application:

**RemoteKaleidoscopeService.java**

```java
package io.github.marcelbraghetto.kaleidoscope.remote;

public class RemoteKaleidoscopeService extends Service {
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mKaleidoscopeInterface.asBinder();
    }

    private IKaleidoscopeInterface mKaleidoscopeInterface = new IKaleidoscopeInterface.Stub() {
        @Override
        public void drawLine(int x1, int y1, int x2, int y2) throws RemoteException {
            showToast("Remote: Received drawLine command");
        }
    };

    private void showToast(final String message) {
        new Handler(Looper.getMainLooper()).post(new Runnable() {
            @Override
            public void run() {
                Toast.makeText(getApplicationContext(), message, Toast.LENGTH_SHORT).show();
            }
        });
    }
}
```

At the moment we will just display a *Toast* message when any of the AIDL interface methods are invoked. We'll revisit this later to do something more interesting but for now it will at least tell us that the AIDL communication is working.

**2d. Expose the new remote service**

Of course we have to expose the remote service in the manifest so it can be bound to by the *local* application:

```xml
<permission
    android:name="io.github.marcelbraghetto.kaleidoscope.remote.RemoteKaleidoscopeServicePermission"
    android:protectionLevel="signature"
    />

<application>
    <!-- Insert your other manifest config -->
    <service
        android:name=".RemoteKaleidoscopeService"
        android:exported="true"
        android:permission="io.github.marcelbraghetto.kaleidoscope.remote.RemoteKaleidoscopeServicePermission">
        <intent-filter>
            <action android:name="RemoteKaleidoscopeService" />
        </intent-filter>
    </service>
</application>
```

The configuration is *very* similar to what we did earlier for the *local* application.

Build and run the *remote* application so it is installed on your test device and is ready to have its service bound, ready for the next part.

##3. Bind to the remote service

Now that we have a *remote* service available, we can revisit the *local* application and **bind** to it. This will allow us to start sending AIDL commands to the *remote* service finally - woohoo!!

**3a. Register remote service permission**

Remember how we put in basic permissions for our exposed services (if not then you weren't paying attention!)? Well, in order to bind to a service that needs a permission, we have to add the permission as a **uses-permission** clause in our manifest which matches the **permission** defined in the *remote* app manifest.

Edit the manifest for the *local* application and add a new line, above the existing permission line:

```xml
<uses-permission android:name="io.github.marcelbraghetto.kaleidoscope.remote.RemoteKaleidoscopeServicePermission" />
```

**If you don't do this, you will get a security exception when you try to bind to the service!**

**3b. Bind a connection to the remote service**

Ok, finally we get to write some actual Java code to use the services we created. To bind a connection to our *remote* service we need to follow the following steps:

**3c. Create a field reference to our AIDL interface**

During the binding of the remote service, we will be assigning the *IBinder* parameter that is given to us to a local field of type *IKaleidoscopeInterface* which will then become our bridge to sending commands to the bound service.

We also need to create and hold an instance of a **ServiceConnection** which will be used during the binding (and unbinding) and who will initialize the reference to our AIDL interface instance. 

**MainActivity.java**

```java
public class MainActivity extends AppCompatActivity {
    private boolean mIsRemoteKaleidoscopeInterfaceBound;
    private IKaleidoscopeInterface mRemoteKaleidoscopeInterface;

    ...

    private final ServiceConnection mRemoteServiceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            // Note how we are using the AIDL Stub.asInterface to map the
            // provided IBinder to our AIDL interface.
            mRemoteKaleidoscopeInterface = IKaleidoscopeInterface.Stub.asInterface(service);
            mIsRemoteKaleidoscopeInterfaceBound = true;
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            mRemoteKaleidoscopeInterface = null;
            mIsRemoteKaleidoscopeInterfaceBound = false;
        }
    };    
}
```

**3d. Start the intent to bind the remote service**

Now that we have a few fields in place we can actually attempt to bind to the remote service. We'll do this when the activity is created.

This was actually quite tricky to get working correctly, the ability to a bind a service through an *implicit* intent was prohibited starting in Android Lollipop (5.0).

The problem is that we don't have any access to the actual Java class in the *remote* application so we can't create an explicit intent easily, for example:

```java
// This code will not compile from within the local application project
// because the RemoteKaleidoscopeService.java class is not in it's
// source path (nor should it be).
Intent intent = new Intent(this, RemoteKaleidoscopeService.class);
```

Hmm, what to do about that... well I discovered a way to still do it by getting Android to help us to create an explicit intent instead by using the *remote intent-filter action* and querying its properties.

```java
private Intent createRemoteServiceIntent() {
    // Create a basic intent using the intent filter action of
    // the remote service as per its manifest configuration.
    Intent intent = new Intent("RemoteKaleidoscopeService");

    // Attempt to resolve the service through the Android OS
    ResolveInfo info = getPackageManager().resolveService(intent, Context.BIND_AUTO_CREATE);

    // If the service failed to resolve it could mean that the
    // remote app is not installed or something wasn't configured
    // correctly, so we can't really start/bind the service...
    if(info == null) {
        return null;
    }

    // Otherwise, grab the resolved package name and service name and
    // assign them to the intent and bingo we have an explicit service intent!
    intent.setComponent(new ComponentName(info.serviceInfo.packageName, info.serviceInfo.name));
    return intent;
}
```

Sweet, so now we can bind the connection - and when the activity is destroyed we should unbind it:

**MainActivity.java**

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    ...
    Intent remoteServiceIntent = createRemoteServiceIntent();
    if(remoteServiceIntent != null) {
        bindService(remoteServiceIntent, mRemoteServiceConnection, Context.BIND_AUTO_CREATE);
    }
}

@Override
protected void onDestroy() {
    super.onDestroy();

    if(mIsKaleidoscopeInterfaceBound) {
        unbindService(mRemoteServiceConnection);
    }
}
```

**3d. Test the connection**

Ok, you've been hanging in there through this colossal wall of text, so let's actually see the connection working. To test it out, we'll write a method to send a *drawLine* command to the *remote* service.

For now, I'll just drop a button into the main activity and fire the command when it is clicked. I won't bore you with the button code - I'm sure you can do that, but here is the code to send the command to the *remote* service:

```java
private void sendDrawLineCommand() {
    // Don't bother sending the command if we are
    // not yet bound to the service.
    if(mIsRemoteKaleidoscopeInterfaceBound) {
        // Calling AIDL methods can cause a RemoteException
        // it's up to you how you deal with it...
        try {
            mRemoteKaleidoscopeInterface.drawLine(1, 1, 2, 2);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }
}
```

The following image is an example of what happens when this code runs. The interesting thing to note is that the *Toast* message being displayed is actually coming from the *remote* service! This means we finally have our service connection running via AIDL!!!

![alt AIDL connection](/images/aidl/connection_toast.jpg)

## 4. Back the other way

You might recall that at the start of this blog post, we created an AIDL service for the *local* app. We haven't yet used it but the goal was to have **two way AIDL communication** running.

So far we have **one way communication**, where the *local* application can send AIDL commands to the *remote* service. The next step is to have the *remote* service send commands **back** to the *local* application via the *local service* we wrote earlier.

**4a. Binding to the local service**

The *remote service* will need to bind itself to our *local service* in order to send commands back. The subtle difference here is that the *local* app is currently binding a service connection in an activity, whereas the *remote* app will need to bind it within its own service!

First off, we need to register the *permission* needed to use the *local AIDL service*. Edit the manifest in the *remote* app and add the following *uses-permission* clause:

```xml
<uses-permission android:name="io.github.marcelbraghetto.kaleidoscope.local.LocalKaleidoscopeServicePermission" />
```

Then edit the existing *RemoteKaleidoscopeService.java* class.

Similar to before, we need a few fields to hold our AIDL interface reference and *ServiceConnection* instance.

To make things clear, I've included the entire class below, as you can see, the service will now attempt to connect back to the *local* service and effectively route all AIDL commands to the *local service*.

**RemoteKaleidoscopeService.java**

```java
public class RemoteKaleidoscopeService extends Service {
    private boolean mIsLocalKaleidoscopeInterfaceBound;
    private IKaleidoscopeInterface mLocalKaleidoscopeInterface;

    @Override
    public void onCreate() {
        super.onCreate();

        Intent localServiceIntent = createLocalServiceIntent();
        if(localServiceIntent != null) {
            bindService(localServiceIntent, mLocalServiceConnection, Context.BIND_AUTO_CREATE);
        }
    }

    @Override
    public void onDestroy() {
        super.onDestroy();

        if(mIsLocalKaleidoscopeInterfaceBound) {
            unbindService(mLocalServiceConnection);
        }
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mKaleidoscopeInterface.asBinder();
    }

    private IKaleidoscopeInterface mKaleidoscopeInterface = new IKaleidoscopeInterface.Stub() {
        @Override
        public void drawLine(int x1, int y1, int x2, int y2) throws RemoteException {
            if(mIsLocalKaleidoscopeInterfaceBound) {
                try {
                    mLocalKaleidoscopeInterface.drawLine(x1, y1, x2, y2);
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            }
        }
    };

    private Intent createLocalServiceIntent() {
        // Create a basic intent using the intent filter action of
        // the local service as per its manifest configuration.
        Intent intent = new Intent("LocalKaleidoscopeService");

        // Attempt to resolve the service through the Android OS
        ResolveInfo info = getPackageManager().resolveService(intent, Context.BIND_AUTO_CREATE);

        // If the service failed to resolve it could mean that the
        // remote app is not installed or something wasn't configured
        // correctly, so we can't really start/bind the service...
        if(info == null) {
            return null;
        }

        // Otherwise, grab the resolved package name and service name and
        // assign them to the intent and bingo we have an explicit service intent!
        intent.setComponent(new ComponentName(info.serviceInfo.packageName, info.serviceInfo.name));
        return intent;
    }

    private ServiceConnection mLocalServiceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mLocalKaleidoscopeInterface = IKaleidoscopeInterface.Stub.asInterface(service);
            mIsLocalKaleidoscopeInterfaceBound = true;
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            mLocalKaleidoscopeInterface = null;
            mIsLocalKaleidoscopeInterfaceBound = false;
        }
    };
}

```

Cool, now run and deploy the updated *remote* app. Although we can't see it yet, the AIDL commands are going from the *local* app to the *remote* service, then back to the *local* app service.

Let's revisit the *local* app and have it respond to its own AIDL service commands!

**4b. Update local app and service**

Our *local* app activity will listen for broadcast events that we will emit from the *LocalKaleidoscopeService* when it receives AIDL commands from the *remote service*. 

A pure Android approach to this would be to use a *LocalBroadcastReceiver*, through which the *local* service will broadcast its AIDL commands as it receives them. Thats all well and good but its a bit cumbersome so I'll be using the [EventBus library by Green Robot](https://github.com/greenrobot/EventBus) instead of the built in Android system.

Follow the directions on the GitHub page for EventBus to get the library into your app.

I'll be creating a new *event* that is broadcast whenever our AIDL commands are received, which the main activity will listen for. Note that for simplicity this class has public fields instead of getters etc.

Move back to the *local* app again and create a new class for our event:

**DrawLineEvent.java**

```java
public class DrawLineEvent {
    public int x1;
    public int y1;
    public int x2;
    public int y2;

    public DrawLineEvent(int x1, int y1, int x2, int y2) {
        this.x1 = x1;
        this.y1 = y1;
        this.x2 = x2;
        this.y2 = y2;
    }
}
```

Then in your *MainActivity* add the following methods:

```java
@Override
protected void onResume() {
    super.onResume();
    EventBus.getDefault().register(this);
}

@Override
protected void onPause() {
    super.onPause();
    EventBus.getDefault().unregister(this);
}

@SuppressWarnings("unused")
public void onEventMainThread(@NonNull DrawLineEvent event) {
    Toast.makeText(this, "Local Main Activity received DrawLineEvent!", Toast.LENGTH_SHORT).show();
}
```

Basically we are just *subscribing* to the event bus in *onResume* and unsubscribing in *onPause*. Any time something broadcasts an instance of the *DrawLineEvent* class, we will receive it.

**4c. Broadcasting the AIDL events**

Edit the existing *LocalKaleidoscopeService* class and instead of displaying a *Toast*, we'll get it to use the *EventBus* to broadcast the event:

**LocalKaleidoscopeService.java**

```java
...

private final IKaleidoscopeInterface mKaleidescopeInterface = new IKaleidoscopeInterface.Stub() {
    @Override
    public void drawLine(int x1, int y1, int x2, int y2) throws RemoteException {
        EventBus.getDefault().post(new DrawLineEvent(x1, y1, x2, y2));
    }
};
```

Sweet, now run the updated *local* app again and you should be seeing the *Toast* message generated directly from the main activity (not the service itself) proving that:

- The *local activity* **received** the *EventBus* broadcast from the *local service*; because
- the *local service* **received** an *AIDL command* and then **broadcast** it as an *event*; because
- the *remote service* **sent** an *AIDL command* to the *local service*; because
- the *remote service* **received** an *AIDL command* from the *local application*; because
- the *local application* **sent** an *AIDL command* to the *remote service*; because
- we clicked a button...

**Did that completely do your head in or what!??** But its pretty damn cool if you ask me!

##5. Where's the colours!??

Ok so this whole blog post is called **AIDL Kaleidoscope** but so far I have failed to impress you with anything graphically interesting at all.

The core purpose of this blog post was to explore and illustrate the use of **AIDL interfaces** to allow **cross process / application** communication - a form of **IPC**.

So I guess now that all the boring theory is out of the way, we can have a bit of fun and try to make a not-too-lame graphical visualisation of what we have constructed.

I won't walk through the actual code for the next steps because it isn't inherently about AIDL, but you can find all the source code in the GitHub page for this blog post.

Here is a quick overview of the kaleidoscope idea though:

1. The user draws on the screen.
2. As the user draws, the *drawLine* AIDL commands are sent to the *remote* service.
3. The *remote* service will then mutate the drawing commands and relay them back in a different way.
4. The *remote* service will try to do something to cause the *local* app to render the drawn lines in a *kaleidoscope* fashion - mirrored copies of the original drawing commands.

Ok so the actual kaleidoscope effect ended up a bit lame but here is a preview of what it looks like:

![alt Finished app](/images/aidl/aidl_finished.jpg)

##6. Final thoughts

This was a fun exercise for me, trying out some techniques that are not typically part of my day to day Android development. I failed pretty hard on getting a convincing *kaleidoscope* visual effect, but the focus of this blog post was *IPC via AIDL communication*.

Some random thoughts:

- IPC via AIDL is not extremely difficult - you just need to make sure things are lined up the correct way in the manifest entries.
- In a way it's kinda like a giant [dependency injection](https://en.wikipedia.org/wiki/Dependency_injection) system - once the *remote service* is deployed, it can be updated as many times as needed with different implementations of the AIDL interfaces and the consuming applications never need to be updated.
- It would be a great vehicle for distributing work tasks amongst different processes, each of which could hold specific responsibility over its domain, again in a DI kinda way it could allow for hot swapping implementations.
- For applications with a common suite or with related product value, it could provide a way to integrate them together at a fairly deep level - and give the end user a seamless experience across the product suite.

Enjoy!

.:: END ::.

