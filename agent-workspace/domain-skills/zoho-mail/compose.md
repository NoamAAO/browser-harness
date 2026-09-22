# Zoho Mail: compose and send with attachments

Web client at `https://mail.zoho.com/zm/` (`.eu` for EU-datacenter orgs). Field-tested Sep 2026.

## Accounts and sessions

- Chrome may hold a session for a *different* Zoho account. The mailbox title tells you which:
  `Inbox - Zoho Mail (user@domain)`. Check `page_info()["title"]` before composing.
- The profile panel (top-right avatar) has no account switcher. A second account is added by the
  user at `https://accounts.zoho.com/signin`; both sessions then coexist. Auth wall = stop and ask.
- After the user logs in, pick the right tab by title, not by `ensure_real_tab()` (that attaches to
  the user's active tab, which may be unrelated):

```python
tabs=[t for t in list_tabs(include_chrome=False) if "mail.zoho.com" in t["url"] and "user@domain" in t["title"]]
switch_tab(tabs[0]["targetId"]); cdp("Target.activateTarget", targetId=tabs[0]["targetId"])
```

## Compose fields (stable, by placeholder / aria-label)

- New Mail button: element whose trimmed text is `New Mail`.
- To: `input[placeholder="To Recipients"]`, click, `type_text(addr)`, `press_key("Enter")` commits the chip.
  Chip shows only the local part; the full address is on the ancestor `div.zmCB[aria-label="<email>"]`.
- Cc: `input[placeholder="CC Recipients"]`. Subject: `input[placeholder="Subject"]`.
- Body: same-origin iframe with `title="Text editor area"`. Set HTML directly, then fire `input`:

```python
js("""(()=>{const f=[...document.querySelectorAll("iframe")].find(i=>(i.getAttribute("title")||"").includes("Text editor"));
const b=f.contentDocument.body; b.innerHTML=HTML + b.innerHTML; b.dispatchEvent(new Event("input",{bubbles:true}));})()""")
```

  Keep the existing `innerHTML` (signature lives there). `<div>` per paragraph, `<div><br></div>` as spacer.
- Send: element whose trimmed text is `Send`. Success = compose tab closes, title flips back to
  `Inbox - ...`, a `Mail sent` toast shows for a few seconds. Verify in Sent: rows carry
  `aria-label="From Me, Subject ..., Delivered Time ..., N attachments"`. That N is unreliable (said 1 for a
  2-attachment mail); open the row and count `2 Attachments` text if it matters.

## Attachments: intercept the file chooser

There is no `input[type=file]` in the DOM, not even after opening the attach dialog. The toolbar button
`button[title="Attachment"]` opens Zoho's own dialog (Desktop / My Attachments / Zoho docs ...), whose
`Upload files` button triggers the native picker. Intercept it on the *explicit page session*:

```python
from browser_harness.helpers import _send
sid = _send({"meta":"session"})["session_id"]
cdp("Page.enable", session_id=sid)
cdp("Page.setInterceptFileChooserDialog", session_id=sid, enabled=True); drain_events()
click_at_xy(*attach_btn_xy); wait(2)          # opens Zoho dialog
click_at_xy(*upload_files_xy)                 # "Upload files" inside the dialog
ev=[]
for _ in range(8):
    wait(0.5); ev += [e for e in drain_events() if e.get("method")=="Page.fileChooserOpened"]
    if ev: break
cdp("DOM.setFileInputFiles", session_id=sid, files=[abs_paths...], backendNodeId=ev[0]["params"]["backendNodeId"])
cdp("Page.setInterceptFileChooserDialog", session_id=sid, enabled=False)
wait(5)                                        # files list in the dialog
click_at_xy(*dialog_attach_btn_xy)             # blue "Attach" bottom-left of the dialog
```

Then poll until the compose shows `2 Attachments` and no `Scanning Virus` text. Filenames render split
across nodes (`ticket-print` + `.pdf`), so match on the stem.

Without `session_id=sid` the intercept silently does nothing and the dialog just closes.

## Traps

- Do not push multi-MB payloads through `js()`: the daemon IPC dies with `BrokenPipeError` around a few MB.
  Building a `File` from base64 in-page is therefore out for real attachments.
- Page CSP blocks `fetch()` to `http://127.0.0.1:*`, so a local file server is no workaround either.
- Synthetic `DragEvent("drop")` with a `DataTransfer` did not attach anything.
- `Page.fileChooserOpened` never fires for the first `Attachment` click, only for `Upload files`.
- `mail.zoho.eu` may throw a certificate error from some networks; `.com` redirects logged-in EU users correctly.
