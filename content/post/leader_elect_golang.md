+++
title = "基于数据库的简单Leader Elect"
date = 2019-03-11T15:14:25+08:00
description = "基于数据库的简单Leader Elect"
Tags = ["Leader Elect"] 
Categories = ["golang"]
+++


某个系统只用来执行定时任务，如果只部署单台服务，那么又容易单点故障，如果部署多台服务，又如何只保证每次只会其中一台去执行呢，在这里，可以对N台服务，做一个简单的leader elect，成为leader的实例才可以去执行定时任务。虽然当前出现很多开源的leader选举组件，比如zookeeper、etcd等，但有时候对我们个人开发者或者小项目来说，代价太高。因此，我们可以基于mysql数据库进行一个简单的leader elect。

### 设计
首先，新建一张表，表结构如下
```MYSQL
CREATE TABLE IF NOT EXISTS `leader_election` (
  `service_id` varchar(50) NOT NULL DEFAULT '',
  `leader_id` varchar(50) NOT NULL DEFAULT '',
  `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`service_id`),
  UNIQUE KEY `uniq_service_id` (`service_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
service_id指的是某个具体的任务Id，leader_id指的是某台竞选为leader的实例id，比如有A、B两台服务，那A成为leader，leader_id则是A的唯一标志。

实现的sql语句
```MYSQL
insert ignore into leader_election (service_id, leader_id, update_time) values (?, ?, now())
on duplicate key update leader_id = if(timestampdiff(second, update_time, now()) > ?, values(leader_id), leader_id),
update_time = if(leader_id = values(leader_id), values(update_time), update_time)
```
### 代码实现

首先我们实现一个简单的选举接口，文件election.go
```Go
type Election interface {
    IsLeader() (bool, error)
    BecomeLeader() (bool, error)
    Id() string
}
 
type Elector struct {
    election Election
}
 
func NewElector(election Election) *Elector {
    elector := &Elector{election: election}
    return elector
}
 
func (e *Elector) Id() string {
    return e.election.Id()
}
 
func (e *Elector) IsLeader() (bool, error) {
    return e.election.IsLeader()
}
 
func (e *Elector) BecomeLeader() (bool, error) {
    leader, err := e.election.BecomeLeader()
    if err != nil {
        log.WithField("err", err.Error()).Error("Error while trying to become a leader")
    }
    if leader {
        log.WithField("id", e.Id()).Info("Become a leader successfully")
    }
    return leader, err
}
```
接着来一个具体的实现mysql_elector.go
```Go
type MysqlElector struct {
    db *sql.DB
    id string
    serviceId string
}

func NewMysqlElector(db *sql.DB, id string, serviceId string) (*MysqlElector, error) {
    elector := &MysqlElector{db: db, id: id, serviceId: serviceId}
    _, err := elector.BecomeLeader()
    if err != nil {
        return nil, err
    }
    return elector, nil
}

func (e *MysqlElector) Id() string {
    return e.id
}

func (e *MysqlElector) IsLeader() (bool, error) {
    r, err := db.CountLeaderElection(e.serviceId, e.id)
    if err != nil {
        return false, err
    }
    return r > 0, nil
}

func (e *MysqlElector) BecomeLeader() (bool, error) {
    _, err := db.InsertLeaderElection(e.serviceId, e.id, cons.ElectionTimeOut)
    if err != nil {
        return false, err
    }
    return e.IsLeader()
}
```
每台服务需要周期性去竞选leader，因为leader服务一旦宕机，另外一台服务可以马上竞选为leader。这里设定每隔10s去竞选leader或者维护leader状态，如果某个leader超过30s没有维护自己的leader状态，则leader会被其他实例获取。

编写一个定时调度任务，去竞选leader以及维护leader状态。
```Go
// ScheduledElector triggers election on scheduled interval
type ScheduledElector struct {
    Elector
    ticker  *time.Ticker
    paused   bool
}
 
func NewScheduledElector(election Election) *ScheduledElector {
    scheduledElector := &ScheduledElector{Elector: Elector{election}, ticker: time.NewTicker(time.Second * cons.ElectionInterval)}
    go scheduledElector.scheduledElection()
    return scheduledElector
}
 
 
func (se *ScheduledElector) pausedScheduledElection() {
    se.paused = true
}
 
func (se *ScheduledElector) resumeScheduledElection() {
    se.paused = false
}
 
func (se *ScheduledElector) IsPausedScheduledElection() bool {
    return se.paused
}
 
func (se *ScheduledElector) Elect() (bool, error) {
    leader, err := se.BecomeLeader()
    if err != nil {
        log.WithFields(logrus.Fields{"id": se.Id(), "err": err.Error()}).Error("Error while become leader")
    }
    return leader, err
}
 
func (se *ScheduledElector) scheduledElection() {
    for {
        select {
            case <- se.ticker.C:
                if !se.paused {
                    se.Elect()
                } else {
                    log.WithFields(logrus.Fields{"id": se.Id()}).Warn("Scheduled election is paused")
                }
        }
    }
}
```
系统启动main方法
```Go
func main() {
    config.InitConfig()
    log.Printf("system instance is is %s", config.GetConfig().GetInstanceId())
    db.InitDatabase(config.GetConfig().Mysql)
    if config.GetConfig().IsHA() {
        elector = initElector()
    }
    go func() {
        service.InitJob(elector)
    }()
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":8080", nil)
    defer db.CloseDB()
}
 
func initElector() *election.Elector {
    log.Print("system current ha mode, elect leader")
    elector, err := election.NewMysqlElector(db.DB(), config.GetConfig().GetInstanceId(), cons.ElectionTaskName)
    if err != nil {
        log.Fatalf("Error while create mysql elector, error: %v", err)
    }
    scheduledElector := election.NewScheduledElector(elector)
    log.Printf("init elector successfully")
    return &scheduledElector.Elector
}
```
启动的时候根据参数进行判断，如果当前是HA部署了多个实例，则启动leader elect，否则单实例不用开启。

接下来执行具体的业务job逻辑
```Go
func InitJob(elector *election.Elector) {
    log.Info("System job init")
    id := config.GetConfig().GetInstanceId()
    start := time.Now()
    if isLeader(elector) {
        .....
    }
    log.WithField("cost", time.Since(start)).Info("xx job execute over")
    c := cron.New()
    c.AddFunc("@every 1m", func() {
        if isLeader(elector) {
            .....
        } else {
            log.WithFields(logrus.Fields{"id": id}).Error("Not leader")
        }
 
    })
    c.AddFunc("@every 5m", func() {
        if isLeader(elector) {
            ....
        } else {
            log.WithFields(logrus.Fields{"id": id}).Error("Not leader")
        }
    })
    c.Start()
}
 
func isLeader(elector *election.Elector) bool {
    if elector == nil {
        log.WithFields(logrus.Fields{"id": config.GetConfig().GetInstanceId()}).Info("Not ha mode")
        return true
    }
    leader, err := elector.IsLeader()
    if err != nil {
        log.WithFields(logrus.Fields{"id": elector.Id(), "err": err.Error()}).Error("Leader check failed")
        util.PromEventCount.WithLabelValues("IsLeader").Inc()
        return false
    }
    return leader
}
```