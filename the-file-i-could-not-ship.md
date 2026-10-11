# The file I could not ship

*From my [SIEM home lab](https://github.com/exosphere8/siem-home-lab). Turning a hobby
project into something I can sell, and the 2,362-line file that almost made that a problem.*

---

## The setup

My toolkit, `siemlab`, checks Wazuh 5 rules against the Wazuh Common Schema: Wazuh 5 rejects
a rule that names a field outside it. I had copied the official field list into the package,
so the check worked offline. It was a plain list of field names, generated from a CSV in one
of Wazuh's repositories.

Then I decided I want to sell `siemlab`.

## The assumption

"It's just a list of field names. Facts aren't copyrightable, and everyone uses these names."

## What changed my mind

Before packaging, I checked the license of every repository I had taken anything from. Most
were what I expected. The schema came from `wazuh-indexer-plugins`, which is licensed under
the **AGPL-3.0**.

I'm not a lawyer, and a bare list of names may well not be protected at all. But "may well
not be" is a poor foundation for a product. If I'm wrong, the cost lands on the people who
paid me. Shipping a file derived from AGPL code inside a commercial package is exactly the
kind of question I don't want a customer to have to ask.

## What I did

- **Stopped bundling the file.** `siemlab wazuh5 fetch-schema` downloads the list from the
  official repository at a pinned release, checks it against a pinned SHA-256, and caches
  it on the user's machine. Without it, the field check is skipped with a clear warning.
- **Kept the tests self-contained** with a 16-line list of the field names my own rules use.
- **Wrote down every third-party dependency** in a notices file. MITRE ATT&CK allows
  commercial use but requires an attribution line. pySigma (LGPL) is an optional dependency,
  never bundled. "Wazuh" is a trademark, so the product only uses it to say what it works with.
- **Changed my own license** from 0.5.0: free for non-commercial use, with a commercial
  license for everything else. Versions up to 0.4.0 stay MIT, because a license that's
  already been granted can't be taken back.

## What I'd take from this

- **License checks belong before the first release, not before the first sale.** Every file
  you ship is a promise about what the buyer is allowed to do with it.
- **"Small" is not a license.** The size of a file says nothing about the terms it comes with.
- **Make the safe path the easy path.** One command to fetch the schema costs the user ten
  seconds and removes the question entirely.
