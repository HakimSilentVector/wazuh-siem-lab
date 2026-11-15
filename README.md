# Wazuh SIEM Home Lab

A self-built detection and response lab running Wazuh. I built it to learn SOC analysis and detection engineering the only way that sticks: deploy the sensor, run real attacker activity against my own endpoints, write and tune the detections, and document what actually fired and what didn't.

This repo is the index. Each phase has its own writeup with the rule, the evidence, and the finding. The point was never to collect green checkmarks. It was to prove I can stand up monitoring, catch a technique, explain why the detection works, and be honest about the gaps I found along the way.

## Architecture

- Wazuh all-in-one manager on a repurposed 2015 MacBook Air running Ubuntu Server.
- A macOS laptop enrolled as an agent, used as a victim endpoint.
- A Windows 11 laptop enrolled as a second agent, used as a victim endpoint.
- A Tailscale mesh as the management plane, so the dashboard and SSH stay reachable without exposing anything to the public internet.
- Attacks run against my own enrolled endpoints (purple team). Detections get built on the manager.

## Phases

| Phase | Focus | MITRE ATT&CK | Headline finding |
|------|-------|--------------|------------------|
| 0 to 2 | Server build, zero-trust management plane, agent enrollment | n/a | A reachable, hardened monitoring plane with no public exposure |
| 3 | SSH brute-force detection | T1110 | Fixed auth log collection, ran a real brute-force, watched Wazuh rules 5710 and 5712 fire, backed it with a custom Python detector |
| 4 | File Integrity Monitoring | T1565.001 | macOS syscheck does not surface file-modification alerts in this setup. Proven across all three scan paths, not assumed. |
| 5 | LaunchAgent persistence | T1543.001 | macOS syscheck does detect file creation (rule 554) via scheduled scan. Creation and modification are different code paths. |
| 6 | Custom detection rules | T1543.001 | Took the generic syscheck event and wrote a custom rule in the local range that enriches it into a prioritized, ATT&CK-mapped persistence detection |
| 7 | Active response | n/a | Automated response chained off the custom rule, with a hard safety rule: nothing destructive binds to a broad trigger |
| 8 | Remediation | n/a | The destructive action is gated on a narrow, high-confidence trigger built before the response, never after |
| 9 | Windows agent enrollment | n/a | A second operating system in the detection path |
| 10 | Windows FIM | T1565.001 | Windows FIM surfaces both creation and modification, the direct contrast to the macOS modification miss from Phase 4 |
| 11 | Who-data attribution | T1070 | Upgraded Windows FIM to who-data, so the modification alert now carries the actor: the user, the process, and the PID |

## Headline findings

**The macOS create-versus-modify boundary (Phases 4 and 5).** A matched pair proving macOS Wazuh FIM catches a newly created file but not tampering with an existing one. Defender takeaway: it is viable for detecting new persistence, but it cannot be trusted to catch edits to a file that is already there. That is the kind of limitation you only learn by testing every path and reading the logs, not by trusting the docs.

**From a generic alert to a prioritized detection (Phase 6).** The vendor syscheck event fires on any new file in the watched directory with no severity and no context. I wrote a custom rule that chains off it, escalates the level so it crosses the alerting threshold, and maps it to the ATT&CK technique. That is the step from reading alerts to engineering them.

**What changed versus who changed it (Phases 10 and 11).** Windows FIM first proved it catches a file modification, then who-data added the actor behind the change. Detecting that a file moved is useful. Knowing which user and process moved it is the part an analyst actually needs.

## Coverage

ATT&CK techniques exercised so far: T1110, T1543.001, T1565.001, T1070. More get added as phases publish.

NIST CSF: the lab walks Detect through Respond. Detection rules on the manager, automated active response, and gated remediation.

## Engineering discipline

A few habits I kept across every phase, because they are the difference between a lab that teaches you something and one that just produces screenshots:

- Custom rules live in the local reserved range so vendor ruleset updates never overwrite them.
- Every config change is backed up and hash-verified before and after.
- Detections get validated against the live pipeline, not assumed to work because the rule loaded.
- The incidents that happened to the lab itself (a manager that lost power and went dark, a network card that failed to come back after a rough boot, a flapping cable) are documented as their own investigations. Real monitoring breaks, and triaging your own infrastructure is the job.

## Evidence handling

Every screenshot and log excerpt in these writeups is sanitized. Real IPs, hostnames, usernames, and file paths are replaced with documentation-range placeholders or boxed out of images, while the evidence that matters (rule IDs, file hashes, ATT&CK mappings, timestamps) stays intact. The goal is to show the work without handing anyone a map of my network.

## Published writeups

The full phase writeups publish here over time. Available now:

- [Phase 5: macOS LaunchAgent persistence (T1543.001)](phases/phase-05-launchagent-persistence.md)

## Status

Phases 0 through 11 are complete and documented. Phase 12 (Sysmon process telemetry on the Windows agent) is in progress. More phase writeups land in the list above as I sanitize and publish them.
