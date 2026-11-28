# Osysharp.Artifacts — changes

## 1.1.0
- Spots (PLATFORM-694). `ArtifactView` takes `spots`, `pins`, `focus` and `onSpot`: with `spots: true` an HTML artifact is
  served with the platform's spot script, a click while `onSpot` is bound reports the `ArtifactSpot` the reader picked
  (its key, the point within it and its first words), `pins` (`ArtifactPin[]`) are drawn at their spots with labels of at
  most 3 characters, and `focus` scrolls to a spot. The frame's sandbox and the artifact's CSP are unchanged.
- `ArtifactComment` has a `Spot` ("page" unless one is named), and `ArtifactComment.OnArtifact` takes `spot` after `y`;
  `ArtifactComment.SpotRefusal(spot)` says what is wrong with a key.
- Needs platform 0.25.2 (`minPlatform "0.25.2"`).

## 1.0.0
- Grouped into classes (`osy restructure`). Calls move:
  - `ArtifactAssetTypes()` → `ArtifactAsset.Types()`
  - `ArtifactAssets(artifact, …)` → `artifact.Assets(…)`
  - `ArtifactBytesLie(…)` → `Artifact.BytesLie(…)`
  - `ArtifactFile` → `ArtifactAsset.BundleFile` (a class)
  - `ArtifactIsText(…)` → `Artifact.IsText(…)`
  - `ArtifactKindOf(…)` → `Artifact.KindOf(…)`
  - `ArtifactLeaf(…)` → `ArtifactAsset.Leaf(…)`
  - `ArtifactMagic(…)` → `Artifact.FormatOf(…)`
  - `ArtifactPathRefusal(…)` → `ArtifactAsset.PathRefusal(…)`
  - `ArtifactTypes()` → `Artifact.Types()`
  - `ArtifactVersions(artifact)` → `artifact.Versions()`
  - `CommentOnArtifact(…)` → `ArtifactComment.OnArtifact(…)`
  - `PublishArtifact(…)` → `Artifact.Publish(…)`
  - `PublishArtifactBundle(…)` → `ArtifactAsset.PublishBundle(…)`
  - `ResolveArtifactComment(comment, …)` → `comment.Resolve(…)`
- Published tools, actions and actors keep their names: what calls them by name is unchanged.
- A project moves when it bumps the kit: `use Osysharp.Artifacts@1;`, then the calls above.

## 0.4.0
- Its organization-policy settings name their area from the platform's list (`Area = OrgPolicyArea.Agents`), not as text.
  Needs platform 0.23.0 (`minPlatform "0.23.0"`): an older platform refuses this version by its floor.
