# Osysharp.Artifacts

Things made for review — an agent-made HTML prototype, a picture, a PDF, a markdown note — kept on a row of your app with
every version, commented per version, and shown in a viewer the browser sandboxes twice.

```osy
app Studio {
  use Osysharp.Artifacts@1;
  use Osysharp.Storage;
  use Osysharp.Ui;
}

policy ReadsArtifactsOf(string subjectType, Guid subjectId) => …;
policy WritesArtifactsOf(string subjectType, Guid subjectId) => …;

var a = Artifact.Publish("Design", design.Id, "index.html", "text/html", bytes);   // again → a new version; old ones kept
ArtifactView(a.File.CurrentVersion, h: 720);

// A prototype with files: the entry loads them by relative path, served beside it.
ArtifactAsset.PublishBundle("Design", design.Id, "index.html", page, files);   // files: List<ArtifactAsset.BundleFile> { Path, ContentType, Bytes }

// Each artifact on an origin of its own, on a domain that is not the app's site.
app.Artifacts = new ArtifactsSetup { ViewerBase = "osyrin.dev" };
```

- **Versions.** An artifact holds one `FileAsset`; publishing under the same name supersedes it (`File.Supersede`), so
  every earlier version stays (`artifact.Versions()`).
- **Safe viewer.** Bytes come only from the platform's `/_artifacts/{version}/{token}/{name}`, marked with the CSP
  `sandbox` directive (an opaque origin even when opened directly; no network, forms, popups or navigation; script for
  HTML only), in an `ArtifactFrame` with `sandbox="allow-scripts"`. Its address is signed for the reader after their
  own read.
- **Its own origin.** With `ArtifactsSetup.ViewerBase` (or `ArtifactView(viewer:)`), each artifact is served from
  `https://<artifact>-<placement>.<base>/…`: never the app's site, never another artifact's origin, nothing else on that
  host, a token for that host alone, framed only by the app's page.
- **Bundles.** `ArtifactAsset.PublishBundle`: an HTML entry plus scripts, styles, pictures and fonts (≤ 300 files, ≤ 8 deep,
  the whole ≤ `MaxBytes`), loaded by relative path under the same signed address. No fetches (`connect-src 'none'`).
- **Checked uploads.** Ten allow-listed types, bytes sniffed, ≤ 25 MB; an organisation's policy may lower the cap,
  forbid HTML prototypes and fix the viewer base (`ArtifactsSetup`).
- **Comments** on one version, optionally at a point of it.

1.0.0 grouped the calls into the types they are about (`PublishArtifact` → `Artifact.Publish`): see CHANGELOG.md.

Reference: `osy docs ui-artifacts-kit`. Tests: `osy test kits/artifacts/tests`.

## Examples

Taken from this package's own tests, which compile and run on every release, and from its reference page, whose examples are compiled when the site is built.

```osy
using Osysharp.Identity;
using Osysharp.Storage;
using Osysharp.Artifacts;

[Test(Checkout)]
void a_comment_names_its_spot() {
  Guid id;
  runas (Ann) {
    id = Artifact.Publish("Design", CheckoutId(), "spots.html", "text/html", Encoding.UTF8.GetBytes("<h1>Pay</h1><button id=\"pay\">Pay</button>")).Id;
    var spotPath = new List<ArtifactAsset.BundleFile>();
    spotPath.Add(new ArtifactAsset.BundleFile { Path = ".osy/spots.js", ContentType = "text/javascript", Bytes = Encoding.UTF8.GetBytes("1") });
    Assert.Throws<ValidationException>(() => ArtifactAsset.PublishBundle("Design", CheckoutId(), "taken.html", Encoding.UTF8.GetBytes("<p>t</p>"), spotPath));
    UnitOfWork.Commit();
  }
  runas (Rui) {
    var artifact = Artifact.Single(a => a.Id == id);
    ArtifactComment.OnArtifact(artifact, 1, "Make it bigger.", 0.5m, 0.25m, "pay");
    ArtifactComment.OnArtifact(artifact, 1, "Overall fine.");
    Assert.Throws<ValidationException>(() => ArtifactComment.OnArtifact(artifact, 1, "Here.", null, null, "not a key"));
    UnitOfWork.Commit();
    var said = ArtifactComment.Where(c => c.Artifact == artifact).OrderBy(c => c.Text).ToList();
    Assert.Equal("pay", said[0].Spot);
    Assert.Equal("page", said[1].Spot);
  }
}
```

From [`tests/artifacts.test.osy`](tests/artifacts.test.osy).

```osy
app Studio {
  use Osysharp.Artifacts@1;
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

## Learn more

- [The Osysharp.Artifacts reference](https://osysharp.com/reference/ui/artifacts-kit/)
- [Osysharp.Artifacts in the package catalogue](https://osysharp.com/packages/#osysharp-artifacts)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
