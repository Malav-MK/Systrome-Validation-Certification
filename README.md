# Systrome-Validation-Certification
## Security Advisory: Unvalidated `t` Parameter Causes Client-Side Resource Consumption in Systrome Validation & Certification

## Summary

Systrome Validation & Certification (version `<FILL IN>`) uses the client-supplied
`t` URL parameter directly as a `setInterval` delay in `qrcode_auth.html` without
validation. A value of `0`, a negative value, or a non-numeric value causes the
interval callback to fire continuously, consuming CPU in the victim's browser and
repeatedly issuing requests.

**Scope of impact (stated plainly):** this is a **client-side, self-limited**
condition. It affects only the browser session of the user who opens the crafted
link, requires that user to open the link, stops when the page or tab is closed,
and has **no effect on other users and no server-side impact**. It is published
here for completeness and vendor remediation; it may not meet every CNA's
threshold for a security vulnerability.

- **CWE:** CWE-1284 (Improper Validation of Specified Quantity in Input);
  related CWE-400 (Uncontrolled Resource Consumption)
- **CVSS v3.1:** `AV:N/AC:L/PR:N/UI:R/S:U/C:N/I:N/A:L` — 3.1 (Low)
- **Affected product:** Systrome Validation & Certification `<version>`
- **Component:** `qrcode_auth.html`, client-side timer setup

## Details

The page reads URL parameters without validation and uses `t` as a millisecond
interval:

```javascript
var param = get_param();                 // raw, undecoded URL params
var myInterval = window.setInterval(function() {
    changeQRcode();
}, param.t * 1000);                       // no validation of param.t
```

When `param.t` is `0`, negative, or non-numeric, `param.t * 1000` evaluates to
`0` or `NaN`, which `setInterval` treats as the minimum delay. The callback then
runs as fast as the browser allows. Each invocation also sets an `<img>` `src` to
`qrcode_encrypt.php?act=qrcode_generate...`, so the victim's browser issues
requests in a tight loop until the tab is closed.

## Proof of Concept

A crafted link of the form:

```
http://<host>/qrcode_auth.html?ip=1.1.1.1&t=0
```

causes the victim's browser tab to loop continuously once opened. A self-contained
demonstration harness that counts the interval firing rate (without sending
traffic to any host) is available at `<harness URL>`.

## Impact

Opening the crafted link degrades the responsiveness of the victim's own browser
tab and causes repeated requests from that browser until the tab is closed. The
condition does not affect other users and does not cause server-side denial of
service on its own.

## Remediation

Validate the `t` parameter before use: parse as an integer, apply a sensible
floor and ceiling, and fall back to a default when invalid. For example:

```javascript
var t = parseInt(param.t, 10);
if (!isFinite(t) || t < 10) t = 30;   // floor + default
t = Math.min(t, 300);                 // ceiling
```

Additionally, rate-limit `qrcode_generate` server-side.

## Timeline

- `05/10/2026` — Public disclosure

## Credit

Discovered by [Nishi Mehta](https://www.linkedin.com/in/nishi-mehta-812194248/)
