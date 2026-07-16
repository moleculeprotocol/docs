---
icon: landmark-flag
---

# Labs API

## Labs API Reference

The full, current Labs API reference lives at [API Reference → Labs API](../api-reference/labs-api.md). This page is kept as a migration pointer for readers arriving from older links.

{% hint style="warning" %}
The pre-OCL Labs API documented here previously — `ipnftUid` identifiers and the `*V2` operations — **has been removed from the API**. If your integration still uses those names, migrate using the tables below.
{% endhint %}

### Migrating from the removed V2 API

Identifiers changed from IPNFT-based to Lab-based:

| Removed          | Current                                                     |
| ---------------- | ----------------------------------------------------------- |
| `ipnftUid`       | `oclId` (canonical 32-byte lab identifier, `0x` + 64 hex)   |
| `ipnftSymbol`    | `shortname`                                                 |
| `ipnftAddress`   | `labAccountAddress`                                         |
| `ipnftTokenId`   | `labNftTokenId`                                             |

Operations were renamed (same intent, new names and `oclId`-based arguments):

| Removed                          | Current                          |
| -------------------------------- | -------------------------------- |
| `projects` / `projectsV2`        | `labs`                           |
| `projectWithDataRoomAndFilesV2`  | `labWithDataRoomAndFiles`        |
| `dataRoomFileV2`                 | `dataRoomFile`                   |
| `projectActivity` / `projectActivityV2` | `labActivity`            |
| `activitiesV2`                   | `activities`                     |
| `projectAnnouncementsV2`         | removed — announcements are surfaced via `labActivity` / `activities` |
| `createProject`                  | `createLab`                      |
| `initiateCreateOrUpdateFileV2`   | `initiateCreateOrUpdateFile`     |
| `finishCreateOrUpdateFileV2`     | `finishCreateOrUpdateFile`       |
| `updateFileMetadataV2`           | `updateFileMetadata`             |
| `deleteDataRoomFileV2`           | `deleteDataRoomFile`             |
| `createAnnouncementV2`           | `createAnnouncement`             |

The owner-management mutations (`addLabOwner` / `removeLabOwner`) were retired without replacement — lab membership is managed on-chain via the [AccessResolver](contracts/accessresolver.md) role system. The `dataRoomPassphrase` mutation (Telegram-bot passphrase) still appears in the schema but its resolver has been retired, so it is no longer functional.

See the [current Labs API reference](../api-reference/labs-api.md) for full signatures, auth, and examples, including the complete [Deprecated & Renamed Operations](../api-reference/labs-api.md#deprecated--renamed-operations) table.
