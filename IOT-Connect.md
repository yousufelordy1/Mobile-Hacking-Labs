# IoT Connect

This lab focuses on broadcast receivers, which is an Android component that listens for system-wide or app-wide messages (called broadcasts) and reacts to them even without the app being open. Apps use them to respond to things like "battery low," "SMS received," or custom messages sent by other apps. If a receiver is exported and doesn't properly validate what it receives, any other app (or an attacker with adb access) can send it a crafted broadcast to make it do something it wasn't supposed to do. This lab walks through exactly that.
<p align="center">
  <img width="237" height="191" alt="logo" src="https://github.com/user-attachments/assets/34f90f55-ae5c-4fdf-bc96-94062030fb1b" />
</p>

<p align="center">
  <a href="https://academy.mobilehackinglab.com/course/lab-iot-connect">
    <img src="https://img.shields.io/badge/View%20the%20Lab-blue?style=for-the-badge" alt="View the Lab" />
  </a>
</p>


## The App

The idea of this lab is simple: after signing up and logging in, and go to the setup screen where you can turn on some of the devices but not all of them. To control all of them at once, you need "master access." When you tap the master switch and it asks you for a PIN.

<p align="center">
  <img width="220" alt="Setup screen" src="https://github.com/user-attachments/assets/ab5ed5c0-d55a-4a2a-99f7-b50473571ee2" />
  &nbsp;&nbsp;&nbsp;&nbsp;
  <img width="220" alt="PIN prompt" src="https://github.com/user-attachments/assets/c40b25fb-ddba-408c-93a8-f94029fa14dd" />
</p>


The core idea of this lab is to bypass that PIN.

# Static Analysis

Looking at the receivers in the manifest, we can spot `MasterReceiver` — this is the one we're looking for:

```xml
<receiver
    android:name="com.mobilehackinglab.iotconnect.MasterReceiver"
    android:enabled="true"
    android:exported="true">
    <intent-filter>
        <action android:name="MASTER_ON"/>
    </intent-filter>
</receiver>
```

Notice `android:exported="true"` — this means the receiver can be triggered directly from outside the app, without going through its UI at all.

But before we trigger the broadcast, we need to look at what the receiver actually does when it fires, so we know exactly what to send. Searching for `MasterReceiver` in JADX, we find that the actual logic lives in `CommunicationManager.kt`, inside the `initialize()` function:

```java
public final BroadcastReceiver initialize(Context context) {
    ...
    public void onReceive(Context context2, Intent intent) {
        if (Intrinsics.areEqual(intent != null ? intent.getAction() : null, "MASTER_ON")) {
            int key = intent.getIntExtra("key", 0);
            if (context2 != null) {
                if (Checker.INSTANCE.check_key(key)) {
                    CommunicationManager.INSTANCE.turnOnAllDevices(context2);
                    Toast.makeText(context2, "All devices are turned on", 1).show();
                } else {
                    Toast.makeText(context2, "Wrong PIN!!", 1).show();
                }
            }
        }
    }
    ...
}
```

Breaking down the conditions inside `onReceive()`:

- `intent.getAction() == "MASTER_ON"` the broadcast's action must match `MASTER_ON`, the same action declared in the manifest's intent-filter.
- `int key = intent.getIntExtra("key", 0)` the receiver expects an integer extra called `key`, which is really just the PIN. If we don't send one, it defaults to `0`.
- `Checker.INSTANCE.check_key(key)` this is the actual PIN check. If it returns `true`, all devices turn on and we get the "All devices are turned on" toast. If `false`, we get "Wrong PIN!!".

So the real target here is `check_key()` we need to find which PIN makes it return `true`.

Guessing a random 3-digit PIN isn't realistic, you'd need way too many tries by hand, and if you already knew it, you'd just type it in through the app's UI. 

The practical approach is to try every possible PIN from `000` to `999`, so we need a loop that sends the broadcast over and over, once for each value, until one hits the right value. This technique is called brute force.

## Exploitation

Based on what we learned from static analysis, we can now brute force the receiver by sending the broadcast with every possible 3-digit PIN:

```bash
for pin in {000..999}; do
    adb shell am broadcast -a MASTER_ON --ei key $pin
done
```
Breaking down the command:

- `for pin in {000..999}; do ... done` a bash loop that iterates through every value from `000` to `999`.
- `adb shell am broadcast` sends a broadcast from the shell.
- `-a MASTER_ON` sets the broadcast's action to `MASTER_ON`, matching what `MasterReceiver` listens for.
- `--ei key $pin` passes an integer extra named `key` (the PIN), set to the current value of `$pin` in the loop.

  After running the loop, all devices are turned on:

<p align="center">
  <img width="220" alt="All devices on" src="https://github.com/user-attachments/assets/1588b5f7-0742-4c4e-b3f0-e56659e2951a" />
</p>



## Outcome

`android:exported="true"` doesn't mean a component is unprotected it means the protection logic itself becomes the attack surface. 
So when you see `exported="true"`, go check the function behind it and its conditions.

---

I hope this write-up was helpful. Feel free to reach out if you have any questions or feedback and if you found it useful, a star ⭐ on the repo is always appreciated!
