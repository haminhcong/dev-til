# How to write unit tests for a Golang Project

This document is inspired by a Popular Golang Project I was working - Harrbor Container Registry.

Source Code referrenced by this document is come from Harbor v2.11.0 source code: <https://github.com/goharbor/harbor/releases/tag/v2.11.0>

To understand how unit tests are written in Harbor, we can look into some Harbor features. First, is Garbage Collector feature:

<https://github.com/goharbor/harbor/blob/v2.11.0/src/jobservice/job/impl/gc/garbage_collection.go>

We can see that this feature is mainly processed by `GarbageCollector` struct with folowing functions:

```golang

// GarbageCollector is the struct to run registry's garbage collection
type GarbageCollector struct {
	artCtl            artifact.Controller
	artrashMgr        artifactrash.Manager
	blobMgr           blob.Manager
	registryCtlClient client.Client
	logger            logger.Interface
	redisURL          string
	deleteUntagged    bool
	dryRun            bool
	// holds all of trashed artifacts' digest and repositories.
	// The source data of trashedArts is the table ArtifactTrash and it's only used as a dictionary by sweep when to delete a manifest.
	// As table blob has no repositories data, and the repositories are required when to delete a manifest, so use the table ArtifactTrash to capture them.
	trashedArts map[string][]model.ArtifactTrash
	// hold all of GC candidates(non-referenced blobs), it's captured by mark and consumed by sweep.
	deleteSet       []*blobModels.Blob
	timeWindowHours int64
	workers         int
}

func (gc *GarbageCollector) MaxFails() uint
func (gc *GarbageCollector) MaxCurrency() uint
func (gc *GarbageCollector) ShouldRetry() bool
func (gc *GarbageCollector) Validate(_ job.Parameters) error
func (gc *GarbageCollector) init(ctx job.Context, params job.Parameters) error
func (gc *GarbageCollector) parseParams(params job.Parameters)
func (gc *GarbageCollector) Run(ctx job.Context, params job.Parameters) error
func (gc *GarbageCollector) mark(ctx job.Context) error
func (gc *GarbageCollector) sweep(ctx job.Context) error
func (gc *GarbageCollector) cleanCache(ctx context.Context) error
func (gc *GarbageCollector) deletedArt(ctx job.Context) (map[string][]model.ArtifactTrash, error)
func (gc *GarbageCollector) markOrSweepUntaggedBlobs(ctx job.Context) ([]*blobModels.Blob, error)
func (gc *GarbageCollector) uselessBlobs(ctx job.Context) ([]*blobModels.Blob, error)
func (gc *GarbageCollector) markDeleteFailed(ctx job.Context, blob *blobModels.Blob) error
func (gc *GarbageCollector) shouldStop(ctx job.Context) bool
func saveGCRes(ctx job.Context, sweepSize, blobs, manifests int64) error
```

In these function, the most important function is `Run` function

```golang
func (gc *GarbageCollector) Run(ctx job.Context, params job.Parameters) error
```

And addtional for GarbageCollector work properly, in this struct we need another components likes:

- artifact.Controller
- artifactrash.Manager
- blob.Manager
- ...

to let us can write unit test for GarbageCollector `functions`, we can see that most components of GarbageCollector struct is declared as a pair of method and function

<https://github.com/goharbor/harbor/blob/v2.11.0/src/controller/artifact/controller.go>

```golang
package artifact

type Controller interface {
	// Ensure the artifact specified by the digest exists under the repository,
	// creates it if it doesn't exist. If tags are provided, ensure they exist
	// and are attached to the artifact. If the tags don't exist, create them first.
	// The "created" will be set as true when the artifact is created
	Ensure(ctx context.Context, repository, digest string, option *ArtOption) (created bool, id int64, err error)
	// Count returns the total count of artifacts according to the query.
	// The artifacts that referenced by others and without tags are not counted
	Count(ctx context.Context, query *q.Query) (total int64, err error)
	// List artifacts according to the query, specify the properties returned with option
	// The artifacts that referenced by others and without tags are not returned
	List(ctx context.Context, query *q.Query, option *Option) (artifacts []*Artifact, err error)
	// Get the artifact specified by ID, specify the properties returned with option
	Get(ctx context.Context, id int64, option *Option) (artifact *Artifact, err error)
	// Get the artifact specified by repository name and reference, the reference can be tag or digest,
	// specify the properties returned with option
	GetByReference(ctx context.Context, repository, reference string, option *Option) (artifact *Artifact, err error)
	// Delete the artifact specified by artifact ID
	Delete(ctx context.Context, id int64) (err error)
	// Copy the artifact specified by "srcRepo" and "reference" into the repository specified by "dstRepo"
	Copy(ctx context.Context, srcRepo, reference, dstRepo string) (id int64, err error)
	// UpdatePullTime updates the pull time for the artifact. If the tagID is provides, update the pull
	// time of the tag as well
	UpdatePullTime(ctx context.Context, artifactID int64, tagID int64, time time.Time) (err error)
	// GetAddition returns the addition of the artifact.
	// The addition is different according to the artifact type:
	// build history for image; values.yaml, readme and dependencies for chart, etc
	GetAddition(ctx context.Context, artifactID int64, additionType string) (addition *processor.Addition, err error)
	// AddLabel to the specified artifact
	AddLabel(ctx context.Context, artifactID int64, labelID int64) (err error)
	// RemoveLabel from the specified artifact
	RemoveLabel(ctx context.Context, artifactID int64, labelID int64) (err error)
	// Walk walks the artifact tree rooted at root, calling walkFn for each artifact in the tree, including root.
	Walk(ctx context.Context, root *Artifact, walkFn func(*Artifact) error, option *Option) error
	// HasUnscannableLayer check artifact with digest if has unscannable layer
	HasUnscannableLayer(ctx context.Context, dgst string) (bool, error)
}

type controller struct {
	tagCtl       tag.Controller
	repoMgr      repository.Manager
	artMgr       artifact.Manager
	artrashMgr   artifactrash.Manager
	blobMgr      blob.Manager
	labelMgr     label.Manager
	immutableMtr match.ImmutableTagMatcher
	regCli       registry.Client
	abstractor   Abstractor
	accessoryMgr accessory.Manager
}
```

<https://github.com/goharbor/harbor/blob/v2.11.0/src/pkg/artifactrash/manager.go>

```golang
package artifactrash

// Manager is the only interface of artifact module to provide the management functions for artifacts
type Manager interface {
	// Create ...
	Create(ctx context.Context, artifactrsh *model.ArtifactTrash) (id int64, err error)
	// Delete ...
	Delete(ctx context.Context, id int64) (err error)
	// Filter lists the artifact that needs to be cleaned, which creation_time is not in the time window.
	// The unit of timeWindow is hour, the represent cut-off is time.now() - timeWindow * time.Hours
	Filter(ctx context.Context, timeWindow int64) (arts []model.ArtifactTrash, err error)
	// Flush cleans the trash table record, which creation_time is not in the time window.
	// The unit of timeWindow is hour, the represent cut-off is time.now() - timeWindow * time.Hours
	Flush(ctx context.Context, timeWindow int64) (err error)
}


type manager struct {
	dao dao.DAO
}

func (m *manager) Create(ctx context.Context, artifactrsh *model.ArtifactTrash) (id int64, err error) {
	return m.dao.Create(ctx, artifactrsh)
}
func (m *manager) Delete(ctx context.Context, id int64) error {
	return m.dao.Delete(ctx, id)
}
func (m *manager) Filter(ctx context.Context, timeWindow int64) (arts []model.ArtifactTrash, err error) {
	return m.dao.Filter(ctx, time.Now().Add(-time.Duration(timeWindow)*time.Hour))
}

func (m *manager) Flush(ctx context.Context, timeWindow int64) (err error) {
	return m.dao.Flush(ctx, time.Now().Add(-time.Duration(timeWindow)*time.Hour))
}
```

Using a pair of interface and struct like this will help us that, when we run program, the interface and struct will auto paired: <https://go.dev/tour/methods/10>

Then when using interface, when we write unit tests for `GarbageCollector` functions, we can implement theses Interfaces by MockStruct, and replace real implement by them in tests:

<https://github.com/goharbor/harbor/blob/v2.11.0/src/jobservice/job/impl/gc/garbage_collection_test.go>

```golang
type gcTestSuite struct {
	htesting.Suite
	artifactCtl       *artifacttesting.Controller
	artrashMgr        *trashtesting.FakeManager
	registryCtlClient *registryctl.Mockclient
	projectCtl        *projecttesting.Controller
	blobMgr           *blob.Manager

	originalProjectCtl project.Controller

	regCtlInit func()
}

func (suite *gcTestSuite) SetupTest() {
	suite.artifactCtl = &artifacttesting.Controller{}
	suite.artrashMgr = &trashtesting.FakeManager{}
	suite.registryCtlClient = &registryctl.Mockclient{}
	suite.blobMgr = &blob.Manager{}
	suite.projectCtl = &projecttesting.Controller{}

	suite.originalProjectCtl = project.Ctl
	project.Ctl = suite.projectCtl

	regCtlInit = func() { commom_regctl.RegistryCtlClient = suite.registryCtlClient }
}

```

As we can see in test setup, `suite.artifactCtl` and `suite.artrashMgr` point to mock structures:

```golang
	artifacttesting "github.com/goharbor/harbor/src/testing/controller/artifact"
	trashtesting "github.com/goharbor/harbor/src/testing/pkg/artifactrash"
```

<https://github.com/goharbor/harbor/blob/v2.11.0/src/testing/controller/artifact/controller.go>

```golang
package artifact

import (
	context "context"

	artifact "github.com/goharbor/harbor/src/controller/artifact"

	mock "github.com/stretchr/testify/mock"

	processor "github.com/goharbor/harbor/src/controller/artifact/processor"

	q "github.com/goharbor/harbor/src/lib/q"

	time "time"
)

// Controller is an autogenerated mock type for the Controller type
type Controller struct {
	mock.Mock
}
```

<https://github.com/goharbor/harbor/blob/v2.11.0/src/testing/pkg/artifactrash/manager.go>

```golang
package artifactrash

import (
	"context"

	"github.com/stretchr/testify/mock"

	"github.com/goharbor/harbor/src/pkg/artifactrash/model"
)

// FakeManager is a fake tag manager that implement the src/pkg/tag.Manager interface
type FakeManager struct {
	mock.Mock
}
```

After registrer these mock objects, we can init `GarbageCollector` with theses mock objects instead real objects before run tests, and setup behavior for them:

<https://github.com/goharbor/harbor/blob/v2.11.0/src/jobservice/job/impl/gc/garbage_collection_test.go>

This is example when we write test for `deletedArt` function

```golang
func (suite *gcTestSuite) TestDeletedArt() {
	ctx := &mockjobservice.MockJobContext{}
	logger := &mockjobservice.MockJobLogger{}
	ctx.On("GetLogger").Return(logger)

	suite.artifactCtl.On("List").Return([]*artifact.Artifact{
		{
			Artifact: pkgart.Artifact{
				ID:           1,
				RepositoryID: 1,
			},
		},
	}, nil)
	suite.artifactCtl.On("Delete").Return(nil)
	suite.artrashMgr.On("Filter").Return([]model.ArtifactTrash{
		{
			ID:                1,
			Digest:            suite.DigestString(),
			ManifestMediaType: schema2.MediaTypeManifest,
		},
	}, nil)

	gc := &GarbageCollector{
		artCtl:     suite.artifactCtl,
		artrashMgr: suite.artrashMgr,
	}

	arts, err := gc.deletedArt(ctx)
	suite.Nil(err)
	suite.Equal(1, len(arts))
}
```

on `deletedArt` function, `artifactCtl` and `artrashMgr` are used to process business:

```golang

// deletedArt contains the two parts of artifact, no actually deletion for dry run mode.
// 1, required part, the artifacts were removed from Harbor.
// 2, optional part, the untagged artifacts.
func (gc *GarbageCollector) deletedArt(ctx job.Context) (map[string][]model.ArtifactTrash, error) {
	if os.Getenv("UTTEST") == "true" {
		gc.logger = ctx.GetLogger()
	}

	// allTrashedArts contains the artifacts that actual removed and simulate removed(for dry run).
	allTrashedArts := make([]model.ArtifactTrash, 0)

	// artMap : map[digest : []ArtifactTrash list]
	artMap := make(map[string][]model.ArtifactTrash)
	// handle the optional ones, and the artifact controller will move them into trash.
	if gc.deleteUntagged {
		untaggedArts, err := gc.artCtl.List(ctx.SystemContext(), &q.Query{
			Keywords: map[string]interface{}{
				"Tags": "nil",
			},
		}, &artifact.Option{WithAccessory: true})
		if err != nil {
			return artMap, err
		}
		gc.logger.Info("start to delete untagged artifact (no actually deletion for dry-run mode)")
		for _, untagged := range untaggedArts {
			// for dryRun, just simulate the artifact deletion, move the artifact to artifact trash
			if gc.dryRun {
				var simulateDeletions []model.ArtifactTrash
				err = gc.artCtl.Walk(ctx.SystemContext(), untagged, func(a *artifact.Artifact) error {
					simulateDeletion := model.ArtifactTrash{
						MediaType:         a.MediaType,
						ManifestMediaType: a.ManifestMediaType,
						RepositoryName:    a.RepositoryName,
						Digest:            a.Digest,
						CreationTime:      time.Now(),
					}
					simulateDeletions = append(simulateDeletions, simulateDeletion)
					return nil
				}, &artifact.Option{WithAccessory: true})
				if err != nil {
					gc.logger.Errorf("walk the artifact %s failed, error: %v", untagged.Digest, err)
					continue
				}
				allTrashedArts = append(allTrashedArts, simulateDeletions...)
			} else {
				if gc.shouldStop(ctx) {
					return nil, errGcStop
				}
				if err := gc.artCtl.Delete(ctx.SystemContext(), untagged.ID); err != nil {
					// the failure ones can be GCed by the next execution
					gc.logger.Errorf("failed to delete untagged:%d artifact in DB, error, %v", untagged.ID, err)
					continue
				}
			}
			gc.logger.Infof("delete the untagged artifact: ProjectID:(%d)-RepositoryName(%s)-MediaType:(%s)-Digest:(%s)",
				untagged.ProjectID, untagged.RepositoryName, untagged.ManifestMediaType, untagged.Digest)
		}
		gc.logger.Info("end to delete untagged artifact (no actually deletion for dry-run mode)")
	}

	// filter gets all of actually deleted artifact, here do not need time window as the manifest candidate has to remove all of its reference.
	// For dryRun, no need to get the actual deletion artifacts since the return map is for the mark phase to call v2 remove manifest.
	if !gc.dryRun {
		actualDeletions, err := gc.artrashMgr.Filter(ctx.SystemContext(), 0)
		if err != nil {
			return artMap, err
		}
		allTrashedArts = append(allTrashedArts, actualDeletions...)
	}

	// group the deleted artifact by digest. The repositories of blob is needed when to delete as a manifest.
	if len(allTrashedArts) > 0 {
		gc.logger.Info("artifact trash candidates.")
		for _, art := range allTrashedArts {
			gc.logger.Info(art.String())
			_, exist := artMap[art.Digest]
			if !exist {
				artMap[art.Digest] = []model.ArtifactTrash{art}
			} else {
				repos := artMap[art.Digest]
				repos = append(repos, art)
				artMap[art.Digest] = repos
			}
		}
	}

	return artMap, nil
}
```

```golang
// when artCtl is used
untaggedArts, err := gc.artCtl.List(ctx.SystemContext(), &q.Query{
	Keywords: map[string]interface{}{
		"Tags": "nil",
	},
}, &artifact.Option{WithAccessory: true})

if err := gc.artCtl.Delete(ctx.SystemContext(), untagged.ID); err != nil {

// when artrashMgr is used
actualDeletions, err := gc.artrashMgr.Filter(ctx.SystemContext(), 0)

```

because of this, we need fake their behavior before calling to `deletedArt` to start test

```golang
	suite.artifactCtl.On("List").Return([]*artifact.Artifact{
		{
			Artifact: pkgart.Artifact{
				ID:           1,
				RepositoryID: 1,
			},
		},
	}, nil)
	suite.artifactCtl.On("Delete").Return(nil)
	suite.artrashMgr.On("Filter").Return([]model.ArtifactTrash{
		{
			ID:                1,
			Digest:            suite.DigestString(),
			ManifestMediaType: schema2.MediaTypeManifest,
		},
	}, nil)
```

After setup behavior like these function run in real world, we perform init `GarbageCollector` with fake elements, call to `deletedArt` and verify result after this function run:

```golang
	gc := &GarbageCollector{
		artCtl:     suite.artifactCtl,
		artrashMgr: suite.artrashMgr,
	}
	arts, err := gc.deletedArt(ctx)
	suite.Nil(err)
	suite.Equal(1, len(arts))
```

## Golang Unit Test Packages - Frameworks

- `testify`- <https://github.com/stretchr/testify> using for create mock and suites
