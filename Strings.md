# Strings
This lab focuses on capturing a flag hidden inside the app's memory by looking into the exported activities, as they told us as a hint: look into the exported activities. So our mission is to find a way to trigger that activity because once it's triggered, the flag gets loaded into the app's memory. From there, we'll need to search into the app's memory to find it, and as another hint, the instructions told us that the flag follows this format: `MHL{...}`, so once we trigger the activity, we know what format to search for.

<p align="center">
  <img width="237" height="191" alt="logo" src="https://github.com/user-attachments/assets/c9856e49-54fb-46e7-a3d4-352950230ef7" />
</p>

<p align="center">
  <a href="https://academy.mobilehackinglab.com/course/lab-strings">
    <img src="https://img.shields.io/badge/View%20the%20Lab-blue?style=for-the-badge" alt="View the Lab" />
  </a>
</p>


## Static Analysis

Looking at the activities in the manifest, we can spot `Activity2` this is the one we're looking for:

```xml
<activity
    android:name="com.mobilehackinglab.challenge.Activity2"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.VIEW"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <category android:name="android.intent.category.BROWSABLE"/>
        <data
            android:scheme="mhl"
            android:host="labs"/>
    </intent-filter>
</activity>
```

`android:exported="true"` means this activity can be triggered from outside the app without going through its UI at all. It's also set up as a **deep link**: the `<data>` tag tells us it responds to URIs starting with `mhl://labs`.

But be careful `exported="true"` doesn't always mean you can just run a command and be done. Sometimes it works right away, but most of the time you'll need to look into the function behind it to understand its conditions before you know exactly what to send.

This takes us to `Activity2`'s own code, specifically the `onCreate()` function:

```java
public final class Activity2 extends AppCompatActivity {
    private final native String getflag();

    /* JADX INFO: Access modifiers changed from: protected */
    @Override // androidx.fragment.app.FragmentActivity, androidx.activity.ComponentActivity, androidx.core.app.ComponentActivity, android.app.Activity
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_2);
        SharedPreferences sharedPreferences = getSharedPreferences("DAD4", 0);
        String u_1 = sharedPreferences.getString("UUU0133", null);
        boolean isActionView = Intrinsics.areEqual(getIntent().getAction(), "android.intent.action.VIEW");
        boolean isU1Matching = Intrinsics.areEqual(u_1, cd());
        if (isActionView && isU1Matching) {
            Uri uri = getIntent().getData();
            if (uri != null && Intrinsics.areEqual(uri.getScheme(), "mhl") && Intrinsics.areEqual(uri.getHost(), "labs")) {
                String base64Value = uri.getLastPathSegment();
                byte[] decodedValue = Base64.decode(base64Value, 0);
                if (decodedValue != null) {
                    String ds = new String(decodedValue, Charsets.UTF_8);
                    byte[] bytes = "your_secret_key_1234567890123456".getBytes(Charsets.UTF_8);
                    Intrinsics.checkNotNullExpressionValue(bytes, "this as java.lang.String).getBytes(charset)");
                    String str = decrypt("AES/CBC/PKCS5Padding", "bqGrDKdQ8zo26HflRsGvVA==", new SecretKeySpec(bytes, "AES"));
                    if (str.equals(ds)) {
                        System.loadLibrary("flag");
                        String s = getflag();
                        Toast.makeText(getApplicationContext(), s, 1).show();
                        return;
                    } else {
                        finishAffinity();
                        finish();
                        System.exit(0);
                        return;
                    }
                }
                finishAffinity();
                finish();
                System.exit(0);
                return;
            }
            finishAffinity();
            finish();
            System.exit(0);
            return;
        }
        finishAffinity();
        finish();
        System.exit(0);
    }

    public final String decrypt(String algorithm, String cipherText, SecretKeySpec key) {
        Intrinsics.checkNotNullParameter(algorithm, "algorithm");
        Intrinsics.checkNotNullParameter(cipherText, "cipherText");
        Intrinsics.checkNotNullParameter(key, "key");
        Cipher cipher = Cipher.getInstance(algorithm);
        try {
            byte[] bytes = Activity2Kt.fixedIV.getBytes(Charsets.UTF_8);
            Intrinsics.checkNotNullExpressionValue(bytes, "this as java.lang.String).getBytes(charset)");
            IvParameterSpec ivSpec = new IvParameterSpec(bytes);
            cipher.init(2, key, ivSpec);
            byte[] decodedCipherText = Base64.decode(cipherText, 0);
            byte[] decrypted = cipher.doFinal(decodedCipherText);
            Intrinsics.checkNotNull(decrypted);
            return new String(decrypted, Charsets.UTF_8);
        } catch (Exception e) {
            throw new RuntimeException("Decryption failed", e);
        }
    }

    private final String cd() {
        String str;
        SimpleDateFormat sdf = new SimpleDateFormat("dd/MM/yyyy", Locale.getDefault());
        String format = sdf.format(new Date());
        Intrinsics.checkNotNullExpressionValue(format, "format(...)");
        Activity2Kt.cu_d = format;
        str = Activity2Kt.cu_d;
        if (str != null) {
            return str;
        }
        Intrinsics.throwUninitializedPropertyAccessException("cu_d");
        return null;
    }
}
```

It looks like a lot at first, but really it's just three conditions stacked on top of each other all of them need to be true before the flag shows up.


Breaking down the three conditions:

### Condition 1

```java
if (isActionView && isU1Matching)
```

Both `isActionView` and `isU1Matching` need to be `true` for this condition to pass. Let's break each one down.

```java
boolean isActionView = Intrinsics.areEqual(getIntent().getAction(), "android.intent.action.VIEW");
```

This means the intent's action must be `VIEW`.

```java
boolean isU1Matching = Intrinsics.areEqual(u_1, cd());
```

For this to be `true`, whatever `u_1` holds must match what the `cd()` function returns.

Now let's take `u_1`:

```java
String u_1 = sharedPreferences.getString("UUU0133", null);
```

This line reads a value from shared preferences:

```java
SharedPreferences sharedPreferences = getSharedPreferences("DAD4", 0);
```

Now let's take the `cd()` function:

```java
private final String cd() {
    String str;
    SimpleDateFormat sdf = new SimpleDateFormat("dd/MM/yyyy", Locale.getDefault());
    String format = sdf.format(new Date());
    Intrinsics.checkNotNullExpressionValue(format, "format(...)");
    Activity2Kt.cu_d = format;
    str = Activity2Kt.cu_d;
    if (str != null) {
        return str;
    }
    Intrinsics.throwUninitializedPropertyAccessException("cu_d");
    return null;
}
```

This function simply returns today's date, formatted as `dd/MM/yyyy`.

### Condition 2

```java
if (uri != null && Intrinsics.areEqual(uri.getScheme(), "mhl") && Intrinsics.areEqual(uri.getHost(), "labs"))
```

This condition checks the URI we send, three things need to be true:

- `uri != null` the intent must actually carry a URI (data). Once we have a valid URI, the code pulls out its last path segment and tries to Base64-decode it:

```java
  String base64Value = uri.getLastPathSegment();
  byte[] decodedValue = Base64.decode(base64Value, 0);
  if (decodedValue != null) {
```

  This just checks that the decoding actually worked if `base64Value` wasn't valid Base64, `decodedValue` would come back `null` and this check would fail.
- `uri.getScheme() == "mhl"` the URI's scheme must be `mhl`.
- `uri.getHost() == "labs"` the URI's host must be `labs`.

Put together, this means the URI we send has to look like `mhl://labs/...`.

### Condition 3

```java
if (str.equals(ds))
```

Let's break down `ds` and `str`.

```java
String ds = new String(decodedValue, Charsets.UTF_8);
```

`ds` is simply the Base64-decoded value we sent in the URI, converted into a readable string.

```java
byte[] bytes = "your_secret_key_1234567890123456".getBytes(Charsets.UTF_8);
String str = decrypt("AES/CBC/PKCS5Padding", "bqGrDKdQ8zo26HflRsGvVA==", new SecretKeySpec(bytes, "AES"));
```

`str` is the result of decrypting a hardcoded AES ciphertext (`bqGrDKdQ8zo26HflRsGvVA==`), using a hardcoded key (`your_secret_key_1234567890123456`). To actually decrypt AES/CBC, we also need an IV. If we look back at how `str` is built, that leads us to `Activity2Kt`, which stores it:

```java
public final class Activity2Kt {
    private static String cu_d = null;
    public static final String fixedIV = "1234567890123456";
}
```

So now we have everything we need to decrypt this value ourselves, offline:

- **Algorithm:** `AES/CBC/PKCS5Padding`
- **Key:** `your_secret_key_1234567890123456`
- **IV:** `1234567890123456`
- **Ciphertext:** `bqGrDKdQ8zo26HflRsGvVA==`

Since all four pieces are sitting right there in the code, we can run the same decryption ourselves and get the exact value the app expects `ds` to equal.


We can decrypt this locally with a quick Python script:

```python
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad
import base64

key = "your_secret_key_1234567890123456".encode('utf-8')
iv = "1234567890123456".encode('utf-8')
ciphertext_b64 = "bqGrDKdQ8zo26HflRsGvVA=="

ct = base64.b64decode(ciphertext_b64)
cipher = AES.new(key, AES.MODE_CBC, iv)
decrypted = cipher.decrypt(ct)
plain = unpad(decrypted, 16)
print(plain.decode('utf-8'))
```

Running this gives us the plaintext: `mhl_secret_1337`

Now, since `ds` is expected to equal this value, and `ds` comes from Base64-decoding the value we put in the URI, we need to Base64-**encode** `mhl_secret_1337` ourselves — so that when the app decodes it back, it gets the right string.

```python
import base64
print(base64.b64encode(b"mhl_secret_1337").decode())
```

This gives us: `bWhsX3NlY3JldF8xMzM3`  this is the value we'll put at the end of our deep link URI.



## Exploitation

Now that we know what all three conditions need, let's put it together.

**Step 1 Satisfy Condition 1 (the date check)**

Since nothing in the app ever writes today's date into shared preferences on its own, we need to write it ourselves. First, we create the file locally:

```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <string name="UUU0133">16/08/2026</string> <!-- your current date -->
</map>
```

Then we push it to the device:

```bash
adb push DAD4.xml /data/local/tmp/DAD4.xml
```

`adb push` can't write directly into the app's private storage, so we push it into a temp folder first. Then, with root access, we move it into place and fix its ownership:

```bash
adb shell
su
cp /data/local/tmp/DAD4.xml /data/data/com.mobilehackinglab.challenge/shared_prefs/DAD4.xml
chown u0_a339:u0_a339 /data/data/com.mobilehackinglab.challenge/shared_prefs/DAD4.xml
chmod 660 /data/data/com.mobilehackinglab.challenge/shared_prefs/DAD4.xml
```

`chown` sets the file's owner to match the app's own user (`u0_a339`), since the file was created by root and the app itself needs to be able to read it. `chmod 660` gives read/write access to the owner and group only, keeping it consistent with how the app's other private files are permissioned.

**Step 2 Satisfy Conditions 2 and 3 (the deep link)**

We fire the deep link with the correct scheme, host, and our Base64-encoded secret from earlier:

```bash
adb shell am start -W -a android.intent.action.VIEW -d "mhl://labs/bWhsX3NlY3JldF8xMzM3" com.mobilehackinglab.challenge
```

All three conditions now pass, and the app calls `getflag()` from its native library.

**Step 3 The decoy**

The Toast that appears just says **"Success"** not the actual flag. That's intentional: the native library computes the real flag with obfuscated math and leaves it sitting in memory, but always returns the literal string `"Success"` instead.

**Step 4 Search memory for the flag**

Since we know the flag follows the format `MHL{...}`, we can search the live process's memory for its hex signature (`4d 48 4c 7b` = `MHL{`), using Frida via objection:

```bash
objection -g com.mobilehackinglab.challenge explore
```
<p align="center">
  <img width="600" alt="Memory search result showing the flag" src="https://github.com/user-attachments/assets/ae2e61de-7976-471c-974e-9b1145386238" />
</p>

## Outcome

This lab showed that one exported activity can have several checks stacked on top of each other a shared preferences check, a deep link check, and a decryption check, all before anything happens. And even after passing every check, the real flag wasn't given to us directly it was hidden in the app's memory.

---

I hope this write-up was helpful. Feel free to reach out if you have any questions or feedback and if you found it useful, a star ⭐ on the repo is always appreciated!
