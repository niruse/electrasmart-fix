# Electra Smart for Home Assistant: fixes for the 2026 outages

Two separate things broke Electra Smart air conditioners in Home Assistant in
autumn 2026. This repository fixes the first one and documents a router-side fix
for the second.

| Symptom | Since | Cause | Fix |
|---|---|---|---|
| The integration fails to load with `KeyError: 'deviceToken'` | 2026-09-30 | Electra's cloud stopped sending `deviceToken`, and pyElectra 1.2.4 requires it | Install this custom integration until Home Assistant 2026.11.0 |
| The Electra Smart app shows **"No connection"** (אין חיבור) and Home Assistant shows the AC as unavailable, although the AC is on Wi-Fi | 2026-10-08 | The cloud hostname that older Electra Wi-Fi modules look up was deleted | Add one local DNS record on your router |

They are independent. You may have one, the other, or both.

---

## ☕ Support

If you found this project helpful, you can [buy me a coffee](https://coff.ee/niruse)!

## 🚨 Disclaimer

> This is an unofficial integration. You are responsible for the use of your credentials and your
> data. The author and contributors take no responsibility for any issues, account restrictions or
> data loss. Use at your own risk.


## Problem 1: the integration fails to load

Home Assistant's log shows:

```text
Error setting up entry ... for electrasmart
  File ".../electrasmart/device/__init__.py", line 20, in __init__
    self.token: str = data["deviceToken"]
KeyError: 'deviceToken'
```

Electra's cloud stopped including `deviceToken`, and sometimes `name`, in the
device list. pyElectra 1.2.4, which Home Assistant up to 2026.10.x pins, reads
those keys unconditionally. The token is never used afterwards.

The fix is upstream in pyElectra 1.2.5 and arrives with **Home Assistant
2026.11.0**. A backport to 2026.10.x was proposed but not merged. Until then,
this repository provides the core integration from Home Assistant 2026.9.4 with
one addition: it fills in the missing keys with empty defaults before pyElectra
sees the device. Nothing else changes, so your existing configuration entry and
entities keep working.

### Install

**HACS:** HACS → ⋮ → *Custom repositories* → add
`https://github.com/niruse/electrasmart-fix` as an *Integration* → install
*Electra Smart* → restart Home Assistant.

**Manual:** copy `custom_components/electrasmart` into
`/config/custom_components/` and restart Home Assistant.

Home Assistant will log that it found a custom integration called
`electrasmart` that it hasn't tested. That warning is expected, because this
copy intentionally overrides the built-in one.

### Remove it after upgrading

Once you are on Home Assistant **2026.11.0 or newer**, delete
`/config/custom_components/electrasmart` or uninstall it in HACS, then restart.
The built-in integration takes over with the same entities.

### What changed compared with Home Assistant 2026.9.4

- `__init__.py`: wraps `ElectraAirConditioner.__init__` so missing device
  fields get defaults instead of raising `KeyError`.
- `manifest.json`: adds `version`, `issue_tracker`, and this repository as
  documentation, which custom integrations require.

All other files are unchanged copies of `homeassistant/components/electrasmart`.

---

## Problem 2: the app says "No connection" but the AC is on Wi-Fi

### Symptoms

- The Electra Smart app shows the crossed-out cloud with **"No connection"**
  (אין חיבור).
- Home Assistant shows the AC as unavailable.
- Your router shows the AC as connected to Wi-Fi with a good signal, but it
  moves almost no data.
- Re-pairing the AC or restarting the router doesn't help.

### Cause

Older Electra Smart Wi-Fi modules find Electra's MQTT server through a hostname
on IBM's retired Watson IoT Platform:

```text
alk2da.messaging.internetofthings.ibmcloud.com
```

For years that name pointed at Electra's own server, the same one newer modules
reach as `iot.ecpiot.co.il`. On **2026-10-08** the record was deleted. The IBM
zone's SOA serial is `2026100803`, and our AC went offline that afternoon.

Since then the module asks DNS for the name every second or two, gets
"non-existent domain", and never connects. Newer modules that use
`iot.ecpiot.co.il` are not affected.

### Check whether this is your problem

```bash
nslookup alk2da.messaging.internetofthings.ibmcloud.com
```

If the answer is `Non-existent domain` (NXDOMAIN), the name is gone. If your
router can show per-device traffic, an affected AC shows a stream of DNS
lookups and no other connections.

### Fix: point the old name back at Electra's server

First look up the server's current address:

```bash
nslookup iot.ecpiot.co.il
```

On 2026-10-10 it was `62.90.149.38`. Then add a local DNS **A record** on the
DNS server your AC uses, which is normally your router:

| Name | Type | Value |
|---|---|---|
| `alk2da.messaging.internetofthings.ibmcloud.com` | A | the IP of `iot.ecpiot.co.il`, e.g. `62.90.149.38` |

Where to add it:

- **UniFi Network:** create a DNS record policy with record type *A*. It lives
  under *Policy Engine / Policy Table → Create New Policy → DNS*, and menu
  names vary by version.
- **Pi-hole:** *Local DNS Records*, add the domain and IP.
- **AdGuard Home**, including the Home Assistant add-on: *Filters → DNS
  rewrites → Add DNS rewrite*.
- **OpenWrt or dnsmasq:** add
  `address=/alk2da.messaging.internetofthings.ibmcloud.com/62.90.149.38`.
- **ISP routers without local DNS:** run Pi-hole or AdGuard Home, for example
  as a Home Assistant add-on, add the record there, and hand it out as the DNS
  server through DHCP.

No restart is needed. The module retries DNS constantly. In our case it
reconnected about a minute after the record was added, and the app and Home
Assistant came back on their own.

### Is it safe?

The record only restores what the name pointed to before October 8: Electra's
own server, which newer Electra modules already use. Nothing is intercepted or
changed. If Electra moves the server, update the IP. Remove the record once
Electra ships a firmware fix.

### How it was found

The AC had a strong Wi-Fi signal, yet the router's flow log showed hundreds of
DNS lookups every half hour and no other connections. The last successful
connection on record was TLS on port 8883 to the IBM hostname above. That
hostname now returns NXDOMAIN from every public resolver. Aleph Security's 2023
research names it as the MQTT host of older Electra controllers and shows that
it shared an IP with `iot.ecpiot.co.il`.

---

## Credits and thanks

- **[@jafar-atili](https://github.com/jafar-atili)** wrote the Home Assistant
  Electra Smart integration and the [pyElectra](https://github.com/jafar-atili/pyElectra)
  library. Almost all the code here is theirs, taken from Home Assistant core.
  Thank you!
- **[@noamokman](https://github.com/noamokman)** co-maintains the core
  integration. They landed the pyElectra 1.2.5 bump for Home Assistant
  2026.11.0 and tried to get it into a 2026.10 patch release
  ([home-assistant/core#185591](https://github.com/home-assistant/core/pull/185591)).
- **[@GuyKh](https://github.com/GuyKh)** reported the `deviceToken` breakage
  ([home-assistant/core#183846](https://github.com/home-assistant/core/issues/183846))
  and wrote the upstream fix
  ([jafar-atili/pyElectra#20](https://github.com/jafar-atili/pyElectra/pull/20)).
- **[Aleph Security](https://alephsecurity.com/2023/06/19/electra-smart/)**
  published 2023 research on the Electra Smart controller that documented its
  MQTT hostnames. That is what let us recognise the deleted IBM name.
- The **Home Assistant** contributors who maintain the integration in core.

## License

Apache License 2.0, the same as Home Assistant core. The integration is derived
from `homeassistant/components/electrasmart` at Home Assistant 2026.9.4, with
the changes listed above.

This project is not affiliated with Electra, IBM, or Home Assistant.

---

## בעברית, בקצרה

**תקלה 1: האינטגרציה לא נטענת** (`KeyError: 'deviceToken'`, מאז 30.9.2026).
הענן של אלקטרה הפסיק לשלוח שדה שהספרייה pyElectra 1.2.4 דורשת. התקינו את
הרכיב מהמאגר הזה (HACS → Custom repositories) ואתחלו את Home Assistant. אחרי
עדכון ל-2026.11.0 מחקו אותו, כי התיקון הרשמי נכנס לגרסה הזו.

**תקלה 2: באפליקציה כתוב "אין חיבור" אף שהמזגן מחובר ל-Wi-Fi** (מאז 8.10.2026).
מודולי ה-Wi-Fi הישנים של אלקטרה מוצאים את השרת דרך השם
`alk2da.messaging.internetofthings.ibmcloud.com`, והרשומה הזו נמחקה. הפתרון
הוא רשומת DNS מקומית בראוטר מסוג A שמפנה את השם הזה לכתובת של
`iot.ecpiot.co.il` (ב-10.10.2026 הכתובת הייתה `62.90.149.38`). אין צורך
לאתחל דבר. המזגן מתחבר מחדש תוך כדקה.
