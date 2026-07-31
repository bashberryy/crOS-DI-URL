# crOS-DI-URL
chromeos oobe sign in process assets

## what's the DI?
chromeos's di (domain identifier) is what controls what a chromeos device's domain manages the device.
the URL is 
- https://accounts.google.com/v3/signin/identifier?client_id=77185425430.apps.googleusercontent.com&flow=deviceEnrollment&hd=type-any-domain-here&hl=en-US&use_native_navigation=1&flowName=GlifSetupChromeOs&continue=http://accounts.google.com/programmatic_auth_chromeos?hl%3Den-US%26scope%3Dhttps://www.google.com/accounts/OAuthLogin%26client_id%3D77185425430.apps.googleusercontent.com%26access_type%3Doffline&dsh=S-1374596101:1780346705514969

after "deviceEnrollement&hd=" is where the DI is set. hd stands for hosted domain, the same usage and meaning as DI.

if webview was controllable in chromeos OOBE, you would be able to change the DI to gmail.com for example, and sign in with a personal account. unfortunately, webview isn't controllable so it is unknown if this would be an exploit.

## where if the DI used?
the DI is used on login to your chromeos device. the hd will change depending on the signup flow.

## how was the URL found?
although it's likely that you could've just looked through google source code of chromeos for webview usage on OOBE to find this URL, it was found by trafficking through strict cloudflare restrictions on a proxy, leading to chromeos webview saying the site couldn't be reached, and literally saying the URL. 

## READ THIS
update: `` hd=* ``: Forces Google to ask for an account from any managed Google Workspace domain (blocking standard @gmail.com personal accounts).
you could probably still remove the hd parameter entirely.
