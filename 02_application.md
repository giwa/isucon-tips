# Application

## Centos

### install

```
yum -y install epel-release
yum install -y ack
yum install -y graphviz
```

#### Redis
[Redis](https://github.com/giwa/isucon-tips/blob/master/04_redis.md)

#### go package

```
go get -u github.com/google/pprof

```

## Objective
Understand application structure and problems. Check followings.

- N+1 in view, serialize.
- Data insert and update.
- Data initialization in for loop.
- large data
- how to serve file
- multiple nest

Request

```
$ ack  --with-filename '\.(GET|POST|PUT|DELETE|PATCH)' app.go

app.go
738:    e.GET("/initialize", getInitialize)
739:    e.GET("/", getIndex)
740:    e.GET("/register", getRegister)
741:    e.POST("/register", postRegister)
742:    e.GET("/login", getLogin)
743:    e.POST("/login", postLogin)
744:    e.GET("/logout", getLogout)
746:    e.GET("/channel/:channel_id", getChannel)
747:    e.GET("/message", getMessage)
748:    e.POST("/message", postMessage)
749:    e.GET("/fetch", fetchUnread)
750:    e.GET("/history/:channel_id", getHistory)
752:    e.GET("/profile/:user_name", getProfile)
753:    e.POST("/profile", postProfile)
755:    e.GET("add_channel", getAddChannel)
756:    e.POST("add_channel", postAddChannel)
757:    e.GET("/icons/:file_name", getIcon)
```


DB
```
$ ack  --with-filename '(SELECT|UPDATE|DELETE|INSERT)' app.go

app.go
98:     if err := db.Get(&u, "SELECT * FROM user WHERE id = ?", userID); err != nil {
109:            "INSERT INTO message (channel_id, user_id, content, created_at) VALUES (?, ?, ?, NOW())",
127:    err := db.Select(&msgs, "SELECT * FROM message WHERE id > ? AND channel_id = ? ORDER BY id DESC LIMIT 100",
194:            "INSERT INTO user (name, salt, password, display_name, avatar_icon, created_at)"+
206:    db.MustExec("DELETE FROM user WHERE id > 1000")
207:    db.MustExec("DELETE FROM image WHERE id > 1001")
208:    db.MustExec("DELETE FROM channel WHERE id > 10")
209:    db.MustExec("DELETE FROM message WHERE id > 10000")
210:    db.MustExec("DELETE FROM haveread")
243:    err = db.Select(&channels, "SELECT * FROM channel ORDER BY id")
306:    err := db.Get(&user, "SELECT * FROM user WHERE name = ?", name)
355:    err := db.Get(&u, "SELECT name, display_name, avatar_icon FROM user WHERE id = ?",
400:            _, err := db.Exec("INSERT INTO haveread (user_id, channel_id, message_id, updated_at, created_at)"+
402:                    " ON DUPLICATE KEY UPDATE message_id = ?, updated_at = NOW()",
414:    err := db.Select(&res, "SELECT id FROM channel")
428:    err := db.Get(&h, "SELECT * FROM haveread WHERE user_id = ? AND channel_id = ?",
463:                            "SELECT COUNT(*) as cnt FROM message WHERE channel_id = ? AND ? < id",
467:                            "SELECT COUNT(*) as cnt FROM message WHERE channel_id = ?",
506:    err = db.Get(&cnt, "SELECT COUNT(*) as cnt FROM message WHERE channel_id = ?", chID)
520:            "SELECT * FROM message WHERE channel_id = ? ORDER BY id DESC LIMIT ? OFFSET ?",
536:    err = db.Select(&channels, "SELECT * FROM channel ORDER BY id")
558:    err = db.Select(&channels, "SELECT * FROM channel ORDER BY id")
565:    err = db.Get(&other, "SELECT * FROM user WHERE name = ?", userName)
589:    err = db.Select(&channels, "SELECT * FROM channel ORDER BY id")
614:            "INSERT INTO channel (name, description, updated_at, created_at) VALUES (?, ?, NOW(), NOW())",
665:            _, err := db.Exec("INSERT INTO image (name, data) VALUES (?, ?)", avatarName, avatarData)
669:            _, err = db.Exec("UPDATE user SET avatar_icon = ? WHERE id = ?", avatarName, self.ID)
676:            _, err := db.Exec("UPDATE user SET display_name = ? WHERE id = ?", name, self.ID)
688:    err := db.QueryRow("SELECT name, data FROM image WHERE name = ?",
```


functions
```
$ ack  --with-filename 'func ' app.go

app.go
41:func (r *Renderer) Render(w io.Writer, name string, data interface{}, c echo.Context) error {
45:func init() {
96:func getUser(userID int64) (*User, error) {
107:func addMessage(channelID, userID int64, content string) (int64, error) {
125:func queryMessages(chanID, lastID int64) ([]Message, error) {
132:func sessUserID(c echo.Context) int64 {
141:func sessSetUserID(c echo.Context, id int64) {
151:func ensureLogin(c echo.Context) (*User, error) {
179:func randomString(n int) string {
189:func register(name, password string) (int64, error) {
205:func getInitialize(c echo.Context) error {
214:func getIndex(c echo.Context) error {
233:func getChannel(c echo.Context) error {
263:func getRegister(c echo.Context) error {
271:func postRegister(c echo.Context) error {
290:func getLogin(c echo.Context) error {
298:func postLogin(c echo.Context) error {
321:func getLogout(c echo.Context) error {
328:func postMessage(c echo.Context) error {
353:func jsonifyMessage(m Message) (map[string]interface{}, error) {
369:func getMessage(c echo.Context) error {
412:func queryChannels() ([]int64, error) {
418:func queryHaveRead(userID, chID int64) (int64, error) {
439:func fetchUnread(c echo.Context) error {
482:func getHistory(c echo.Context) error {
551:func getProfile(c echo.Context) error {
582:func getAddChannel(c echo.Context) error {
601:func postAddChannel(c echo.Context) error {
624:func postProfile(c echo.Context) error {
685:func getIcon(c echo.Context) error {
711:func tAdd(a, b int64) int64 {
715:func tRange(a, b int64) []int64 {
723:func main() {
```



## Solution sample


### Stop db access
`https://github.com/giwa/isucon7-qualifier/blob/4b6be1607ab11dcc3b5f854eab5bdfaaf95719df/src/isubata/app.go#L688`

### fix loop query
`https://github.com/giwa/isucon7-qualifier/blob/4b6be1607ab11dcc3b5f854eab5bdfaaf95719df/src/isubata/app.go#L353-L367`

`https://github.com/giwa/isucon7-qualifier/blob/4b6be1607ab11dcc3b5f854eab5bdfaaf95719df/src/isubata/app.go#L390-L397`



## Profiling
### ppof
[Go言語のプロファイリングツール、pprofのWeb UIがめちゃくちゃ便利なので紹介する](https://medium.com/eureka-engineering/go%E8%A8%80%E8%AA%9E%E3%81%AE%E3%83%97%E3%83%AD%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AA%E3%83%B3%E3%82%B0%E3%83%84%E3%83%BC%E3%83%AB-pprof%E3%81%AEweb-ui%E3%81%8C%E3%82%81%E3%81%A1%E3%82%83%E3%81%8F%E3%81%A1%E3%82%83%E4%BE%BF%E5%88%A9%E3%81%AA%E3%81%AE%E3%81%A7%E7%B4%B9%E4%BB%8B%E3%81%99%E3%82%8B-6a34a489c9ee)

```
# Goがインストール済みなことを確認
$ go version
go version go1.9.2 darwin/amd64
# pprofをインストール（すでに入っていればアップデート）
$ go get -u github.com/google/pprof
# graphvizが必要なのでインストール
$ yum install -y graphviz
```

```
$ pprof -http=localhost:8080 ~/isucon7-qualify/webapp/go/src/isubata/  http://localhost:6060/debug/pprof/profile
[POINT]
上記はpprof/profileのエンドポイントは、CPUに関してのプロファイルとなりますが、エンドポイントを変えればメモリやブロック待ち時間など色々なプロファイリングが可能です。
http://localhost:6060/debug/pprof/heap
http://localhost:6060/debug/pprof/block
http://localhost:6060/debug/pprof/goroutine
http://localhost:6060/debug/pprof/threadcreate
http://localhost:6060/debug/pprof/mutex
```

- http://localhost:6060/debug/pprof/block?debug=1

```
--- contention:
cycles/second=2904003184
```

http://localhost:8080/ui/peek

```
 Type: cpu
Time: Sep 2, 2018 at 2:52pm (JST)
Duration: 30s, Total samples = 20ms (0.067%)
Showing nodes accounting for 20ms, 100% of 20ms total
----------------------------------------------------------+-------------
      flat  flat%   sum%        cum   cum%   calls calls% + context 	 	 
----------------------------------------------------------+-------------
                                              10ms 50.00% |   runtime.runqgrab /usr/local/Cellar/go/1.10.3/libexec/src/runtime/proc.go:4800
                                              10ms 50.00% |   runtime.sysmon /usr/local/Cellar/go/1.10.3/libexec/src/runtime/proc.go:4221
      20ms   100%   100%       20ms   100%                | runtime.usleep /usr/local/Cellar/go/1.10.3/libexec/src/runtime/sys_darwin_amd64.s:418
----------------------------------------------------------+-------------
                                              10ms   100% |   runtime.schedule /usr/local/Cellar/go/1.10.3/libexec/src/runtime/proc.go:2541
         0     0%   100%       10ms 50.00%                | runtime.findrunnable /usr/local/Cellar/go/1.10.3/libexec/src/runtime/proc.go:2286
                                              10ms   100% |   runtime.runqsteal /usr/local/Cellar/go/1.10.3/libexec/src/runtime/proc.go:4835
----------------------------------------------------------+-------------
         0     0%   100%       10ms 50.00%                | runtime.mcall /usr/local/Cellar/go/1.10.3/libexec/src/runtime/asm_amd64.s:351
                                              10ms   100% |   runtime.park_m /usr/local/Cellar/go/1.10.3/libexec/src/runtime/proc.go:2604
----------------------------------------------------------+-------------
         0     0%   100%       10ms 50.00%                | runtime.mstart /usr/local/Cellar/go/1.10.3/libexec/src/runtime/proc.go:1193
                                              10ms   100% |   runtime.mstart1 /usr/local/Cellar/go/1.10.3/libexec/src/runtime/proc.go:1227
----------------------------------------------------------+-------------
                                              10ms   100% |   runtime.mstart /usr/local/Cellar/go/1.10.3/libexec/src/runtime/proc.go:1193
         0     0%   100%       10ms 50.00%                | runtime.mstart1 /usr/local/Cellar/go/1.10.3/libexec/src/runtime/proc.go:1227
                                              10ms   100% |   runtime.sysmon /usr/local/Cellar/go/1.10.3/libexec/src/runtime/proc.go:4221
----------------------------------------------------------+-------------
                                              10ms   100% |   runtime.mcall /usr/local/Cellar/go/1.10.3/libexec/src/runtime/asm_amd64.s:351
         0     0%   100%       10ms 50.00%                | runtime.park_m /usr/local/Cellar/go/1.10.3/libexec/src/runtime/proc.go:2604
                                              10ms   100% |   runtime.schedule /usr/local/Cellar/go/1.10.3/libexec/src/runtime/proc.go:2541
----------------------------------------------------------+-------------
                                              10ms   100% |   runtime.runqsteal /usr/local/Cellar/go/1.10.3/libexec/src/runtime/proc.go:4835
         0     0%   100%       10ms 50.00%                | runtime.runqgrab /usr/local/Cellar/go/1.10.3/libexec/src/runtime/proc.go:4800
                                              10ms   100% |   runtime.usleep /usr/local/Cellar/go/1.10.3/libexec/src/runtime/sys_darwin_amd64.s:418
----------------------------------------------------------+-------------
                                              10ms   100% |   runtime.findrunnable /usr/local/Cellar/go/1.10.3/libexec/src/runtime/proc.go:2286
         0     0%   100%       10ms 50.00%                | runtime.runqsteal /usr/local/Cellar/go/1.10.3/libexec/src/runtime/proc.go:4835
                                              10ms   100% |   runtime.runqgrab /usr/local/Cellar/go/1.10.3/libexec/src/runtime/proc.go:4800
----------------------------------------------------------+-------------
                                              10ms   100% |   runtime.park_m /usr/local/Cellar/go/1.10.3/libexec/src/runtime/proc.go:2604
         0     0%   100%       10ms 50.00%                | runtime.schedule /usr/local/Cellar/go/1.10.3/libexec/src/runtime/proc.go:2541
                                              10ms   100% |   runtime.findrunnable /usr/local/Cellar/go/1.10.3/libexec/src/runtime/proc.go:2286
----------------------------------------------------------+-------------
                                              10ms   100% |   runtime.mstart1 /usr/local/Cellar/go/1.10.3/libexec/src/runtime/proc.go:1227
         0     0%   100%       10ms 50.00%                | runtime.sysmon /usr/local/Cellar/go/1.10.3/libexec/src/runtime/proc.go:4221
                                              10ms   100% |   runtime.usleep /usr/local/Cellar/go/1.10.3/libexec/src/runtime/sys_darwin_amd64.s:418
----------------------------------------------------------+-------------
```


## Logger
[graqt](https://github.com/serinuntius/graqt)

https://serinuntius.hatenablog.jp/entry/2018/05/01/083000


## N+1
message

https://github.com/ryotarai/isucon7q/commit/c98e4447f0f1ea822ebe64df2e33cfc86c6752b4#diff-65a7f662a18446a4877f858721cc010e

history

https://github.com/ryotarai/isucon7q/commit/c98e4447f0f1ea822ebe64df2e33cfc86c6752b4#diff-65a7f662a18446a4877f858721cc010e


