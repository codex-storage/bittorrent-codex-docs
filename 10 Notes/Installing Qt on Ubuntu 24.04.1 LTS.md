---
tags:
  - qt/ubuntu
---
#qt/ubuntu 

The following guide was also helpful: [https://web.stanford.edu/dept/cs_edu/resources/qt/install-linux](https://web.stanford.edu/dept/cs_edu/resources/qt/install-linux).

If you previously installed:

```bash
sudo apt install --no-install-recommends qtbase5-dev qttools5-dev libqt5svg5-dev
```

Uninstall it with:

```bash
sudo apt remove qtbase5-dev qttools5-dev libqt5svg5-dev
```
### The actual steps to install Qt

1. [Download Qt](https://www.qt.io/download-qt-installer)
2. As indicated in the tutorial above, in order to install Qt your machine, you need to create account.
3. Your installer will have name like this: `qt-online-installer-linux-x64-4.8.1.run`. Make it executable (`chmod 755 qt-online-installer-linux-x64-4.8.1.run`) and run it.
4. In the installer provide your Qt credentials.
5. Select the installation option *Qt 6.8 for desktop development* below:

![[install-qt-ubuntu.png]]

6. It is possible that the installer will ask you to install some additional packages:

```bash
sudo apt install libxcb-cursor0 libxcb-cursor-dev
```

7. That's should be it. Qt should be installed and ready for use.