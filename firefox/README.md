# Firefox

Here are the extensions I use on my Firefox setup:

[uBlock Origin](https://addons.mozilla.org/fr/firefox/addon/ublock-origin/)  
[SponsorBlock](https://addons.mozilla.org/fr/firefox/addon/sponsorblock/)  
[Bitwarden](https://addons.mozilla.org/fr/firefox/addon/bitwarden-password-manager/)  
[Dark Reader](https://addons.mozilla.org/fr/firefox/addon/darkreader/)  
[Netflix 1080p](https://addons.mozilla.org/fr/firefox/addon/netflix-1080p-firefox/)  
[Tree Style Tab](https://addons.mozilla.org/fr/firefox/addon/tree-style-tab/)  
[Video Ad-Block, for Twitch](https://addons.mozilla.org/en-US/firefox/addon/video-ad-block-for-twitch/)  
[Profile Switcher for Firefox](https://addons.mozilla.org/fr/firefox/addon/profile-switcher/)  
[Return Youtube Dislike](https://addons.mozilla.org/en-US/firefox/addon/return-youtube-dislikes/)

## Custom Firefox Sync server
Access:
```
about:config
```

Then change:
```
identity.sync.tokenserver.uri
```
to whatever your server is.

## Increase Firefox refresh rate to 144hz
Access:
```
about:config
```

Then change:
```
layout.frame_rate
```
to `144`.

## Force-enable WebRender
Access:
```
about:config
```

Then change:
```
gfx.webrender.all
```
to `true`.

## Widevine DRM crash with Firejail
Simple fix:
```
sed -i 's/# browser-allow-drm no/browser-allow-drm yes/g' /etc/firejail/firejail.config
```

## FIDO keys does not work with Firejail
Simple fix:
```
sed -i 's/# browser-disable-u2f yes/browser-disable-u2f no/g' /etc/firejail/firejail.config
```

## Dark Reader
Brightness -> -15  
Contrast -> off  
Sepia -> +10  
Grayscale -> off

## Profile Switcher
To be able to use this extension properly, you need to install an AUR package:
```
yay -S firefox-profile-switcher-connector
```