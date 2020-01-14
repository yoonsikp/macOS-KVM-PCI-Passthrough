## Troubleshooting

A reminder that Clover frequently freezes, so you should always edit the config.plist file instead.

If your macOS install is hanging on boot:
In the config.plist change
```
<key>Boot</key>
	<dict>
		<key>Arguments</key>
		<string></string>
```
to
```
<key>Boot</key>
	<dict>
		<key>Arguments</key>
		<string>-v -s</string>
```
Delete clover.raw and recreate the bootloader
`sudo ./clover-image.sh --iso Clover-v2.4k-4289-X64.iso --img clover.raw --cfg config.plist`

If your mouse/keyboard isn't working:
Change ```<controller type='usb' model='ehci'/>``` to ```<controller type='usb' model='piix4-usb-uhci'/>```

If only your mouse isn't working:
Use ```<input type='mouse' bus='usb'/>``` instead of ```<input type='tablet' bus='usb'/> ```

If you ever need to delete your VM:
Type ```sudo virsh undefine --nvram macos```. Keep in mind if you add it back, you must go through the process of changing the resolution in the UEFI again.

Speed up macOS:
```
defaults write NSGlobalDomain NSAppSleepDisabled -bool YES
sudo sysctl -w kern.timer.coalescing_enabled=0
```
