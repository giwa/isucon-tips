# Redis

## Objective
Understand redis installation and usage.


### install

```
# Ubuntu / Debian:
sudo apt-get install build-essential tcl wget

# RHEL-ish systems:
sudo yum groupinstall 'Development Tools'
sudo yum install tcl wget
```

```
wget http://download.redis.io/releases/redis-5.0-rc4.tar.gz

tar -xvf redis-5.0-rc4.tar.gz

cd redis-5.0-rc4
make
sudo make install
cd utils/
./install_server.sh
```

start

```
sudo systemctl start redis_6379

```

Enable
```
sudo systemctl enable redis_6379
```


### Cli

```
redis-cli
> set foo bar
> get foo
```

### Golang client

https://github.com/go-redis/redis

Setup

```
client := redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "", // no password set
		DB:       0,  // use default DB
	})
```

Get/set

```
// Get
val, err := client.Get("key").Result()

// Set
err := client.Set("key", "value", 0).Err()
```

Get Range

```
// GetRange(key string, start, end int64) *StringCmd
val, err := client.GetRange("key", 3, 5).Result()
```

HGet/HSet for structure

```
// HGet(key, field string) *StringCmd
val, err := client.HGet("key", "foo")

// HSet(key, field string, value interface{}) *BoolCmd
err := client.HSet("key", "foo", "val").Result()
```

How to store/retrieve list
// TODO: 実際に動かして確認