# Osysharp.Artifacts — changes

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
