# Back Up Android Apps and Data in Modern Times (2024)

_2024/04/15 Cecill Etheredge // ijsf

---

Imagine this. Your phone just fell on the bricks. The display is busted and barely working. What next?

Phones have become an essential part in our daily interactions with banks, social networks, entertainment, and what not. Losing or breaking a phone has never been more nervewrecking. Backing up data on Android is becoming harder, even for more advanced users.

A decade ago, there were viable options available for power users. Apps like Titanium Backup come to mind. But Android is being locked down tighter every year, because of security and compliance for trusted computing. This mindset of turning phones into walled gardens, where even a power user shall shall have no access, is becoming the norm and is trickling down to alternative Android forks as well.

The problem is that sometimes you just want to image everything on a machine, whether it is a server, a desktop, or a phone. But for phones, this goes against everything the walled garden stands for. Because the walled garden is supposed to keep you, the owner of the device, out as well.

Modern versions of LineageOS, 18.1 as of the moment of writing, do no longer provide a practical way for applications to gain root. Backup applications such as Titanium Backup no longer work. There are still other methods such as Magisk and boot image patching in order to get these backup apps to work, but these are very intrusive and may come with considerable risk.

There are backup options available in Android distributions, but many apps now downright refuse to hand out access to any of their data. The result is that these backup methods as well as the `apk backup` and `apk restore` commands no longer provide a proper _full_ way of backing up app data and are next to useless for full backup purposes.

Then there are those seemingly shady backup apps have sprung up in the last few years. Some of them need root access and all of them consist of scripts that are glued together in odd ways. Really, none of them can be trusted.

So why not back things up manually instead? This quick 'n dirty guide to get you started with a few lines of shell script!

## 1. Method

Currently, the `/data` partition is still accessible through a adb shell with root privileges, which can be gained with `adb root` by only changing some non-intrusive settings on the phone. This means that app APKs and data can in fact be backed up and restored as before, without any external tools, by using a adb-capable host machine.

Something can be said about the security implications of supplying root to apps on the phone.. that's all understandable, as long as they don't take root on our phone away from adb!

Some things to keep in mind:

* Some apps may not respond well to their data being restored on a different machine, such as Signal. These apps will typically crash immediately when started. For example, some apps may encrypt their data using a keychain on the phone which makes it very difficult to restore things.
* Use `adb logcat` to debug any issues with apps.

## 2. Apps

### 2.1. Backup

Download all app APKs from phone to current directory.

Bash:
```
for APP in $(adb shell pm list packages -3 -f)
do
  adb pull $( echo ${APP} | sed "s/^package://" | sed "s/base.apk=/base.apk /").apk
done
```

Powershell:
```
foreach ($APP in $(adb shell pm list packages -f -3)) {Invoke-Expression $($APP.replace('package:','adb pull ').replace('base.apk=','base.apk ')+'.apk')}
```

### 2.2. Restore

Upload all APKs in current directory to phone.

Bash:
```
for APP in *.apk
do
  adb install ${APP}
done
```

Batch:
```
for /f %f in ('dir /b .') do adb install %f
```

## 3. App data

Make sure adb has root access.

Host:
```
adb root
```

### 3.1. Backup

Upload `backup_data.sh` bash script.

Host:
```
adb push backup_data.sh /data/data/ && adb shell chmod 755 /data/data/backup_data.sh
```

#### List apps

Host
```
adb shell ls /data/data/
```

#### Backup

Assuming `/data/data/_backup` as location on phone for tarballs, though this can be changed.

Backup specific app:

Host
```
adb shell bash /data/data/backup_data.sh /data/data/_backup com.google.android.apps.authenticator2
```

#### Pull backups

Assuming `/data/data/_backup` as location on phone for tarballs, though this can be changed.

Host
```
adb pull /data/data/_backup
```

### 3.2. Restore

Upload `restore_data.sh` bash script.

Host:
```
adb push restore_data.sh /data/data/ && adb shell chmod 755 /data/data/restore_data.sh
```

#### Push backups

Host
```
adb push _backup /data/data/
```

#### Restore

Restore specific app:

Host
```
adb shell bash /data/data/restore_data.sh /data/data/_backup com.google.android.apps.authenticator2
```

Restore all apps from `/data/data/_backup`:

Host:
```
adb shell cd /data/data/_backup && for f in *.tar.bz2; do bash /data/data/restore_data.sh /data/data/_backup ${f%.data.tar.bz2}; done;
```

#### Cleanup

Host
```
rm -rf /data/data/_backup
```
