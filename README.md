# Playing Around with Security in Android 7.0

### by David Levitsky

The contents of this document are the result of an independent study project performed at Cal Poly during Winter Quarter of 2017. The project was composed of two phases:

* writing an Android application and ensuring it is encrypted using Android 7.0's file-based encryption API's
* attempting to retrieve sensitive data from memory by searching for gaps in encryption during booting up / locking / unlocking of the phone

Research on this project was conducted with the latest version of Android Studio at the time, version 2.2.2, a Nexus 5X device which was updated to run 7.0 as its operating system, and a Mac running OS X El Capitan. This post is written from the chronological perspective of instructions I executed and can hopefully be used as a guide to either continue this project at a later point in time, or to save someone else some time in the future.

## **Phase 1 - Application Development** ##

### Getting 7.0 ###
The first step was obtaining a phone that was eligible to receive the Android 7.0 update and installing it. Google rolls out software updates over the air (OTA), so depending on the device, it may or may not receive an upgrade. Some initial research showed that only devices manufactured in the past two years would be open to the update, so I settled with a Nexus 5X. Downloading 7.0 was as simple as opening a notification saying that the device had a software update available and setting aside a few hours for installation.


### Setting up for Development ###
To begin development directly on your device, you will have to click on "Settings", scroll to "About Phone", and tap it 7 times to enable Developer Mode (you can find exact instructions [here](http://www.androidcentral.com/how-enable-developer-settings-android-42)). You can now enable USB debugging and directly connect the device to your laptop via USB so you can transfer source files to and from your device using Android Studio.

This project was concerned with the new file-based encryption mechanism in 7.0, which is not the default encryption on the device. Be sure to turn on file based encryption under your device Settings. You will get a warning saying that it is in beta mode and your device data may be wiped upon enabling it, so be mindful of any important data you may have.

### Developing a Sample Application ###

The goal of my sample application was pretty simple - to poke around the [Google Developer API for Direct Boot](https://developer.android.com/training/articles/direct-boot.html) to see whether I can verify that FBE is enabled and access different parts of storage, such as device-encrypted and credential-encrypted. 

1. I wrote a simple Activity that populates a list.
2. Using a DevicePolicyManager, I could check the encryption status of the device using the getStorageEncryptionStatus function. The value returned was 5, which corresponds to the enum [ENCRYPTION_STATUS_ACTIVE_PER_USER](https://developer.android.com/reference/android/app/admin/DevicePolicyManager.html#ENCRYPTION_STATUS_ACTIVE_PER_USER). This enum is only available in API's 24 and up (Android 7.0 and up), and is only returned if FBE is active.

The code snippet looks something like this:

``` 
DevicePolicyManager localDPM = (DevicePolicyManager) getApplicationContext()             .getSystemService(getApplicationContext().DEVICE_POLICY_SERVICE);

int encryption_status = localDPM.getStorageEncryptionStatus();
```

where encryption_status indictates the type of encryption being utilized by the device. At this point in time, I verified that device was indeed running FBE in Android 7.0 and I could move on to trying to access different parts of memory.

### Testing out Direct Boot ###

One of the main features of 7.0 is the concept of direct boot, where Android provides you with "device-protected storage" that can be accessed even prior to you entering your PIN or password to unlock the device. *Direct Boot* refers to the state that your Android device is in upon being powered on (booting up) but without any inputted user credentials. In this state, your device should still be able to access *device-protected storage* for insensitive information, such as any scheduled alarms.

Not all applications should be able to run in Direct Boot mode, so in order to get my application permission to access device-protected storage, I had to request specific access to run during direct boot. This was accomplished with a line marking "directBootAware" as true at the beginning of my AndroidManifest.xml file for my application:

**android:directBootAware="true"**

```
<application
    android:allowBackup="true"
    android:directBootAware="true"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:supportsRtl="true"
    android:theme="@style/AppTheme">
```

To actually access my application's memory in this state, I had to set up a [BroadcastReceiver](https://developer.android.com/reference/android/content/BroadcastReceiver.html) to receive Intents (think system-level messages being sent and received). This was also accomplished in the Manifest.xml file. I made a class called MyReceiver, which extended BroadcastReceiver, and implemented the OnReceive function. Then, in my Manifest, I declared this Receiver.

```
<receiver android:name=".MyReceiver"
     android:exported="true"
     android:directBootAware="true">

     <intent-filter>
         <action android:name="android.intent.action.BOOT_COMPLETED" />
     </intent-filter>
     </receiver>
```

Note the declaration of the **intent-filter** as well. This specifies which intents your Receiver will listen to and process. At this point in time, I ran into my first issue with development during this project. When a device finishes booting and is in *Direct Boot* mode, a 
*LOCKED_BOOT_COMPLETED* intent is sent. When the device is unlocked, a *BOOT_COMPLETED* intent is sent. Theoretically, both of these Intents should be defined in the intent-filter of the manifest. Device-protected storage would be accessible when the *LOCKED_BOOT_COMPLETED* intent is sent, and all storage would be accessible upon the receival of the *BOOT_COMPLETED* intent. 

However, defining the *LOCKED_BOOT_COMPLETED* intent would cause my application to crash for reasons I could not determine. This was opened as an [issue](https://github.com/googlesamples/android-DirectBoot/issues/6) on Google's GitHub for DirectBoot in July 2016. I also posted a [question](http://stackoverflow.com/questions/41968212/using-broadcast-receiver-with-direct-boot-in-android-nougat-7-0?noredirect=1#comment71498828_41968212) on StackOverflow without much resolution. I lost a lot of time on trying to debug the problem, so I decided to forget about it and move forward - both device-protected storage and credential-protected storage are accessible after the device is unlocked.

## Committing Application Data to Memory ##
The way to commit application data to memory was through the use of the application's Context and through [SharedPreferences](https://developer.android.com/reference/android/content/SharedPreferences.html). The code below shows how to save data to device-protected storage, which does not require user authentication to be accessed:

```
Context c = getApplicationContext();
SharedPreferences settings = c.getSharedPreferences("PREFERENCES", 0);
SharedPreferences.Editor editor = settings.edit();
editor.putString("Test Key", "Test Value");
editor.commit();
```

The code below shows how to save data to credential-protected storage, which cannot be accessed until the user unlocks the device. As you can see, the only difference in the code is calling the **createDeviceProtectedStorageContext** function.

```
Context deviceProtected = c.createDeviceProtectedStorageContext();
SharedPreferences protectedSettings = deviceProtected.getSharedPreferences("PROTECTED", 0);
SharedPreferences.Editor protectedEditor = protectedSettings.edit();
protectedEditor.putString("Protected Key", "Protected Value");
protectedEditor.commit();
```

Getting the data from memory is straightforward as well. SharedPreferences uses key-value pairs, so you will need to query by the correct key. For device-protected:
```
Context c = getApplicationContext();
SharedPreferences settings = c.getSharedPreferences("PREFERENCES", 0);
String value = settings.getString("Test Key", "");

```
And for credential-protected:

```
Context deviceProtected = c.createDeviceProtectedStorageContext();
SharedPreferences pSettings = deviceProtected.getSharedPreferences("PROTECTED", 0);
String pVal = pSettings.getString("Protected Key", "");
```

At this point in time, we are able to commit to and query from the two separate types of memory storage.

## **Phase 2 - Memory Analysis** ##
In this phase, I wanted to be able to do a memory dump of my Android device and use an open-source tool to process the dump for sensitive information. It turns out that acquiring a RAM dump from an Android phone is more difficult than it sounds. Initially, I tried using Eclipse's Memory Analysis Tool to perform a heapdump and look through all of the strings contained in the application's memory. The tool is pretty cool and breaks down everything happening in the application, but since the encryption was happening by the OS itself, I needed a greater portion of memory to be dumped.

Two tools I found were [LiME](https://github.com/504ensicsLabs/LiME#example) and [Volatility](https://github.com/volatilityfoundation/volatility/wiki/Android). LiME needs to be loaded up into a device's kernel and then could perform a RAM dump in a certain format which Volatility could process. Volatility has some cool tools that specifically processed a RAM dump to look for things such as encrypted key formats. Because RSA keys start with a similar format, the Volatility tool identifies sections of memory that contain bytes similar to the initial bytes of an RSA key. A better explanation can be found [here](https://volatility-labs.blogspot.com/2014/01/truecrypt-master-key-extraction-and.html). Volatility says in its documentation that it supports LiME formats.

I started off with attempting to get LiME on my device. The exact instructions I used are found at this [link](https://github.com/volatilityfoundation/volatility/wiki/Android), but I did not get very far.

1. Initialize the Android Build Environment (Mac OS X)
```
$ hdiutil create -type SPARSE -fs 'Case-sensitive Journaled HFS+' -size 40g ~/android.dmg
$ hdiutil attach ~/android.dmg.sparseimage -mountpoint /Volumes/android
$ curl https://storage.googleapis.com/git-repo-downloads/repo > repo
$ chmod a+x repo
$ mkdir /Volumes/android/dev
$ cd /Volumes/android/dev
$ ~/repo init -u https://android.googlesource.com/platform/manifest
$ ~/repo sync
```
This sync took a few hours with occasional random crashes. Not sure if that was due to issues GitHub was experiencing during that week or for other reasons, but this step was time consuming.

```
$ source build/envsetup.sh
```
Here I ran into a stopping point. This is the command to run the script that I believe actually initalizes the build environment, but I could not find this script. I decided to see how far I could get without this step so I continued on.

2. Download the Android Kernel Source Code
```
$ git clone https://android.googlesource.com/kernel/msm.git ~/android-source
```
I used msm because this is the kernel for my Nexus 5X device. Google has a list of instructions for [downloading and building a kernel](http://source.android.com/source/building-kernels.html) that contains this information. I ended up referring to this guide rather than the original instructions I was looking at because they were not working for me.

3. Cross-Compile the Kernel

Setting these environmental variables is supposed to help with the cross-compilation process.
```
$ export ARCH=arm
$ export SUBARCH=arm
$ export CROSS_COMPILE=arm-eabi-
```

Prior to compilation, I needed my device's configuration. Since I was using a real device, I tried to pull it using ADB in Android Studio:

```
$ cd ~/android-sdk/platform-tools
$ ./adb pull /proc/config.gz
```

to which I received an error message saying "config.gz" was not found. Bummer. The guide I was following mentions that some devices do not have the ability to export their configurations, so I decided to try to build my own. I went back to the Google page for [downloading and building a kernel](http://source.android.com/source/building-kernels.html).

At this point, I had a long battle of trying to clone different Git repositories which took hours to download and trying to figure out the correct configuration I needed for my Nexus 5X device. Unfortunately due to time constraints, this was my stopping point for the quarter and I don't believe I managed to successfully build a kernel. The idea was to build my own kernel, download the correct configuration, cross-compile it with LiME, root my Android device, and then load up my custom kernel onto the device so I could execute a RAM dump. Note that all of these steps are outlined in detail at the [guide](https://github.com/volatilityfoundation/volatility/wiki/Android) I started off with, but the instructions did not work as well as I had hoped.

## Summary ##
Prior to beginning this project, I did some initial research on how I would achieve my goals and didn't think that the process would end up as difficult and intricate as it was (rookie mistake). I've had experiences developing in Android Studio, so I felt pretty confident following the Developer API's to integrate different memory storage locations when developing my application. However, once I got to the steps of trying to download and create a custom kernel and commands weren't working, I got frustrataed and stuck. I feel that I am at a point where I could pick up the project pretty quickly in the future. If I had more time this quarter, I would continue poking around to either find my device's kernel configuration or build my own.

There are tools out there made by companies such as Cellibrite that are made specifically to access memory on a device and do RAM dumps of mobile phones, but when I called to request access, they quoted me a price of $15,000 for a license which isn't very feasible for a college student. Regardless, with some more time I think I would have figured something out to make more tangible progress.
