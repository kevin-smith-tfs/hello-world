Great question — here’s a **clean, minimal helper** that fits *exactly* into what you showed in `SNMPDevice`, without touching the async scanner or the transport logic.

This is written so you can literally drop it into `models.py` inside the `SNMPDevice` class.

---

## 1) Add this helper inside `SNMPDevice`

```python
async def resolve_credentials(self, cred_list):
    """
    Try each SNMP credential until one works.
    Caches the first valid credential on the device.
    """
    if getattr(self, "_cred_resolved", False):
        return self.credentials

    for cred in cred_list:
        try:
            ok = await self._test_credential(cred)
            if ok:
                self.credentials = cred
                self._cred_resolved = True
                return cred
        except Exception:
            continue

    raise RuntimeError(f"No valid SNMP credentials for {self.address}")
```

---

## 2) Add this tiny test method

This uses a **single lightweight OID** (no walks, no tables):

```python
async def _test_credential(self, cred):
    """
    Quick SNMP probe to validate a credential.
    """
    from pysnmp.hlapi.asyncio import getCmd, ObjectType, ObjectIdentity

    errorIndication, errorStatus, _, varBinds = await getCmd(
        SnmpEngine(),
        cred,
        await self.target,
        ContextData(),
        ObjectType(ObjectIdentity("SNMPv2-MIB", "sysName", 0)),
    )

    if errorIndication or errorStatus:
        return False

    return True
```

---

## 3) Wire it in **right above** this line (the one you showed):

```python
self.credentials = UsmUserData(...)
```

### Replace with:

```python
# line above this already exists in your file
if not getattr(self, "_cred_resolved", False):
    await self.resolve_credentials(self.activation.credentials)

self.credentials = self.credentials
```

This keeps your existing flow unchanged.

---

## 4) What this gives you

* One SNMP probe per device
* Automatic v2/v3 fallback
* No scanner changes
* No extra polling
* No AG burn

Once resolved, every future call uses the working credential.

---

If you paste the surrounding block from `models.py`, I can show you **exactly** where to drop it line-for-line.
