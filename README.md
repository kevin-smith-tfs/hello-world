fetch dt.entity.network:device
| fieldsAdd dash1 = indexOf(entity.name, "-")
| fieldsAdd prefix = substring(entity.name, from: 0, to: dash1)
| fieldsAdd remainder = substring(entity.name, from: dash1 + 1)
| fieldsAdd dash2 = indexOf(remainder, "-")
| fieldsAdd site_id = concat(prefix, "-", substring(remainder, from: 0, to: dash2))
| filter isNotNull(site_id) and site_id != entity.name
| summarize deviceCount = count(), by: {site_id}
| fieldsAdd expectedDevices = 2
| fieldsAdd coveragePct = toDouble(deviceCount) / toDouble(expectedDevices)
| lookup [
    load "/lookups/sites/site_master"
  ], sourceField: site_id, lookupField: site_id
| filter isNotNull(lookup.latitude)
| fieldsAdd category =
    if(deviceCount == 0, "Critical",
      else: if(coveragePct < 0.8, "Warning",
        else: "Good"))
| fields lookup.site_name, lookup.latitude, lookup.longitude,
         lookup.region, lookup.owner, site_id,
         category, deviceCount, coveragePct




fetch events
| filter status == "OPEN"
| fieldsAdd dash1 = indexOf(toString(tags), "site:")
| filter isNotNull(dash1)
| fieldsAdd raw = substring(toString(tags), from: dash1 + 5)
| fieldsAdd dash2 = indexOf(raw, ",")
| fieldsAdd site_id = if(dash2 > 0, substring(raw, from: 0, to: dash2), else: raw)
| summarize alertCount = count(), by: {site_id}
| lookup [
    load "/lookups/sites/site_master"
  ], sourceField: site_id, lookupField: site_id
| filter isNotNull(lookup.latitude)
| fieldsAdd category =
    if(alertCount > 5, "Critical",
      else: if(alertCount > 0, "Warning",
        else: "Good"))
| fields lookup.site_name, lookup.latitude, lookup.longitude,
         lookup.region, lookup.owner, site_id,
         category, alertCount




fetch dt.entity.network:device
| fieldsAdd dash1 = indexOf(entity.name, "-")
| fieldsAdd prefix = substring(entity.name, from: 0, to: dash1)
| fieldsAdd remainder = substring(entity.name, from: dash1 + 1)
| fieldsAdd dash2 = indexOf(remainder, "-")
| fieldsAdd site_id = concat(prefix, "-", substring(remainder, from: 0, to: dash2))
| filter isNotNull(site_id) and site_id != entity.name
| summarize deviceCount = count(), by: {site_id}
| fieldsAdd expectedDevices = 2
| fieldsAdd coveragePct = toDouble(deviceCount) / toDouble(expectedDevices)
| append [
    fetch events
    | filter status == "OPEN"
    | fieldsAdd dash1 = indexOf(toString(tags), "site:")
    | filter isNotNull(dash1)
    | fieldsAdd raw = substring(toString(tags), from: dash1 + 5)
    | fieldsAdd dash2 = indexOf(raw, ",")
    | fieldsAdd site_id = if(dash2 > 0, substring(raw, from: 0, to: dash2), else: raw)
    | summarize alertCount = count(), by: {site_id}
  ]
| summarize deviceCount = sum(deviceCount),
            alertCount = sum(alertCount),
            coveragePct = avg(coveragePct),
            by: {site_id}
| lookup [
    load "/lookups/sites/site_master"
  ], sourceField: site_id, lookupField: site_id
| filter isNotNull(lookup.latitude)
| fieldsAdd hasAlert = if(alertCount > 0, 1, else: 0)
| fieldsAdd category =
    if(hasAlert == 1, "Critical",
      else: if(deviceCount == 0, "Critical",
        else: if(coveragePct < 0.8, "Warning",
          else: "Good")))
| fields lookup.site_name, lookup.latitude, lookup.longitude,
         lookup.region, lookup.owner, site_id,
         category, deviceCount, alertCount, coveragePct
