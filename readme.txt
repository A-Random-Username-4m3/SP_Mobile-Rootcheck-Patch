TODO:
- Add screenshots so this is not just a text file
- Prevent the app stopping updates when it gets too old



Attached is a patched APK that removes the root check of the SP mobile app.

I did this so that I could run this on my rooted phone running a degoogled LineageOS.

I found it strange that a relatively mundane app like this would even need to check for root. Then again, I saw a crap tonne of analytics and useless bloatware (most of which comes from the included libraries), so maybe someone wants to phone home your data to their servers unimpeded. Maybe someone nose why is it so. Just a thought, did not really look into it that much. Might want to examine and disable these next time if needed, when I get the time.

But anyway, props to the devs for not obsfucating the app. Please NEVER obfuscate this!!! I would probably have given up immediately and not learned a thing. 

This is actually a relatively simple thing to do, just a single patch, however since I am admittedly not very bright (and lazy too!), this took me like a few weeks and 200 slices of pizza and 600 liters of mountain dew to actually do this.

Anyway, below is the process of what exactly I did, including tools used, etc. Really learnt a lot from this, regarding Android Application Reverse Engineering.

Note that this is NOT meant to be an instruction manual, this is simply a documentation of my exact thought processes throughout this.

Firstly I used JADX gui (https://github.com/skylot/jadx) to open up the app and spent the next 3 days scrolling around the code uselessly trying the following couple of things:
- the string "We have detected that your mobile device is rooted/jailbroken".
- any function that relates to checking root

I did find something related to the 2nd one (@sources\com\google\firebase\crashlytics\internal\common\CommonUtils.java -> isRooted()). However, when I tried to trace its usages, I couldn't really find anything to the root checker. Nevertheless, I patched the smali for that for this function to always return false, but it did not work. 

And regarding the string "We have detected...", I to find it using the searcher, but I did not see anything, so I thought, perhaps JADX was just faulty, so I reopened the decompile in VSCodium and searched. Also nothing.

But I keep seeing the word "Xamarin" popping up everywhere, so I looked it up, and I found out that the app was built with Xamarin, an application framework to run C# programs crossplatform.

Apparently, Xamarin packs its code into an assemblies.blob file as per this article (https://thecobraden.com/posts/unpacking_xamarin_assembly_stores/). To extract, the article recommends that you use this python package called pyxamstore (https://github.com/jakev/pyxamstore). But there was an error `No module named setuptools`. So I looked at the issues, and seemed that other people were having it too, and the author seemed to have abandoned the project.

Thus I looked for forks to see if anyone had updated the tool. Luckily there was one (https://github.com/AbhiTheModder/pyxamstore)

Alright I finally got the Xamarin unpacker working with `pyxamstore unpack -d ./unknown/assemblies/`. I crossed my fingers and dragged SPMobile.MAUI.dll into dnSpy and it worked! Yay. Looked for anything related to the root checker and I found it immediately:

```
if (Dependency.HardwareSecurity != null && Dependency.HardwareSecurity.IsRooted()) {
	DefaultInterpolatedStringHandler defaultInterpolatedStringHandler = new DefaultInterpolatedStringHandler(201, 4);
	defaultInterpolatedStringHandler.AppendLiteral("We have detected that your mobile device is rooted/jailbroken.");
	...
	Dependency.HardwareSecurity.TerminateApp();
}
```

I used the IL explorer to turn the whole branch into a NOP slide so the code does not even run the root check.

Once that's done, I repacked the code `pyxamstore pack` and thats it! Recompiled the apk and tested it...and it crashed...

Strange, perhaps my patch caused it to crash, so I retried again, except this time I did a relatively harmless patch of changing a single character of a random string...also crashed

Maybe pyxamstore is not reliable? I unpacked and repacked the assemblies.blob file without patching anything but this time it worked.

But then I remembered there were some file with similar names to those unpacked from assemblies.blob, except they're in the format `libaout-*.dll.so`. Thus I made a hypothesis:

These are the generated AOT binaries from original compilation of the app, and the fact that the AOT and JIT binaries don't match causes the crash.

So how do I fix this? After searching online, I found out I could just simply strip the AOT binaries (https://stackoverflow.com/questions/35966941/is-it-possible-to-remove-aot-assemblies-from-third-party-apk) from the app. I then ran `rm libaot-*.dll.so` for each of the architectures, did the antirootcheck patch, recompiled the APK and it worked, at least in a virtual android machine which was not rooted.

I then tested it on my phone (which was rooted), and it worked! Finally its done.
