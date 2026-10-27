---
name: active-directory-attacks
description: 'Use against an Active Directory domain once any domain context (a user, a host, or a recovered key) is held. Kerberoast / AS-REP roasting, DCSync / secrets extraction, ADCS certificate abuse (ESC paths), and RBCD, path-driven by a BloodHound graph.'
mcps: pa-ad, pa-bloodhound
---
Authorized targets only. Doctrine. (1) Map attack paths FIRST with pa-bloodhound (collect, then shortest-path and owned-asset queries) so each escalation follows a proven graph edge rather than noise. (2) Use pa-ad for the concrete primitives the graph justifies: Kerberoast / AS-REP for crackable service tickets, DCSync / secrets extraction once you hold the rights, ADCS certificate abuse (ESC paths) or RBCD where an edge exists. (3) Prefer the lowest-noise edge that the graph proves; hand recovered tickets/hashes to offline cracking by reference. (4) Confirm each escalation with an access oracle. MITRE T1558, T1003.006, T1649.