# Newgrounds.hx

Before using this library make sure you have read the 
<a href="http://www.newgrounds.io/">Introduction to Newgrounds.io</a>!

## Installing the library

**using haxelib:** (not implemented yet)
`haxelib install newgrounds.io`

just use git for now...

`haxelib git newgrounds.io https://github.com/Geokureli/Newgrounds.hx`

## Implement an instance of io.newgrounds.core into your game:

**OpenFL:** add `<haxelib name="newgrounds.io" />` to your project.xml (not implemented). 
You can also just include the local library in your xml via `<classpath path="../[libr path]/lib/src" />`

If you don't want to include openfl in your project, or you just hate my shitty core helpers, 
you can enable the compiler flag `ng_lite`. and it removes all openfl dependencies, 
but limits NG.core's functionality to basic component calls and responses

### Creating the core

`NG.create("app id here", "session id, here, if you know it");`

Once the core is created you can access it via NG.core but this is not possible if the core was instantiated directly.

When your game is being played on Newgrounds.com you can find the sessionId in the loaderVars,
or you can have the API find it automatically with

`NG.createAndCheckSession(myGame.stage, "app id here");`


This will also determine the host that will be used when logging events. You can also set or change 
the host using `NG.core.host`. The host is used to track views and various other events logged to NG.io.

### Manual Login

If no session ID was found, you will need to start one.

```
if (NG.core.loggedIn == false)
    NG.core.requestLogin(function():Void { trace("logged on"): });
```

### Encryption

Setting the encryption method is easy, just call:

`NG.core.initEncryption("encryption key", someEncryptionCipher, someEncryptionFormat);`

Encryption Ciphers:
- **io.newgrounds.crypto.Cipher.NONE**
- **io.newgrounds.crypto.Cipher.AES-128** (not implemented)
- **io.newgrounds.crypto.Cipher.RC4** (default)

Encryption Ciphers:
- **io.newgrounds.crypto.EncryptionFormat.BASE_64** (default)
- **io.newgrounds.crypto.EncryptionFormat.HEX** (not implemented)

You can also use your own encryption method - if you're some kind of crypto-god from The Matrix -
by directly setting NG.core.encryptionHandler

#### Example
```
NG.core.encryptionHandler = myEncryptionHandler;

function myEncryptionHandler(data:String):String {
    
    var encrytedData:String;
    // stuff
    return encrytedData;
}
```

## Using fla assets
If your project already uses a .swf you can add them to your .fla 
and they will automatically listen to your core for events. 
You can also instantiate them in code. These assets work with ng_lite enabled (with caveats)

**MedalPopup:** Just add it where you want it to show and it will 
autoplay when you call medal.unlock(), and the server response with a success event. 
If multiple achievements are unlocked at the same time they will play one after another

_**Note:** If the ng_lite compiler flag is true this will not automatically appear,
you must call playAnim(iconDisplayObj, medalName, medalPoints).
If ng_lite is false MedalPopup will request medals as soon as you start a NG.io session_

**ScoreBrowser:** Once it's added to the stage and a NG.io has started it loads board data, 
it has the following public properties
 - **boardId:** The numeric ID of the scoreboard to display. Defaults to the first ID sent back from the server.
 - **period:** The time-frame to pull scores from (see notes for acceptable values). Defaults to all-time
 - **title:** The title of the scoreBrowser, defaults to whatever the swf already has.
 - **tag:** A tag to filter results by
 - **social:** Whether to only list scores by the user and their friends, defaults to false

## Using Core Objects
Using core methods will cause the core to automatically keep track of server data in the underlying calls. 
much like how `NG.core.requestLogin()` stores the resulting sessionId for future calls, Medal and Scoreboard 
data is maintained from NG.core methods, but not direct `NG.core.calls` 

### Medals 
Use `NG.core.requestMedals()` to populate `NG.core.medals`, once Medal objects are created 
you can interface with them directly. For instance: 
```
var medal =  NG.core.medals.get(id);
trace('${medal.name} is worth ${medal.value}');

if (!medal.unlocked) {
    
    medal.onUnlock.add(function ():Void { trace('${medal.name} unlocked:${medal.unlocked}'); });
    medal.unlock();
}
```

### ScoreBoards
Just like Medals `NG.core.scoreBoards` is auto populated from `NG.core.requestScoreBoards` 
which allows you make postScore and getScores calls directly on the board.

**Note:** ScoreBoard instances persist across multiple requestScoreBoards calls, but a ScoreBoard's score instances do not

## Calling Components and Handling Results
You can talk to the NG.io server directly, but NG.core won't automatically handle 
the response for you (unlike NG.core.requestMedals()). All of the component calls are 
in `NG.core.call.[componentName].[callName]("call args")`

#### Example:
```
var call = NG.core.calls.medal.unlock(medalId);
call.send();
```

### Handling responses
You can add various listeners to a call to track successful or unsuccessful responses from the NG server.

```
var call = NG.core.calls.medal.unlock(medalId);
call.addDataHandler(onMedalUnlockDataReceived);
call.send();
```

The various calls types result in different response data structures. For instance medal.unlock 
responds with a `Response<io.newgrounds.objects.events.MedalUnlockResult>` object. The response type determines the data 
contained in `myResponse.result.data`. 

#### Example Usage:

```
var call = NG.core.calls.medal.unlock(medalId);
call.addDataHandler(
    function(response:Response<MedalUnlockResult>):Void {
        
        if (response.success && response.result.success) {
            
            var data:MedalUnlockResult = response.result.data;
            trace('Medal unlocked, [name=${data.medal.name}] [total NG medal points=${data.medal_points}]');
        }
    }
);
call.send();
```

### Error Handling

If response.success is false, response.result is null, and response.error will have the error info.
If response.result.success is false, response.result.data is null, and response.result.error will have the error info.

You can use `myCall.addSuccessHandler(function():Void { trace("success"); });` 
to only listen for successful responses from the server

You can also use myCall.addErrorHandler to listen for errors thrown by NG server, or errors
 resulting from general Http remoting

```
myCall.addErrorHandler(
    function(e:io.newgrounds.objects.Error):Void {
        
        trace('Error: $e');
    }
);
```

### Chaining call methods
All Call methods support chaining, meaning you can setup your calls without using local vars.
```
NG.core.call.medalUnlock(id)
    .setProperty("debug", true)
    .addSuccessHandler(onSuccess)
    .addErrorHandler(onFail)
    .addStatusHandler(onStatusChange)
    .send();
```

### Queueing calls
All calls can be queued so that they are sent sequentially rather than sending them all at once.

```
NG.core.session = "session id here";
NG.core.app.checkSession().queue();
NG.core.medal.unlock(id).queue();

```

## TODO
 - better readme.md
    - hxml instructions
 - add to haxelib
 - AES-128 encryption
 - Hex encoding
 - kill all humans
 - flash API assets
     - ad viewer - not supported in ng.io v3
     - auto connector - requires ads?
 - continuous integrations
 - local storage
    - save unsent medal unlocks and scoreboard posts
    - save previous session rather than creating a new one
