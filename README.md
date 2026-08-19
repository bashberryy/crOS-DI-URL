# crOS-DI-URL

chromeos oobe sign in process assets

## what's the DI?

chromeos's di (domain identifier) is what controls what a chromeos device's domain manages the device. the URL is

* https://accounts.google.com/v3/signin/identifier?client_id=77185425430.apps.googleusercontent.com&flow=deviceEnrollment&hd=type-any-domain-here&hl=en-US&use_native_navigation=1&flowName=GlifSetupChromeOs&continue=http://accounts.google.com/programmatic_auth_chromeos?hl%3Den-US%26scope%3Dhttps://www.google.com/accounts/OAuthLogin%26client_id%3D77185425430.apps.googleusercontent.com%26access_type%3Doffline&dsh=S-1374596101:1780346705514969

after "deviceEnrollment&hd=" is where the DI is set. hd stands for hosted domain, the same usage and meaning as DI.

if webview was controllable in chromeos OOBE, you would be able to change the DI to gmail.com for example, and sign in with a personal account. unfortunately, webview isn't controllable so it is unknown if this would be an exploit.

## where is the DI used?

the DI is used on login to your chromeos device. the hd will change depending on the signup flow.

## how was the URL found?

although it's likely that you could've just looked through google source code of chromeos for webview usage on OOBE to find this URL, it was found by trafficking through strict cloudflare restrictions on a proxy, leading to chromeos webview saying the site couldn't be reached, and literally saying the URL.

## url findings

these are the URLs found while tracing the chromeos OOBE google sign-in flow.

* `https://accounts.google.com/v3/signin/identifier` — **confirmed:** main Google account identifier page used by the OOBE sign-in flow. `deviceEnrollment` and `hd` appear here.
* `https://accounts.google.com/v3/signin/` — **confirmed:** Google sign-in endpoint family used by the OOBE flow.
* `https://accounts.google.com/signin/recovery` — **confirmed:** Google account recovery endpoint.
* `https://accounts.google.com/AccountChooser` — **found:** Google account chooser endpoint used by the sign-in interface.
* `https://accounts.google.com/restart` — **found / redirects:** redirects back into the Google sign-in flow.
* `https://accounts.google.com/Logout` — **found:** Google logout endpoint referenced by the sign-in process.
* `https://accounts.google.com/_/bscframe` — **found:** Google background/frame endpoint used by the account UI.
* `https://accounts.google.com/_/AccountsSignInUi` — **found:** Google Accounts sign-in UI endpoint.
* `https://accounts.youtube.com` — **found:** Google/YouTube account authentication surface encountered while tracing the account flow.
* `http://accounts.google.com/programmatic_auth_chromeos` — **confirmed:** ChromeOS-specific programmatic authentication continuation URL embedded in the main OOBE URL. it requests the OAuthLogin scope and offline access.
* `https://accounts.google.com/interstitial/doritos/forward/success` — **404 when tested:** endpoint was found during tracing, but the tested success path returned 404.
* `https://accounts.google.com/interstitial/doritos/forward/timeout` — **404 when tested:** endpoint was found during tracing, but the tested timeout path returned 404.

## READ THIS

update: `hd=*`: Forces Google to ask for an account from any managed Google Workspace domain (blocking standard @gmail.com personal accounts). you could probably still remove the hd parameter entirely.

none of the URLs above, by themselves, prove that the DI can be bypassed. the important part is that `hd` is supplied as part of the ChromeOS device-enrollment sign-in flow. changing it would require control over the OOBE webview/navigation rather than simply editing a normal browser URL.

recovery in chromeos webview: https://accounts.google.com/v3/signin/recoveryidentifier?flowName=GlifSetupChromeOs&dsh=S-368440106%3A1787169239399889
