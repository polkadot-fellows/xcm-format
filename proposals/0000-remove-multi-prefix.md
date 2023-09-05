---
Title: Remove "Multi" Prefix
Number: 0
Status: Draft
Version: 0
Authors:
- Francisco Aguirre
Created: 2023-09-05
Impact: Low
Requires:
Replaces:
---

## Summary

The "Multi" prefix on "MultiLocation" and "MultiAsset" was originally designed to mean "Versioned".
However, the versioned types already use the "Versioned" prefix, making things like "VersionedMultiLocation" unnecessary naming.
This RFC proposes to remove the "Multi" prefix, to simplify the naming.

## Motivation

There's no need to have a prefix that has no use.

## Specification

The following types on the spec shall be renamed:
- MultiLocation -> Location
- MultiAsset -> Asset
- MultiAssets -> MultiAssets
- InteriorMultiLocation -> InteriorLocation
- MultiAssetFilter -> AssetFilter
- VersionedMultiAsset -> VersionedAsset
- WildMultiAsset -> WildAsset
- VersionedMultiLocation -> VersionedLocation

## Impact

The RFC proposes a name change, so it's only low impact.

## Alternatives

Things could be left as is or we could rename the types mentioned above to another thing.
One idea is to rename `WildAsset` to `AssetWildcard`, which seems to convey its meaning better.

## Questions and open Discussions

None.
