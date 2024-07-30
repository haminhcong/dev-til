# How to implement scheduler process in Go

The following way is one of many ways to implement scheduler process in Golang.

To implement scheduler with high-availability feature, we need an etcd cluster to acquire distibuted locks. Following this document to create 3-nodes etcd cluster in VM:

[https://facsiaginsa.com/etcd/how-to-setup-etcd-cluster
](https://facsiaginsa.com/etcd/how-to-setup-etcd-cluster)

Scheduler usually run by interval, using [github.com/go-co-op/gocron/v2](github.com/go-co-op/gocron/v2) gocron package to run task in interval. For example:

```golang
package main

import (
	"context"
	"errors"
	httpapi2c "demo/scheduler/client/httpapi2"
	"demo/scheduler/client/etcd"
	client1 "demo/scheduler/client/server1"
	"demo/scheduler/logging"
	"demo/scheduler/models"
	"demo/scheduler/utils"
	"github.com/go-co-op/gocron/v2"
	"github.com/google/uuid"
	"github.com/joho/godotenv"
	"go.etcd.io/etcd/client/v3/concurrency"
	"os"
	"os/signal"
	"strconv"
	"syscall"
	"time"
)

func RunTask1(ctx context.Context, Model1 *models.Model1) {
	logging.Info(rctx, "Total server1 Models count: %d", len(Model1List))

}

func RunTask0(ctx context.Context) {

	taskID := uuid.New().String()
	rctx := context.WithValue(ctx, utils.RequestIDKey{}, taskID)
	// acquire lock
	session, err := concurrency.NewSession(etcd.Ec, concurrency.WithTTL(etcd.LockTimeOut), concurrency.WithContext(rctx))
	if err != nil {
		logging.Error(rctx, "Faied to connect to Etcd Cluster, reason: %s", err)
		return
	}
	defer session.Close()

	m := etcd.NewMutex(session, utils.UpdateModelLockKey)

	if err := m.TryLock(rctx); err != nil {
		if errors.Is(err, etcd.ErrLocked) {
			logging.Warning(rctx, "Etcd lock key is locked. Update Model list usage task canceled.")
			return
		} else {
			logging.Error(rctx, "Failed to lock key for Update Model list usage task, detail %s - Task canceled", err)
			return
		}
	}
	defer func() {
		logging.Info(rctx, "Releasing Lock %s", utils.UpdateModelLockKey)
		_ = m.Unlock(rctx)
		logging.Info(rctx, "Released Lock %s", utils.UpdateModelLockKey)
	}()

	logging.Info(rctx, "Acquired Lock. Start Updateing all Model from server1 Core Service to httpapi2")
	logging.Info(rctx, "Total server1 Models count: %d", len(Model1List))
	// limit concurrency
	maxGoroutines := 50
	guard := make(chan struct{}, maxGoroutines)
	for _, hp := range Model1List {
		guard <- struct{}{} // would block if guard channel is already filled
		go func(hp *models.Model1) {
			RunTask1(rctx, hp)
			<-guard
		}(hp)
		//go RunTask1(rctx, hp)
	}
	logging.Info(rctx, "Updateing all Model from server1 Core Service to httpapi2 Done. Task Success")

}

func handleSignals(ctx context.Context, c chan os.Signal) {

	switch <-c {
	case syscall.SIGINT, syscall.SIGTERM:
		logging.Info(ctx, "Shutdown quickly, bye...")
		//utils.CloseDBConnection()
		logging.Info(ctx, "Closed DB Connection")
	case syscall.SIGQUIT:
		logging.Info(ctx, "Shutdown gracefully, bye...")
		// do graceful shutdown
		etcd.CloseDBConnection()
		logging.Info(ctx, "Closed DB Connection")
	}
	logging.Info(ctx, "Shutdown success, bye...")
	os.Exit(0)
}

func main() {
	ctx := context.Background()
	logging.Init()
	err := godotenv.Load()
	if err != nil {
		logging.Info(ctx, ".env file is not found. Loading config from env...")
	}
	// Init logging
	logging.Info(ctx, "Starting server1 Periodic Jobs")
	httpapi2RegionID := os.Getenv("httpapi2_REGION_ID")

	if httpapi2RegionID == "" {
		logging.Error(ctx, "Failed to get httpapi2 Region ID")
		panic("")
	}
	// Init HTTP Clients
	httpapi2URL := os.Getenv("httpapi2_URL")
	httpapi2APIToken := os.Getenv("httpapi2_API_TOKEN")
	if httpapi2URL == "" || httpapi2APIToken == "" {
		logging.Error(ctx, "Failed to get httpapi2 URL or httpapi2 API TOKEN")
		panic("")
	}
	httpapi2c.Init(httpapi2URL, httpapi2APIToken, DefaultHttpTimeout)
	server1URL := os.Getenv("server1_URL")
	server1APIUserName := os.Getenv("server1_API_USERNAME")
	server1APIPassword := os.Getenv("server1_API_PASSWORD")

	if server1URL == "" {
		logging.Error(ctx, "Failed to get server1 URL")
		panic("")
	}
	if server1APIUserName == "" || server1APIPassword == "" {
		logging.Error(ctx, "Failed to get server1 Username or Password")
		panic("")
	}

	etcdConnStr := os.Getenv("ETCD_CONN_STR")
	if etcdConnStr == "" {
		logging.Error(ctx, "Failed to get etcd connection string")
		panic("")
	}

	etcd.InitDBConnection(etcdConnStr)

	server1HTTPTimeout := DefaultHttpTimeout
	server1HTTPTimeoutStr := os.Getenv("server1_API_HTTP_TIMEOUT")
	if server1HTTPTimeoutStr != "" {
		server1HTTPTimeout, _ = strconv.Atoi(server1HTTPTimeoutStr)
	}

	client1.Init(server1URL, server1APIUserName, server1APIPassword, server1HTTPTimeout)

	// create a scheduler
	s, err := gocron.NewScheduler()
	if err != nil {
		// handle error
	}

	// add a job to the scheduler
	RunTask0Interval := DefaultUpdateTaskInterval
	UpdateIntervalStr := os.Getenv("Update_INTERVAL")
	if UpdateIntervalStr != "" {
		RunTask0Interval, _ = strconv.Atoi(UpdateIntervalStr)
	}

	j, err := s.NewJob(
		gocron.DurationJob(
			time.Duration(RunTask0Interval*SecondFactor),
		),
		gocron.NewTask(
			RunTask0,
			ctx,
		),
	)
	if err != nil {
		logging.Error(ctx, "Failed to add jobs, reason: %s - Program will exit", err)
		// handle error
		panic(err)
	}
	jobID := j.ID().String()
	// each job has a unique id
	logging.Info(ctx, "Running job %s - %s", jobID, j.Name())
	// start the scheduler
	s.Start()

	sigs := make(chan os.Signal, 1)
	signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT)

	<-sigs
	// when you're done, shut it down
	err = s.Shutdown()
	if err != nil {
		// handle error
	}
	// Block until signals is handled
	//go handleSignals(sigs)
	handleSignals(ctx, sigs)
}

```

In this simple process, we used:

`godotenv` to load environment to env variable

Then checking if env exist:

```golang
	// Init logging
	logging.Info(ctx, "Starting server1 Periodic Jobs")
	httpapi2RegionID := os.Getenv("httpapi2_REGION_ID")

	if httpapi2RegionID == "" {
		logging.Error(ctx, "Failed to get httpapi2 Region ID")
		panic("")
	}
```

After every conditions are satifisfied, we create scheduler goroutine with interval setting from config

```golang
	j, err := s.NewJob(
		gocron.DurationJob(
			time.Duration(RunTask0Interval*SecondFactor),
		),
		gocron.NewTask(
			RunTask0,
			ctx,
		),
	)
	if err != nil {
		logging.Error(ctx, "Failed to add jobs, reason: %s - Program will exit", err)
		// handle error
		panic(err)
	}
	jobID := j.ID().String()
	// each job has a unique id
	logging.Info(ctx, "Running job %s - %s", jobID, j.Name())
	// start the scheduler
	s.Start()
```

when mulitple scheduler instances is created, multiple tasks `RunTask0` will run concurrently. To ensure only one task is run at any moment, in  `RunTask0` before process real task, we need to acquire lock from etcd cluster:

```golang
	taskID := uuid.New().String()
	rctx := context.WithValue(ctx, utils.RequestIDKey{}, taskID)
	// acquire lock
	session, err := concurrency.NewSession(etcd.Ec, concurrency.WithTTL(etcd.LockTimeOut), concurrency.WithContext(rctx))
	if err != nil {
		logging.Error(rctx, "Faied to connect to Etcd Cluster, reason: %s", err)
		return
	}
	defer session.Close()

	m := etcd.NewMutex(session, utils.UpdateModelLockKey)

	if err := m.TryLock(rctx); err != nil {
		if errors.Is(err, etcd.ErrLocked) {
			logging.Warning(rctx, "Etcd lock key is locked. Update Model list usage task canceled.")
			return
		} else {
			logging.Error(rctx, "Failed to lock key for Update Model list usage task, detail %s - Task canceled", err)
			return
		}
	}
	defer func() {
		logging.Info(rctx, "Releasing Lock %s", utils.UpdateModelLockKey)
		_ = m.Unlock(rctx)
		logging.Info(rctx, "Released Lock %s", utils.UpdateModelLockKey)
	}()

	logging.Info(rctx, "Acquired Lock. Start Updateing all Model from server1 Core Service to httpapi2")
	logging.Info(rctx, "Total server1 Models count: %d", len(Model1List))
	// limit concurrency
```

If lock is acquired, real task is allowed to run. If lock is not acquried, it means that another task is running hand hold distributed lock. Therefore, current task will need to be canceled. 

```golang
	m := etcd.NewMutex(session, utils.UpdateModelLockKey)
	if err := m.TryLock(rctx); err != nil {
		if errors.Is(err, etcd.ErrLocked) {
			logging.Warning(rctx, "Etcd lock key is locked. Update Model list usage task canceled.")
			return
		} else {
			logging.Error(rctx, "Failed to lock key for Update Model list usage task, detail %s - Task canceled", err)
			return
		}
	}
```

After real task is processed, we need to release lock by `defer` code block:

```bash
	defer func() {
		logging.Info(rctx, "Releasing Lock %s", utils.UpdateModelLockKey)
		_ = m.Unlock(rctx)
		logging.Info(rctx, "Released Lock %s", utils.UpdateModelLockKey)
	}()

```

## Handling jobs concurrently

On some scenarios, we need to run multiple job concurrently, but at a concurrency limit degree to prevent too many call to external resources (external api, db connection...). To achive these goals, we can using goroutine and channel with limit length to create concurrent jobs:

```golang
	// limit concurrency
	maxGoroutines := 50
	guard := make(chan struct{}, maxGoroutines)
	for _, hp := range Model1List {
		guard <- struct{}{} // would block if guard channel is already filled
		go func(hp *models.Model1) {
			RunTask1(rctx, hp)
			<-guard
		}(hp)
		//go RunTask1(rctx, hp)
	}
	logging.Info(rctx, "Updateing all Model from server1 Core Service to httpapi2 Done. Task Success")
```

With this solution, even when input list contains millions elements, the maximum concurrent goroutines run at any moments is still 50 goroutines.
