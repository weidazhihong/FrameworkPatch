# Framework Patch
Modify framework.jar to build a valid certificate chain.

## Requirements
- TEE must be reprogrammed with a valid keybox.
- Intermediate Windows and Linux knowledge.
- Intermediate Java and Smali knowledge.
- WSL (only in Windows).
- Java.
- 7zip.

In GNU/Linux distro, install this packages (I use Ubuntu in WSL2):
```
sudo apt update
sudo apt full-upgrade -y
sudo apt install -y default-jdk zipalign
```

## Tutorial
First, cd to a working (and clean) directory.

Pull framework.jar from your device:
```
adb pull /system/framework/framework.jar
```

Now, compile [smali](https://github.com/google/smali):
(Use WSL if you are in Windows)
```
git clone --depth=1 https://github.com/google/smali.git
cd smali
./gradlew build
```

Then pick smali and baksmali fatJars and paste to working dir.

Using 7zip extract framework.jar to framework/ directory.

Now using [jadx](https://github.com/skylot/jadx) open framework.jar and check these classes:
- android.security.keystore2.AndroidKeyStoreSpi
- android.app.Instrumentation

You must check in where .dex they are, you can know by checking upper text in class declaration, something like this:
```
/* loaded from: classes3.dex */
public class AndroidKeyStoreSpi extends KeyStoreSpi

/* loaded from: classes.dex */
public class Instrumentation 
````

Now using baksmali.jar, decompile that .dex files:
```
java -jar baksmali.jar d framework/classes3.dex -o classes3
java -jar baksmali.jar d framework/classes.dex -o classes
```

After .dex files are decompiled, you must search in folders for this files and modify like this:

- AndroidKeyStoreSpi.smali:

Search for method "engineGetCertificateChain" and near the end should be a line like this:
```
aput-object v0, v2, v3
return-object v2
```

In this example:

v0 -> leaf cert.
v2 -> certificate chain.
v3 -> 0, the position to insert the certificate in certificate chain.

It may be different in your .smali file. Do not copy and paste...

Before aput operation, you must add this:
```
invoke-static {XX}, Les/chiteroman/framework/Main;->modifyCertificate(Ljava/security/cert/Certificate;)Ljava/security/cert/Certificate;
move-result-object XX
```

Replace XX with the leaf certificate register.

So the final code (in this example) should be this:
```
invoke-static {v0}, Les/chiteroman/framework/Main;->modifyCertificate(Ljava/security/cert/Certificate;)Ljava/security/cert/Certificate;
move-result-object v0
aput-object v0, v2, v3
return-object v2
```

- Instrumentation.smali:

Search for "newApplication" methods and before the return operation, add this:
```
invoke-static {XX}, Les/chiteroman/framework/Main;->onNewApp(Landroid/content/Context;)V
```

Replace XX with the Context register.

Now compile again the files:
```
java -jar smali.jar a -a {API_LEVEL} classes3 -o framework/classes3.dex
java -jar smali.jar a -a {API_LEVEL} classes -o framework/classes.dex
```

Replace {API_LEVEL} with the Android version you are running.

Open this project in Android Studio and change EC and RSA keys, you must provide keybox private keys.
Compile as release and copy classes.dex file.

Now add a number greater than the one that already exists in the framework/.

For example, if the greatest number is classes5.dex, you must copy it as classes6.dex

Using 7zip recompile as .zip all framework/ files without compression.

After you have the framework.zip use zipalign:
```
zipalign -f -p -v -z 4 framework.zip framework.jar
```

Now move framework.jar to /system/framework, you can use Magisk module to replace it or mount /system as read-write and replace it.
