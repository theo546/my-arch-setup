# ungoogled-chromium

In this page, I will help you to properly setup the Flatpak version of ungoogled-chromium.

## 1. Add Google as your default search engine

Since Google is not available in the browser default search engine list, you will need to add it using these steps:

Hamburger menu -> Settings -> Search engine -> Manage search engines -> Add
- Search engine: `Google`
- Keyword: `google`
- URL with %s in place of query: `https://www.google.com/search?q=%s`
- Suggestions URL with %s in place of query: `https://suggestqueries.google.com/complete/search?&client=chrome&q=%s`

## 2. Prevent cookies to get cleared every time you close Chromium

Hamburger menu -> Settings -> Pricacy and security -> Cookies and other site data -> Untick `Clear cookies and site data when you quit Chromium`

## 3. Apply the dark theme on the browser and remove the title bar

To apply the GTK theme that we applied on the main tutorial:  
- Hamburger menu -> Settings -> Appearance -> Use GTK+

To remove the title bar:  
- Hamburger menu -> Settings -> Appearance -> Use system title bar and borders

To apply a global dark theme to all websites and Chromium internal pages, switch the flag
```
chrome://flags/#enable-force-dark
```
to `Enabled`

## 4. Allow extensions to be installed

To first allow the installation of an extension called "chromium-web-store", you will need to switch the flag
```
chrome://flags/#extension-mime-request-handling
```
to `Always prompt for install` then restart your browser.

Now access `chrome://extensions/`, activate the `Developer mode` on the top right.

Download the latest version of chromium-web-store from [GitHub](https://github.com/NeverDecaf/chromium-web-store/releases).  
To install the previously downloaded file, drag it on the Chromium window, it'll prompt you to accept the extension permissions, simply click on `Add extension`.

You may now install any extension, I can recommend you to install:
- [uBlock Origin](https://chrome.google.com/webstore/detail/ublock-origin/cjpalhdlnbpafiamejdnhcphjbkeiagm)
- [This dark theme for Chrome](https://chrome.google.com/webstore/detail/dark-theme-for-google-chr/annfbnbieaamhaimclajlajpijgkdblo)
- [This darker theme for Chrome](https://chrome.google.com/webstore/detail/dark-material-black-theme/oifnkbmbefokhdgjfklofplpoglekipa)
- [netflix-1080p](https://github.com/jangxx/netflix-1080p/releases) (drag 'n drop)

## 5. Install the Widevine DRM

```
$ wget -O - https://raw.githubusercontent.com/flathub/com.github.Eloston.UngoogledChromium/beta/widevine-install.sh | bash
```
Then restart the browser.

## Bonus

To fix YouTube white bar with forced dark mode, add this uBlock filter ([source](https://www.reddit.com/r/uBlockOrigin/comments/l1a8pm/tip_using_ublock_to_get_rid_of_youtube_white/)):
```
youtube-nocookie.com,youtube.com##.ytp-gradient-bottom,.ytp-gradient-top
```