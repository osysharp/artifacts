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

```osy
using Osysharp.Identity;
using Osysharp.Storage;
using Osysharp.Artifacts;

[Test(Checkout)]
void a_new_version_keeps_the_old() {
  Guid design;
  Guid id;
  runas (Ann) {
    design = CheckoutId();
    id = Artifact.Publish("Design", design, "index.html", "text/html", Encoding.UTF8.GetBytes("<h1>One</h1>")).Id;
    UnitOfWork.Commit();
  }
  runas (Rui) {
    ArtifactComment.OnArtifact(Artifact.Single(a => a.Id == id), 1, "The pay button is too small.", 0.5m, 0.25m);
    UnitOfWork.Commit();
  }
  runas (Ann) {
    var again = Artifact.Publish("Design", design, "index.html", "text/html", Encoding.UTF8.GetBytes("<h1>Two</h1>"));
    UnitOfWork.Commit();
    Assert.Equal(id, again.Id);
    Assert.Throws<ValidationException>(() => Artifact.Publish("Design", design, "shot.png", "image/png", Encoding.UTF8.GetBytes("<script>1</script>")));
    Assert.Throws<ValidationException>(() => Artifact.Publish("Design", design, "index.html", "text/plain", Encoding.UTF8.GetBytes("two")));
  }
  runas (Rui) {
    var artifact = Artifact.Single(a => a.Id == id);
    var versions = artifact.Versions();
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

```osy
using Osysharp.Identity;
using Osysharp.Storage;
using Osysharp.Artifacts;

[Test(Checkout)]
void a_bundles_relative_files_load_under_its_entrys_address() {
  var page = "<!doctype html><link rel=stylesheet href=\"css/site.css\"><img src=\"img/dot.png\"><script src=\"app.js\"></script>";
  var dot = Convert.FromBase64String("iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk+M9QDwADhgGAWjR9awAAAABJRU5ErkJggg==");
  Guid id;
  Guid version;
  runas (Ann) {
    var design = CheckoutId();
    var files = new List<ArtifactAsset.BundleFile>();
    files.Add(new ArtifactAsset.BundleFile { Path = "app.js", ContentType = "text/javascript", Bytes = Encoding.UTF8.GetBytes("document.title = 'ran';") });
    files.Add(new ArtifactAsset.BundleFile { Path = "css/site.css", ContentType = "text/css", Bytes = Encoding.UTF8.GetBytes("body { margin: 0 }") });
    files.Add(new ArtifactAsset.BundleFile { Path = "img/dot.png", ContentType = "image/png", Bytes = dot });
    id = ArtifactAsset.PublishBundle("Design", design, "index.html", Encoding.UTF8.GetBytes(page), files).Id;
    UnitOfWork.Commit();

    var bad = new List<ArtifactAsset.BundleFile>();
    bad.Add(new ArtifactAsset.BundleFile { Path = "../secret.js", ContentType = "text/javascript", Bytes = Encoding.UTF8.GetBytes("1") });
    Assert.Throws<ValidationException>(() => ArtifactAsset.PublishBundle("Design", design, "other.html", Encoding.UTF8.GetBytes(page), bad));
    var json = new List<ArtifactAsset.BundleFile>();
    json.Add(new ArtifactAsset.BundleFile { Path = "data.json", ContentType = "application/json", Bytes = Encoding.UTF8.GetBytes("{}") });
    Assert.Throws<ValidationException>(() => ArtifactAsset.PublishBundle("Design", design, "other.html", Encoding.UTF8.GetBytes(page), json));
    var lie = new List<ArtifactAsset.BundleFile>();
    lie.Add(new ArtifactAsset.BundleFile { Path = "app.js", ContentType = "text/javascript", Bytes = dot });
    Assert.Throws<ValidationException>(() => ArtifactAsset.PublishBundle("Design", design, "other.html", Encoding.UTF8.GetBytes(page), lie));
  }
  runas (Rui) {
    var artifact = Artifact.Single(a => a.Id == id);
    Assert.Equal(3, artifact.Assets(artifact.File.CurrentVersion.Version).Count);
    version = artifact.File.CurrentVersion.Id;
    var url = SignedFor(version, Api.TokenFor(Rui));
    Assert.StartsWith("/_artifacts/" + artifact.File.CurrentVersion.Id.ToString() + "/", url);
    Assert.EndsWith("/index.html", url);
    var folder = url.Substring(0, url.Length - "index.html".Length);

    var entry = Api.Get(url);
    Assert.Equal(200, entry.StatusCode);
    Assert.Equal(page, entry.Body);
    var script = Api.Get(folder + "app.js");
    Assert.Equal(200, script.StatusCode);
    Assert.Equal("document.title = 'ran';", script.Body);
    Assert.StartsWith("text/javascript", HeaderOf(script, "Content-Type"));
    Assert.StartsWith("sandbox", HeaderOf(script, "Content-Security-Policy"));
    Assert.Contains("connect-src 'none'", HeaderOf(entry, "Content-Security-Policy"));
    Assert.Equal(200, Api.Get(folder + "css/site.css").StatusCode);
    Assert.Equal(200, Api.Get(folder + "img/dot.png").StatusCode);
    Assert.Equal(404, Api.Get(folder + "img/other.png").StatusCode);
  }
  runas (Bo) {
    Assert.Equal("", SignedFor(version, Api.TokenFor(Bo)));
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
