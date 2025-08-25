---
title: "BrunnerCTF 2025 - mobile"
layout: single
date: 2025-08-25
categories: blog
---

# mobile_rivalcakes (easy)

Given this challenge description and a file dump, we need to find a password and a location:

```
The secret Brunner and Othello cults are waging their battles in the streets of Denmark.

One day, you see a member of the Othello cult drop their phone out of their pocket. You grab it, but it seems to be a burner phone. You find only two custom apps, but when opening one, it instantly deletes itself - maybe there is still some data left?

In the cake cults, preferences matter - so keep an eye out and mind what you say.

Your Othello rivals are planning a secret meetup, find out where and find the password to infiltrate them!

Flag format: The flag is formatted as brunner{<w3w_code>_<password>}, where <w3w_code> is the what3words code of the location, without leading slashes. Make sure your "3 word address language" is set to English.
```

While clicking through the files, I quickly found `data/data/dk.brunnerctf.rivalcakes`, which matched the challenge description. An initial check for `password`, `loc`, or `lat` did not bring anything up. I first tried to get an overview:

```bash
❯ tree .
.
├── app_textures
├── app_webview
│   ├── Default
│   │   ├── blob_storage
│   │   │   └── ac572a83-ec69-47ee-8bb1-6a8869725a84 <- INTERESTING
│   │   ├── Local Storage
│   │   │   └── leveldb
│   │   │       ├── 000005.log
│   │   │       ├── CURRENT
│   │   │       ├── LOCK
│   │   │       ├── LOG
│   │   │       ├── LOG.old
│   │   │       └── MANIFEST-000004
...
├── cache
│   ├── Crash Reports
│   │   └── ANR Variations
│   │       └── b65c4b12d7782aaf3ebc0b927a63a2b9 <- INTERESTING
...
├── databases
│   ├── orders.db <- INTERESTING
│   ├── orders.db-journal
│   ├── orders.db-shm
│   └── orders.db-wal
├── files
│   ├── menu_temp.html
│   └── profileInstalled <- INTERESTING
└── shared_prefs
    ├── my_prefs.xml <- INTERESTING
    └── WebViewChromiumPrefs.xml
```

After seeing this, I started by examining the `orders.db` database. The `file` command told me it is an SQLite database.

```bash
sqlite> open orders.db;
sqlite> select * FROM orders limit 100;
15|Customer15|note 15|cancelled|2025-07-07 10:03:00
16|Alice|Extra frosting|delivered|2025-08-10 14:32:11
17|Bob|Meet at location in EXIF|pending|2025-08-11 09:21:44
18|Claire|part1:Othello_is_|cancelled|2025-08-12 08:59:02
20|Customer20|note 20|delivered|2025-07-03 10:02:00
```

This gave me two important clues:
1.  `Meet at location in EXIF` implies there are images that have the location for the w3w code inside their EXIF metadata.
2.  The flag is split into parts, and I found the first one: `part1:Othello_is_`.

Since I had `part1`, I used `ripgrep` (a seriously amazing tool!) to scan for the other parts.

```bash
❯ rg -e "part\d"
data/data/dk.brunnerctf.rivalcakes/files/menu_temp.html
1:<html><body><p>part4:Brunsviger</p></body></html>

data/data/dk.brunnerctf.rivalcakes/shared_prefs/my_prefs.xml
4:    <string name="step2_data">part2:always_better_</string>
```

Nice, this meant `part3` was likely still missing. Checking `data/data/dk.brunnerctf.rivalcakes/shared_prefs/my_prefs.xml`, I even found a path to an image:

```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <string name="recipe_file">/storage/emulated/0/Download/Cakes/best_cake_in_the_world.jpg</string>
    <string name="step2_data">part2:always_better_</string>
</map>
```

I checked that image with `exiftool` and hoped for the best:

```
❯ exiftool best_cake_in_the_world.jpg
======== best_cake_in_the_world.jpg
ExifTool Version Number         : 13.30
File Name                       : best_cake_in_the_world.jpg
...
User Comment                    : part3:Than_an_ugly_
...
GPS Version ID                  : 2.3.0.0
GPS Latitude Ref                : North
GPS Longitude Ref               : East
...
GPS Latitude                    : 55 deg 40' 29.56" N
GPS Longitude                   : 12 deg 33' 52.40" E
GPS Position                    : 55 deg 40' 29.56" N, 12 deg 33' 52.40" E
```

In there, I found `part3` and a GPS location, which I translated with the what3words website to get the code: `55° 40' 29.56" N, 12° 33' 52.40" E` $\implies$ `scarred.marble.improving`

Putting it all together:
> brunner{scarred.marble.improving_Othello_is_always_better_Than_an_ugly_Brunsviger}

# mobile_fridaycake (medium)

The challenge provided an APK with the following description:

```
It's Friday - your family cake night! 🍰 But when you try to order your favorite cake, you find that someone has changed the app and locked the ordering screen!

Recover the Secret Access Code and save the family's cake tradition.
```
I quickly installed the APK via drag-and-drop into my Android Studio emulator to see what I was up against:

![MainActivityScreen](/assets/images/2025-08-14-uiuctf_ai-clippy/image.png)

However, I needed to look at the code to see how I could get past this. My first tool to look at APKs is [JADX](https://github.com/skylot/jadx). I navigated to `AndroidManifest.xml` in the `Resources` tab to discover the starting point of the APK, the `MainActivity`. It was `dk.brunnerctf.fridaycake.MainActivity`, which brought me to this code:

```java
    public static final void onCreate$lambda$0(EditText editText, MainActivity mainActivity, View view) {
        if (Authenticator.INSTANCE.checkCode(editText.getText().toString())) {
            Toast.makeText(mainActivity, "✅ Cake unlocked! Enjoy your Friday! 🎂", 1).show();
        } else {
            Toast.makeText(mainActivity, "❌ Wrong code! No cake for you...", 0).show();
        }
    }
```

The `Authenticator.INSTANCE` seemed like an obvious spot to check. I found the `checkCode` function, which in turn calls `NativeChecker.INSTANCE.verifyCode(...)`.

```java
    public final boolean checkCode(String input) {
        Intrinsics.checkNotNullParameter(input, "input");
        String str = StringsKt.reversed((CharSequence) input).toString() + "::CAKE::";
        ArrayList arrayList = new ArrayList(str.length());
        for (int i = 0; i < str.length(); i++) {
            arrayList.add(Character.valueOf((char) (str.charAt(i) + 2)));
        }
        return NativeChecker.INSTANCE.verifyCode(CollectionsKt.joinToString$default(arrayList, "", null, null, 0, null, null, 62, null));
    }
```

Let's continue and click on `NativeChecker` to see where it leads:

```java
public final class NativeChecker {
    public static final NativeChecker INSTANCE = new NativeChecker();

    public final native String getDecryptedFlag();

    public final native boolean verifyCode(String code);

    private NativeChecker() {
    }

    static {
        System.loadLibrary("native-lib");
    }
}
```

The two functions in `NativeChecker` are implemented via a JNI interface, where Java calls a compiled library. The library `native-lib` is also shipped in the APK at `Resources/lib/<Architecture>/libnative-lib.so`. I didn't bother to continue to reverse-engineer here because there is a `getDecryptedFlag` function I wanted to call first. This is a perfect use case for [Frida](https://frida.re/), a dynamic instrumentation toolkit. It allows us to inject a JavaScript interpreter into a process (in this case, the APK running on the emulator) and modify it.

**DISCLAIMER**: You need to click the "Unlock Cake" button once, as Java lazy-loads the library!

```bash
[Android Emulator 5554::FridayCake ]->
    Java.perform(() => {
        var checker = Java.use("dk.brunnerctf.fridaycake.NativeChecker"); // Class reference
        var checkerInstance = checker.$new(); // Create a new object
        var result = checkerInstance.getDecryptedFlag(); // Get our flag
        console.log("Flag: " + result);
    })
Flag: brunner{Y0u_Us3d_Fr1d4_F0r_Gr4bb1ng_Th1s_R1ght?}
[Android Emulator 5554::FridayCake ]->
```

# mobile_brod-and-co (hard)

The challenge provided an APK with the following description:

```
Brød & Co. just released their new ordering app, but their prices are a bit high. If only I had a coupon code...

Note: The app sometimes crashes when clicking "Place Order". If this happens, try again or try another approach.
```

As with the previous challenge, I started with JADX. All I got from the `MainActivity` was that the app was built with Flutter:

```java
package dk.brunnerne.masterbaker;

import io.flutter.embedding.android.FlutterActivity;

/* loaded from: classes.dex */
public final class MainActivity extends FlutterActivity {
}
```

I had no clue about Flutter, so naturally, I did something else first. I later came back to the challenge to understand what Flutter is and how I could overcome the hurdle:

1.  Flutter Apps are written in Dart and, depending on the compilation mode, compiled to native code.
2.  We have an AOT snapshot for Android arm64. The file `lib/arm64-v8a/libapp.so` contains the compilation flags, e.g., `product no-code_comments no-dwarf_stack_traces_mode dedup_instructions no-tsan no-msan arm64 android compressed-pointers`.

I read and watched these resources:

*   https://blog.tst.sh/reverse-engineering-flutter-apps-part-1
*   https://www.guardsquare.com/blog/current-state-and-future-of-reversing-flutter-apps
*   https://www.youtube.com/watch?v=JNoEUPlgcZk

The last [link](https://youtu.be/JNoEUPlgcZk?feature=shared&t=2160) points out when we can disassemble the app. However, I first tried to understand why the app included a `libnative.so` library, which is not standard for Flutter apps (they typically only ship `libapp.so` and `libflutter.so`).

## Jumping into `libnative.so`

First, I had a look to see if there was anything important for us:

```bash
❯ strings libnative.so | rg -i flag
FLAG:
FLAG|%s
FLAG|INVALID_COUPON
```

This looked promising. I would suspect that the library is called from Flutter via a native mechanism. I checked what methods were exported:

```bash
❯ nm -D libnative.so | grep "T"
0000000000002934 T buffer_manager
00000000000028d4 T buf_util
0000000000002978 T config_loader
00000000000028e4 T data_helper
0000000000002988 T force_data_preservation
00000000000024e0 T get_client_version
0000000000002694 T hidden_decrypt_function
00000000000024ec T hidden_encode_function
00000000000025b4 T hidden_log_function
0000000000001f6c T process_data_complete
0000000000002a9c T test_all_functions
0000000000002768 T util_func_a
00000000000027bc T util_func_b
0000000000002810 T util_func_c
00000000000028c8 T util_func_d
000000000000297c T version_info
```

With so many functions, it was hard to know where to start. My first instinct was to verify that some functions were called from the library. We can do this with `frida-trace`, which tells us which functions were called (*Make sure to use the app first so the libs are loaded!*):

```bash
❯ frida-trace -UF -i 'libnative.so!*'
 ... instrumenting ... 
 17221 ms  process_data_complete(COUPON:test)
```

So, I looked there. I already added comments for further functions. I opted to use the `High Level IL` in BinaryNinja instead of `Pseudo C`, as I found the `if` conditions in the Pseudo C output confusing.

```c
char* process_data_complete(char* coupon){
    // Normal use case via `Check Coupon`
    if (strncmp(coupon, "COUPON:", 7) == 0)
    {
        if (SomeDecryption(&coupon[7]) == 0)
            return strdup("INVALID_COUPON");
        
        return strdup("VALID_COUPON");
    }
    
    // Special case: Starts not with "FLAG:" (i.e. intended way is to submit a valid coupon request?)
    if (strncmp(coupon, "FLAG:", 5) != 0)
    {
        // seems like a hash/encryption algorithm that checks for H/E(x) == 0x5a7c2b19
        if (checkcoupon(coupon) != 0)
        {
            magic();
            char* x0_20 = malloc(__strlen_chk(&emptybuffer, -1) + 0x10);
            
            if (x0_20)
            {
                ...
                sub_7a68ceb99c(x0_20, -1, x0_21 + 0x10, "OK|%s", &emptybuffer, ...);
                return x0_20;
            }
        }
        
        // this opens a socket where data is send and received from
        return sendandreceivefromlocalhost(coupon);
    }
    
    // Other way, the string XXXXXhere_is_the_input with XXXXX != "FLAG:"
    if (SomeDecryption(&coupon[5]))
    {
        magic();
        char* x0_12 = malloc(__strlen_chk(&emptybuffer, -1) + 0x10);
        
        if (x0_12)
        {
            int64_t x0_13;
            int128_t v0_1;
            int128_t v1_1;
            int128_t v2_1;
            int128_t v3_1;
            int128_t v4_1;
            int128_t v5_1;
            int128_t v6_1;
            int128_t v7_1;
            x0_13 = __strlen_chk(&emptybuffer, -1);
            sub_7a68ceb99c(x0_12, -1, x0_13 + 0x10, "FLAG|%s", &emptybuffer, v0_1, v1_1, v2_1, 
                v3_1, v4_1, v5_1, v6_1, v7_1);
            return x0_12;
        }
    }
    return strdup("FLAG|INVALID_COUPON");
}
```

I built a decision tree to understand the logic:

1.  Input starts with `COUPON:`
    *   Some decryption evaluates to $0$ $\implies$ `INVALID_COUPON`
2.  Input does not start with `FLAG:`
    *   A hash is computed over the input. If it matches a certain value, `magic()` is called, and its output is passed to another function.
3.  Some decryption is performed on the coupon starting from the 5th character.
    *   Call `magic()` and pass its output to a function.

This didn't look good at all. We have some decryption/hash functions that somehow make our life harder. I actually copied the code of `magic` into ChatGPT, which told me that it's some kind of AES decryption. However, a small peek shows an important hint:

```c 
int64_t magic(){
    void var_28
    void* var_c0 = &var_28
    void* var_b8 = &emptybuffer
    
    if (staticCallOnce == 0)
        /*
            Loops and bitmagic removed
        */
        __memcpy_chk(dstpp: &emptybuffer, srcpp: srcpp_1, len: 0x2c, dstlen: 0x40)

        *(var_b8 + 0x2c) = 0
        int32_t var_a8_1 = 0
        
        for (int32_t i_3 = 0; i_3 s< 0x2c; i_3 += 1)
            if (zx.d(*(&emptybuffer + sx.q(i_3))) s>= 0x20
                    && zx.d(*(&emptybuffer + sx.q(i_3))) s<= 0x7e)
                int64_t x10_36 = sx.q(var_a8_1)
                var_a8_1 = x10_36.d + 1
                *(&emptybuffer + x10_36) = *(&emptybuffer + sx.q(i_3))
        
        *(&emptybuffer + sx.q(var_a8_1)) = 0
        staticCallOnce = 1
    
    return &emptybuffer
}
```

Based on the function, I thought it's initializing `emptybuffer` only one time and reusing the buffer later via `callOnce`. Both vars are in the `.bss` segment. This seems interesting because the buffer is later used for the flag! But first, we need to understand what `sub_7a68ceb99c` does in our snippet:

```c
    if (SomeDecryption(&coupon[5]))
    {
        magic();
        char* x0_12 = malloc(__strlen_chk(&emptybuffer, -1) + 0x10);
        
        if (x0_12)
        {
            ...
            x0_13 = __strlen_chk(&emptybuffer, -1);
            sub_7a68ceb99c(x0_12, -1, x0_13 + 0x10, "FLAG|%s", &emptybuffer, ...);
            return x0_12;
        }
    }
```

The functions looks confusing due the the variable arguments stack setup of `__vsnprintf_chk`: 

```c
// sub_7a68ceb99c(x0_12, -1, x0_13 + 0x10, "FLAG|%s", &emptybuffer, v0_1, ...);
uint64_t sub_7a68ceb99c(char* arg1, size_t arg2, size_t arg3, char* arg4, ...){
    int128_t var_b0 = arg13
    ...
    int128_t* var_48 = &var_80
    int128_t var_70 = (&var_a0).o
    void arg_0
    var_80 = (&arg_0).o
    return zx.q(__vsnprintf_chk(s: arg1, maxlen: arg3, flag: 0, slen: arg2, format: arg4))
}
```

The documentation for [`__vsnprintf_chk`](https://refspecs.linuxbase.org/LSB_4.1.0/LSB-Core-generic/LSB-Core-generic/libc---vsnprintf-chk-1.html) tells us that `s` is the output buffer, `format` is the `printf` style format, and all arguments after are `variable arguments`. I assume that due to the `variable arguments`, everything is placed on the stack, which gets lost during decompilation. However, we have enough to play mix and match.

```
            magic();
                |
                |-- Writes into the buffer ---|
                                              |
            char* x0_12 = malloc(__strlen_chk(&emptybuffer, -1) + 0x10);
                     |
                     |        x0_13 = __strlen_chk(&emptybuffer, -1);
                     |          |
    sub_7a68ceb99c(x0_12, -1, x0_13 + 0x10, "FLAG|%s", &emptybuffer, v0_1, v1_1, v2_1, v3_1, v4_1, v5_1, v6_1, v7_1);
                     |     |            |        |
                     |     |            |        |--------------------|
                     |     |            |                             |  
                     |     |------------|---------------|             |
                     |                  |               |             |
                     |             |----|               |             |  
                     |             |                    |             |  
__vsnprintf_chk(s: arg1, maxlen: arg3, flag: 0, slen: arg2, format: arg4)
```

When we look at the other arguments passed to `sub_7a68ceb99c`, they are all uninitialized. The only meaningful argument is `&emptybuffer`, which should then contain our flag! My plan was to call `magic()` and then simply read the `emptybuffer`. Instead of recreating the conditions to call `magic()`, I found an exported function that does the job for us in the binary:

```c
int64_t buf_util(){
    magic()
    return &emptybuffer
}
```

Okay, I could easily call this with Frida. To do so, I needed to follow these steps:

1.  Get the function signature to create a `NativeFunction`.
2.  Get the module's in-memory base address.
3.  Rebase the binary in *BinaryNinja* to the module's base address (this makes copying the function address straightforward).
4.  Go to the function in *BinaryNinja* and copy its address.
5.  Execute the *Frida* script.

```bash
[Android Emulator 5554::MasterBaker ]-> Process.getModuleByName("libnative.so").base
"0x7a68cea000"
[Android Emulator 5554::MasterBaker ]-> var buf_util = new NativeFunction(ptr(0x7a68cec8d4),"pointer",[]);
[Android Emulator 5554::MasterBaker ]-> var res = buf_util();
[Android Emulator 5554::MasterBaker ]-> res.readCString();
"brunner{wh0_kn3w_dart_c0u1d_h4nd13_C?!}"
```

## Collection of Failed Ideas

I spend much time on this challenge and tried different loopholes. Just don't give up!

### Returning `VALID_TOKEN`

I tested what happens when the function returns `VALID_TOKEN`. With `frida-trace`, we also have the ability to intercept functions in a web interface:

```bash
>  frida-trace -UN dk.brunnerne.masterbaker -i 'libnative.so!*'
Started tracing 16 functions. Web UI available at http://localhost:61491/
```

So, instead of `INVALID_TOKEN`, I just made it return `VALID_TOKEN`.

```js
defineHandler({
  onEnter(log, args, state) {
    log(`process_data_complete(${args[0].readCString()})`);
  },

  onLeave(log, retval, state) {
    retval.replace(Memory.allocUtf8String("VALID_TOKEN"));
  }
});
```

Hmm, now I was getting a completely new input to the function:

```bash
153592 ms  process_data_complete(COUPON:test2)
155898 ms  process_data_complete({"customer":{...},"order":{...},"metadata":{...}})
```

However, it looked like there was no path in `process_data_complete` to parse JSON.

### Rewriting the input to `process_data_complete`

The second `if` statement checks that the input does not start with `FLAG:` (`if (strncmp(coupon, "FLAG:", 5) != 0)`). Therefore, I adjusted the above-defined handler to intercept the only argument to `process_data_complete`.

```js
defineHandler({
  onEnter(log, args, state) {
    args[0] = Memory.allocUtf8String("TEST")
    log(`process_data_complete(${args[0].readCString()})`);
  },

  onLeave(log, retval, state) {
    log(`process_data_complete() -> ${retval.readCString()}`);
  }
});
```

I kept receiving `ERROR: Connection failed`, which totally confused me. However, this was not a Frida error, which I erroneously believed at first. Going into the code of `sendandreceivefromlocalhost`, there is a call to `socket` and `connect`. The socket call is not allowed, however, as the permission `<uses-permission android:name="android.permission.INTERNET"/>` is missing. My idea then was to take the APK and add the permission:

1.  Decompile the APK via `apktool d <file>`.
2.  Add the permission to `AndroidManifest.xml`.
3.  Resign and realign with [uber-apk-signer](https://github.com/patrickfav/uber-apk-signer): `java -jar uber-apk-signer-1.3.0.jar --allowResign -a masterbaker/dist/MasterBaker.apk`.

When I opened a shell via `adb`, I received gibberish via `nc -s 127.0.0.1 -p 8088 -l`. I stopped working on this as I focused more on the `magic` method.