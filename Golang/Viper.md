[toc]

# Viper

```shell
go get github.com/spf13/viper
```

## 1. 从配置文件读取

```golang
viper.SetConfigName("config")
viper.SetConfigFile("config.yaml")
viper.SetConfigType("yaml")
viper.AddConfigPath("./viper")
if err := viper.ReadInConfig(); err != nil {
	panic(fmt.Errorf("Fatal error config file: %w \n", err))
}
```

> 如果要同时读取多个配置文件, 可以使用 `viper.New()` 创建多个不同的对象来读取

**使用 `SetConfigName` 方法**, 回写配置文件需要这个用这个方法

```golang
v := viper.New()
v.SetConfigName("config")					// 设置配置文件名
v.SetConfigType("yaml")						// 设置配置文件类型
v.AddConfigPath("./config")					// 设置配置文件路径 (绝对路径|相对路径|$环境变量), 可以添加多个路径
v.AddConfigPath(".")
if err := v.ReadInConfig(); err != nil {	// 读取配置文件
	panic(fmt.Errorf("Fatal error config file: %w \n", err))
}
```

**使用 `SetConfigFile` 方法**

```golang
v := viper.New()
v.SetConfigFile("./config/config.yaml")		// 指向配置文件的路径
v.SetConfigType("yaml")
if err := v.ReadInConfig(); err != nil {
	panic(fmt.Errorf("Fatal error config file: %w \n", err))
}
```

> 读取 ini 格式配置文件

```
v := viper.New()
v.SetConfigFile("./config/config.ini")
v.SetConfigType("ini")
if err := v.ReadInConfig(); err != nil {
	panic(fmt.Errorf("Fatal error config file: %w \n", err))
}
```

## 2. 读取配置值

- **Get(key string)** : interface{}
- **GetBool(key string)** : bool
- **GetFloat64(key string)** : float64
- **GetInt(key string)** : int
- **GetIntSlice(key string)** : []int
- **GetString(key string)** : string
- **GetStringMap(key string)** : map[string]interface{}
- **GetStringMapString(key string)** : map[string]string
- **GetStringSlice(key string)** : []string
- **GetTime(key string)** : time.Time
- **GetDuration(key string)** : time.Duration
- **IsSet(key string)** : bool
- **AllSettings()** : map[string]interface{}

```yaml
# MySQL配置
DB:
  HOST: 127.0.0.1
  PORT: 3306
  USER: root
  PASSWORD: admin@123
  DBNAME: test
```

```golang
fmt.Printf("DB.NAME: %t\n", v.Get("DB.DBNAME"))		// %!t(string=test)
fmt.Printf("DB.HOST: %t\n", v.GetString("DB.HOST"))	// %!t(string=192.168.88.134)
fmt.Printf("DB.PORT: %t\n", v.GetInt("DB.PORT"))	// %!t(int=3306)
fmt.Println("DB.USER: ", v.IsSet("DB.USER"))		// true
fmt.Printf("config: %t\n", v.AllSettings())
// map[db:map[dbname:test host:192.168.88.134 password:admin@123 port:3306 user:root] redis:map[host:192.168.88.134 password: port:6379]]
```



## 3. 从环境变量读取

**AutomaticEnv**

`AutomaticEnv` 会自动检查系统环境变量, 把发现的环境变量加载到viper

```goalng
v := viper.New()
v.AutomaticEnv()
fmt.Println(v.Get("GOPATH"))
fmt.Println(v.Get("ENV"))
```

**BindEnv**

> `BindEnv` 能够把viper键和系统环境变量绑定在一起

```golang
v := viper.New()
if err := v.BindEnv("redis.port"); err != nil {			// 把系统环境变量中的 redis.port 绑定到 viper 的 redis.port 键上
	fmt.Println(err)
}
if err := v.BindEnv("go.path", "GOROOT"); err != nil {	// 把系统环境变量的 GOROOT 绑定到 viper 的 go.path 键上
	fmt.Println(err)
}
fmt.Println(v.Get("redis.port"))
fmt.Println(v.Get("go.path"))
```

**SetEnvPrefix**

SetEnvPrefix 能够设置读取的环境变量名的前缀, 在GET时, 自动给key加上前缀

```golang
_ = os.Setenv("id", "13")
_ = os.Setenv("dev_id", "15")
v := viper.New()
v.SetEnvPrefix("dev")
v.AutomaticEnv()
fmt.Println(v.Get("id"))	// 15
```



## 4. 设置/修改配置, 写入配置文件

```golang
v := viper.New()
// 设置默认值
v.SetDefault("USER.HOME_DIR", "/root")
fmt.Println(v.Get("USER.HOME_DIR"))
// 设置/修改值
v.Set("USER.HOME_DIR", "/home")
fmt.Println(v.Get("USER.HOME_DIR"))
v.Set("USER.ENV", "dev")
fmt.Println(v.Get("USER.ENV"))

// 写入配置文件
v.SetConfigType("yaml")
if err := v.WriteConfigAs("./user.yaml"); err != nil {
	panic(fmt.Errorf("Write Config Error: %v\n", err))
}
```

> 写入模式

- **WriteConfig** 
    - 把当前 viper 配置覆盖写入到 **已存在的**`预定义文件`
    - 如果文件不存在则 `panic` *(Config File Not Found)*
    - **`预定义文件`**由 `AddConfigPath`和 `SetConfigName` 决定
- **SafeWriteConfig** 
    - 把当前 viper 配置写入到 **不存在的** `预定义的文件`
    - 如果文件存在, 则`panic` *(Config File Already Exists)*
- **WriteConfigAs** 
    - 把当前 viper 配置覆盖写入到指定文件
- **SafeWriteConfigAs** 
    - 把当前 viper 配置写入到 **不存在的**指定文件
    - 如果文件存在, 则`panic` *(Config File Already Exists)*

> Demo 修改文件内容

```
# MySQL配置
DB:
  USER: root
  PASSWORD: admin@123
```

```
v := viper.New()
v.SetConfigName("user")
v.SetConfigType("yaml")
v.AddConfigPath(".")
// 读取配置文件
if err:= v.ReadInConfig(); err != nil {
	panic(fmt.Errorf("Fatal error config file: %w \n", err))
}
v.Set("db.user", "admin")

// 写入配置文件
if err := v.WriteConfig(); err != nil {
	panic(fmt.Errorf("Write Config Error: %v\n", err))
}
```

```
db:
  password: admin@123
  user: admin
```

## 5. 配置读取优先级

viper 环境变量读取优先级如下, 由高到低:

- 显示调用Set方法, 设置的配置
- flag 模块带来的命令行参数
- 环境变量
- 配置文件
- key/value 存储
- default 默认值

> 配置文件 优先级高于 默认值

```golang
v := viper.New()
v.SetDefault("DB.DBNAME", "default")	// DB.DBNAME = default
v.SetConfigFile("./viper/config.yaml")	// DB.DBNAME = config
v.SetConfigType("yaml")
_ = v.ReadInConfig()
fmt.Println(v.Get("DB.DBNAME"))			// 显示: config
```

> Set方法 优先级高于 配置文件

```golang
v := viper.New()
v.Set("DB.DBNAME", "set")             	// DB.DBNAME = set
v.SetConfigFile("./viper/config.yaml") 	// DB.DBNAME = config
v.SetConfigType("yaml")
_ = v.ReadInConfig()
fmt.Println(v.Get("DB.DBNAME")) 		// 显示: set
```

