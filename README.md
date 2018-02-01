Dimelo-iOS
==========

Dimelo provides a mobile messaging component that allows users of your app
to easily communicate  with your customer support agents. You can send text messages
and images, and receive push notifications and automatic server-provided replies.

The component integrates nicely in any iPhone or iPad app, allows presenting
the chat in tab, modal, popover or even customly designed containers and has
rich customization options to fit perfectly in your application.

Please refer to [Technical Website](http://mobile-messaging.dimelo.com) for other technical references.

Getting Started
---------------

Follow these steps to integrate the Dimelo Mobile SDK in your application.

1) Install the Dimelo library either via CocoaPods or manually (see below).

2) Configure the shared the `Dimelo` shared instance `+[Dimelo sharedInstance]`
 with your API secret (`-setApiSecret:secret:`), optional user identifier and other user-specific info. (See **Authentication** following section)

You can also create a `DimeloConfig.plist` file and add it to your project.
The library will use it to configure the `Dimelo` shared instance.
[See more informations about how to use plist to configure Dimelo](PlistCustomization.md)

3) Set `dimelo.developmentAPNS` = `YES` in your development builds (the milage may vary depending on your build strategy (TestFlight, Fabric.io ...) to receive push notifications. Set it back to `NO` when using a distribution provisioning profil.

To benefit from APNs on Dimelo Mobile SDK integration you will need to have properly configured and generated Apple APN cerficates and have them configured in your SMCC admin configuration interface.

4) Specify a delegate for the `Dimelo` shared instance `+[Dimelo sharedInstance]` with `dimelo.delegate` (usually it is your app delegate)

Implement `-dimeloDisplayChatViewController:` method. This method will be called by `Dimelo` when it needs to present a chat view. Your application determines how and where to display a chat view controller.

To display a chat, get its view controller using `-[Dimelo chatViewController]`
and present it either modally, in popover or in a UITabBarController.
See **Displaying Dimelo Mobile conversation screen** section below for more options.

5) In your app delegate, in  `-application:didRegisterForRemoteNotificationsWithDeviceToken:` set `deviceToken` property on your `Dimelo` instance. This will allow your app to receive push notifications from the Dimelo server when your agent replies to a user.

See **Push Notifications** for more detail.

6) Also in the app delegate, in `-application:didReceiveRemoteNotification:` pass the `userInfo` to `-[Dimelo consumeReceivedRemoteNotification:]`.

These are minimal steps to make chat work in your app. Read on to learn how to customize the appearance and behaviour of the chat to fit perfectly in your app.

See also **Sample Code** section in the end of this README or
download the [Sample App](https://github.com/dimelo/Dimelo-iOS-SampleApp).

Displaying Dimelo Mobile conversation screen
-------------------

Dimelo provides an opaque `UIViewController` instance that you can display how you want (created by `-[Dimelo chatViewController]`). You may put it as a tab in a `UITabBarController`, show in a popover or present modally. You can also use `transitioningDelegate` to present chat view controller in a very custom way.

You can present chat manually (e.g. user taps a button to open "support"),
or the chat may appear automatically (mediated by the delegate method `-dimeloDisplayChatViewController:`).

The chat is displayed automatically in two cases:

1. When the user launches or activates the app using a push notification.

2. When the user taps in-app notification bar that slides from the top in case
   a new message arrives while chat is not visible.

To display the chat manually and reuse the code in `-dimeloDisplayChatViewController:`, call `-[Dimelo displayChatView]`. When necessary, you can request another instance of a chat view controller (`-[Dimelo chatViewController]`) to show the chat in some other context.

Authentication
--------------

With each HTTP request, Dimelo sends a JWT ([JSON Web Token](http://self-issued.info/docs/draft-ietf-oauth-json-web-token.html)).
This token contains user-specific info that you specify (`userIdentifier`, `userName` etc.) and a HMAC signature. User identifier allows Dimelo to identify author of messages from different in the agent's backend. If user identifier is missing (`nil`), then an autogenerated unique installation identifier is used to identify the user (created automatically).

If your app rely on a uniq imutable technical identifier to identify user use `userIdentifier` to also identify them at the Agent interface level.

Use `userName` to provide to agent a human name for the user.

We support two kinds of authentication modes: with server-side secret and a built-in secret.

#### 1. Setup with a built-in secret

This is a convenient mode for testing and secure enough when user identifiers are unpredictable.

You configure `Dimelo` instance (`-[Dimelo sharedInstance]`) with the *secret* key (`-[dimelo setApiSecret:secret:]`) and it creates and signs JWT automatically when needed (as if it was provided by the server).
You simply set necessary user-identifying information and JWT will be computed on the fly.
You do not need any cooperation with your server in this setup.

The security issue here is that built-in secret is rather easy to extract from the app's binary build.
Anyone could then sign JWTs with arbitrary user identifying info to access other users'
chats and impersonate them. To mitigate that risk make sure to use this mode
only during development, or ensure that user identifiers are not predictable (e.g. random UUIDs).

#### 2. Setup with a server-side secret (better security but more complicated)

This is a more secure mode. Dimelo will provide you with two keys: a public API key and a secret key.
The public one will be used to configure `Dimelo` instance and identify your app.
The secret key will be stored on your server and used to sign JWT token on your server.

When you configure Dimelo with a public API key (`-initWithApiKey:hostname:delegate:`),
you will have to set `jwt` property manually with a value received from your server.
This way your server will prevent one user from impersonating another.

1. Set authentication-related properties (`userIdentifier`, `userName`, `authenticationInfo`).
2. Get a dictionary for JWT using `-[Dimelo jwtDictionary]`. This will also contain public API key, `installationIdentifier` etc.
3. Send this dictionary to your server.
4. Check the authenticity of the JWT dictionary on the server and compute a valid signed JWT token.
Use a corresponding secret API key to make a HMAC-SHA256 signature.
*Note:* use raw binary value of the secret key (decoded from hex) to make a
signature. Using hex string as-is will yield incorrect signature.
5. Send the signed JWT string back to your app.
6. In the app, set the `Dimelo.jwt` property with a received string.

You have to do this authentication only once per user identifier,
but before you try to use Dimelo chat. Your application should prevent
user from opening a chat until you receive a JWT token.



Push Notifications
------------------

Dimelo chat can receive push notifications from Dimelo server.
To make them work, three things must be done on your part:

1. Your app should register for remote notifications using `UIApplication` APIs.
   This is not strictly necessary as the Dimelo chat will attempt to do that automatically
   when user sends the first message. But if you are interested in sending notifications
   even before user has used the chat (e.g. a "welcome" message),
   then you should register for remote notifications earlier in the app's lifecycle.

2. Set the `Dimelo.deviceToken` property when you receive the token in `-application:didRegisterForRemoteNotificationsWithDeviceToken:`.

3. Set the `Dimelo.developmentAPNS` property to `YES` in your development builds
   in order to receive development push notifications. Do not forget to set this
   back to `NO` when using a distribution provisioning profil.

4. Let Dimelo consume the notification inside `-application:didReceiveRemoteNotification:`
   using `-[Dimelo consumeReceivedRemoteNotification:]`. If this method returns `YES`,
   it means that Dimelo recognized the notification as its own and you should not
   process the notification yourself. The chat will be updated automatically with a new message.

5. You will receive interactive push notifications with direct reply by default and need to forward the instant reply to Dimelo by calling `-[Dimelo handleRemoteNotificationWithIdentifier:responseInfo:]`  from the  `-application:handleActionWithIdentifier:forRemoteNotification:withResponseInfo:completionHandler:` callback. To disable this, set the  `Dimelo.interactiveNotification` property to  `NO`.

If the notification was received as a result of a user action
(e.g. user opened it from the lock screen), the chat will be displayed automatically.

When the notification is received while your app is running, but the chat is not visible,
Dimelo will display a temporary notification bar on top of the screen (looking similar
to a iOS notification bar when another application is notified).
If the user taps that bar, a chat will open.

You may control whether to display this bar or maybe show the notification differently
using `-dimelo:shouldDisplayNotificationWithText:` delegate method.
If you'd like to show your own notification bar, return `NO` from this method
and use `text` argument to present the notification using your own UI.



Badge Count
-----------

By default, `Dimelo` updates app's badge number automatically based on number of unread messages.
If your application has its own unread count, you might want to disable this behaviour by setting
`updateAppBadgeNumber` property to `NO`. Then you can access the Dimelo-only unread count using `unreadCount` property.


Required Permissions
--------------------

To access all the features of the Dimelo SDK, some permissions have to be defined inside your project info.plist:

|XCode Key|Raw Info.plist Key|Used for|
|---------|------------------|--------|
|Privacy - Photo Library Usage Description|NSPhotoLibraryUsageDescription|Used to allow the user to choose images from its photo album to send them as attachment|
|Privacy - Camera Usage Description|NSCameraUsageDescription|Used to allow the user to take pictures from its camera to send them as attachment|
|Privacy - Location When In Use Usage Description|NSLocationWhenInUseUsageDescription|User to allow the user to send a location as attachment|

If these permissions are not provided, the corresponding options will be hidden from the user.

Customizing Dimelo Mobile SDK Appearance
---------------------------

[see how to customize Dimelo using plist](PlistCustomization.md)

We provide a lot of properties for you to make the chat look native to your application.

For your convenience, properties are organized in two groups: Basic and Advanced.
In many cases it would be enough to adjust the Basic properties only.

You can customize global tint color, navigation bar colors and title.
You can also change the font and the color of any text in the chat view.

Advanced options include images and content insets for text bubbles.
We use 5 kinds of messages. Each kind can be customized independently.

1. User's text message.
2. User's image attachment.
3. Agent's text message.
4. Agent's image attachment.
5. System text notification (automatic reply from the server).

All bubble images must be 9-part sliced resizable images to fit arbitrary content.

Text bubbles may have images in two rendering modes: normal and template.
In *normal mode*, the image is displayed as-is.
In *template mode* it is colored using properties `userMessageBackgroundColor`, `agentMessageBackgroundColor` etc.
Default images use template mode.

If you provide a custom bubble image for text, you should also update
`{user,agent,system}MessageBubbleInsets` properties to arrange your text correctly within a bubble.

Attachment bubbles only use alpha channel of the image to mask the image preview.
Insets do not apply to attachment bubbles.

Check the [API reference](http://rawgit.com/dimelo/Dimelo-iOS/master/Reference/html/index.html) to learn about all customization options.

Localization
------------

- add the localizations to your project:

<p align="center">
<img src="https://s26.postimg.org/ab44y4x7d/demo.png"/>
</p>

- Create a DimeloLocalizable.strings file in Supporting File directory
- Check the supported languages.

<p align="center">
<img src="https://s26.postimg.org/n526rhamx/localization.png"/>
</p>

- add these keys to DimeloLocalizable.strings file if you want to change the default dimelo translation

### dimeloChatTitle
Title of the conversation window. 

This will be used as an argument to NSLocalizedString.


<p align="center">
<img src="http://s12.postimg.org/d3eymmfm5/title.png"/>
</p>

### dimeloSendButtonTitle
Title of the chat send Button. 

This will be used as an argument to NSLocalizedString.


<p align="center">
<img src="https://s26.postimg.org/tnlucx82h/IMG_0010.png"/>
</p>

### dimeloChoosePhotoButtonTitle
Title of the attachment photo button. 

This will be used as an argument to NSLocalizedString.


<p align="center">
<img src="https://s26.postimg.org/ph0zxl8gp/choose_titles.png"/>
</p>

### dimeloLocationButtonTitle
Title of the attachment location button. 

This will be used as an argument to NSLocalizedString.


<p align="center">
<img src="https://s26.postimg.org/ph0zxl8gp/choose_titles.png"/>
</p>

###  dimeloTakePhotoButtonTitle
Title of the Take Photo button. 

This will be used as an argument to NSLocalizedString.


<p align="center">
<img src="https://s26.postimg.org/ph0zxl8gp/choose_titles.png"/>
</p>

### dimeloCancelButtonTitle
Title of the Cancel button. 

This will be used as an argument to NSLocalizedString.


<p align="center">
<img src="https://s26.postimg.org/ph0zxl8gp/choose_titles.png"/>
</p>

### dimeloLoadMoreMessagesButtonTitle
Title of the Load more messages button. 

This will be used as an argument to NSLocalizedString.


<p align="center">
<img src="https://s26.postimg.org/az3splh5l/load_more_messages_title.png"/>
</p>


Reacting To Dimelo Mobile SDK Events
-----------------------

We provide two ways to react to various events in the char:

1. By implementing a `DimeloDelegate` protocol.

2. By subscribing to `DimeloChat*Notification` notifications.

For your convenince all notifications have a corresponding delegate method (e.g. `-dimeloUnreadCountDidChange:`), so you don't have to subscribe to them explicitly if you
wish to process events in `DimeloDelegate` object.

Two particular events that might be interesting to you are `-dimeloDidBeginNetworkActivity:`, `-dimeloDidEndNetworkActivity:`. Dimelo does not change the status bar network activity
indicator to avoid conflicts with your app. If you would like to indicate it, you should
override these methods and update the activity indicator accordingly.

Use `-onOpen:` and `-onClose:` events to get informations using `dimelo` parameter when the chat view is just opened or closed.

Please refer to [API reference](http://rawgit.com/dimelo/Dimelo-iOS/master/Reference/html/index.html) documentation for more information.



How To Install With CocoaPods
-----------------------------

1) Configure your project for CocoaPods (create a `Podfile` and a `xcworkspace`). Refer to [CocoaPods documentation](http://cocoapods.org) to learn how to do it.

2) Add "Dimelo-iOS" to your Podfile like so:

    pod 'Dimelo-iOS', :git => 'https://github.com/dimelo/Dimelo-iOS.git'

3) Run `pod install` to update your project.

4) Include header in your app delegate: `#include "Dimelo.h"`

5) Follow the [API reference](http://rawgit.com/dimelo/Dimelo-iOS/master/Reference/html/index.html) to configure and use Dimelo instance.



How To Install Manually
-----------------------

1) Download contents from Github [Dimelo-iOS project](https://github.com/dimelo/Dimelo-iOS).

2) Drop the `Dimelo.framework` subfolder in your project.

3) Link your target with these frameworks:

 * Dimelo.framework
 * Accelerate.framework
 * MobileCoreServices.framework
 * SystemConfiguration.framework
 * CoreLocation.framework
 * MapKit.framework
 * AddressBookUI.framework

4) Include header in your app delegate: `#include "Dimelo.h"`

API Reference
-------------

[Dimelo Mobile iOS SDK API reference](http://rawgit.com/dimelo/Dimelo-iOS/master/Reference/html/index.html).  Alternatively, you may consult `Dimelo/Dimelo.h` file.

Sample Code
-----------

The following code is a minimal configuration of Dimelo chat. It presents
the chat, handles push notifications and displays network activity in status bar.

DimeloConfig.plist
 
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>title</key>
    <string>Dimelo Mobile Support</string>
</dict>
</plist>
```



AppDelegate.m

```
#import "AppDelegate.h"
#import "Dimelo.h"

@interface AppDelegate () <DimeloDelegate>
@end

@implementation AppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    Dimelo* dimelo = [Dimelo sharedInstance];   
    dimelo.delegate = self;

    //Authentify using build-in authentification
    NSString* secret = @"YOUR SECRET API KEY";
    [dimelo setApiSecret:secret];

    // Authentify the user if you have an internal user_id otherwise this
    // is random
    dimelo.userIdentifier = @"application-user-id";
    dimelo.userName = @"John Doe";

    // Indicated in which environment your app is build
    // to receive APNs
    dimelo.developmentAPNS = NO;

    dispatch_async(dispatch_get_main_queue(), ^{

        [[Dimelo sharedInstance] displayChatView];

    });

    return YES;
}

- (void)application:(UIApplication *)application didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken
{
    // Register the device token.
    [Dimelo sharedInstance].deviceToken = deviceToken;
}

- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo
{
    if ([[Dimelo sharedInstance] consumeReceivedRemoteNotification:userInfo])
    {
        // Notification is consumed by Dimelo, do not do anything else with it.
        return;
    }

    // You app's handling of the notification.
    // ...
}


#pragma mark - Dimelo API callbacks


- (void) dimeloDisplayChatViewController:(Dimelo*)dimelo
{
    UIViewController* vc = [dimelo chatViewController];

    vc.navigationItem.rightBarButtonItem = [[UIBarButtonItem alloc] initWithBarButtonSystemItem:UIBarButtonSystemItemDone target:self action:@selector(closeChat:)];

    [self.window.rootViewController presentViewController:vc animated:YES completion:nil];
}

- (void) closeChat:(id)_
{
    [self.window.rootViewController dismissViewControllerAnimated:YES completion:nil];
}

- (void) dimeloDidBeginNetworkActivity:(Dimelo*)dimelo
{
    [UIApplication sharedApplication].networkActivityIndicatorVisible = YES;
}

- (void) dimeloDidEndNetworkActivity:(Dimelo*)dimelo
{
    [UIApplication sharedApplication].networkActivityIndicatorVisible = NO;
}


@end

```





