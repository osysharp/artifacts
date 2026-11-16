# Osysharp.Artifacts

Things made for review — an agent-made HTML prototype, a picture, a PDF, a markdown note — kept on a row of your app with
every version, commented per version, and shown in a viewer the browser sandboxes twice.

```osy
app Studio {
  use Osysharp.Artifacts@0;
  use Osysharp.Storage;
  use Osysharp.Ui;
}

policy ReadsArtifactsOf(string subjectType, Guid subjectId) => …;
policy WritesArtifactsOf(string subjectType, Guid subjectId) => …;

var a = PublishArtifact("Design", design.Id, "index.html", "text/html", bytes);   // again → a new version; old ones kept
ArtifactView(a.File.CurrentVersion, h: 720);
```

- **Versions.** An artifact holds one `FileAsset`; publishing under the same name supersedes it (`File.Supersede`), so
  every earlier version stays (`ArtifactVersions`).
- **Safe viewer.** Bytes come only from the platform's `/_artifacts/{version}/{name}`, marked with the CSP `sandbox`
  directive (an opaque origin even when opened directly; no network, forms, popups or navigation; script for HTML only),
  in an `ArtifactFrame` with `sandbox="allow-scripts"`. Its address is signed for the reader after their own read.
- **Checked uploads.** Ten allow-listed types, bytes sniffed, ≤ 25 MB; an organisation's policy may lower the cap and
  forbid HTML prototypes (`ArtifactsSetup`).
- **Comments** on one version, optionally at a point of it.

Reference: `osy docs ui-artifacts-kit`. Tests: `osy test kits/artifacts/tests`.

## Examples

Taken from this package's own tests, which compile and run on every release, and from its reference page, whose examples are compiled when the site is built.

```osy
app Studio {
  use Osysharp.Artifacts@0;
  use Osysharp.Storage;
  use Osysharp.Ui;
  model "model/**/*.osy";
}

using Osysharp.Ui;
using Osysharp.Artifacts;

entity Design {
  [Required, MaxLength(80)] string Title;
  security { allow read when IsAuthenticated; }
}

// Whoever may read a design sees its artifacts and publishes on it.
policy ReadsArtifactsOf(string subjectType, Guid subjectId) => subjectType == "Design" && Design.Any(d => d.Id == subjectId);
policy WritesArtifactsOf(string subjectType, Guid subjectId) => ReadsArtifactsOf(subjectType, subjectId);

[Route("/designs/{id}/artifacts")]
component DesignArtifacts(Guid id) {
  render {
    Stack(gap: 2) {
      foreach (var a in Artifact.Where(x => x.SubjectType == "Design" && x.SubjectId == id).Include(x => x.File.CurrentVersion).ToList()) {
        Text(a.Name);
        ArtifactView(a.File.CurrentVersion, h: 720);
      }
    }
  }
}
```

From [the reference page](https://osysharp.com/reference/ui/artifacts-kit/).

```osy
using Osysharp.Identity;
using Osysharp.Storage;
using Osysharp.Artifacts;
using Studio;

[Test(Checkout)]
void a_new_version_keeps_the_old() {
  Guid design;
  Guid id;
  runas (Ann) {
    design = CheckoutId();
    id = PublishArtifact("Design", design, "index.html", "text/html", Encoding.UTF8.GetBytes("<h1>One</h1>")).Id;
    UnitOfWork.Commit();
  }
  runas (Rui) {
    CommentOnArtifact(Artifact.Single(a => a.Id == id), 1, "The pay button is too small.", 0.5m, 0.25m);
    UnitOfWork.Commit();
  }
  runas (Ann) {
    var again = PublishArtifact("Design", design, "index.html", "text/html", Encoding.UTF8.GetBytes("<h1>Two</h1>"));
    UnitOfWork.Commit();
    Assert.Equal(id, again.Id);
    Assert.Throws<ValidationException>(() => PublishArtifact("Design", design, "shot.png", "image/png", Encoding.UTF8.GetBytes("<script>1</script>")));
    Assert.Throws<ValidationException>(() => PublishArtifact("Design", design, "index.html", "text/plain", Encoding.UTF8.GetBytes("two")));
  }
  runas (Rui) {
    var artifact = Artifact.Single(a => a.Id == id);
    var versions = ArtifactVersions(artifact);
    Assert.Equal(2, versions.Count);
    Assert.Equal(1, versions[0].Version);
    Assert.Equal(Crypto.Sha256Hex("<h1>One</h1>"), versions[0].ContentHash);
    Assert.Equal(2, versions[1].Version);
    Assert.Equal(Crypto.Sha256Hex("<h1>Two</h1>"), versions[1].ContentHash);
    Assert.Equal(versions[1].Id, artifact.File.CurrentVersion.Id);
    Assert.Equal("<h1>Two</h1>", Encoding.UTF8.GetString(File.ReadAllBytes(artifact.File)));
    var said = ArtifactComment.Where(c => c.Artifact == artifact).ToList();
    Assert.Equal(1, said.Count);
    Assert.Equal(1, said[0].Version);
  }
  runas (Bo) {
    Assert.False(Artifact.Any(a => a.Id == id));
    Assert.False(ArtifactComment.Any(c => c.Artifact.Id == id));
  }
}
```

From [`tests/artifacts.test.osy`](tests/artifacts.test.osy).

## Learn more

- [The Osysharp.Artifacts reference](https://osysharp.com/reference/ui/artifacts-kit/)
- [Osysharp.Artifacts in the package catalogue](https://osysharp.com/packages/#osysharp-artifacts)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
