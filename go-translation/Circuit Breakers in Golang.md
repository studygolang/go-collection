# Go 中使用熔断器

当你看到 “熔断器” 这个术语时，你会想到什么呢?

![img](http://img.golang.space/shot-1626769887473.png)

从字面意思理解，使用锤子破坏了一个电路。

我们一般都会在自己家里安装熔断器，以阻止异常的电流从电网流向家里。在开始“微服务的熔断器”之前，让我们先看看它是如何工作的。

![img](http://img.golang.space/shot-1626769915898.png)

如上图所示，一个典型的熔断器装置有 2 个主要部件:

1. 用火线紧紧包裹的软铁芯
2. 触体。只要接触点能够形成一个连接点，电流就会从外部电源流向我们的房子。相反，如果连接断开，电流就停止流动。

当电流通过缠绕在软铁芯周围的导线时，软铁芯就像一块电磁铁，当流过它的电流高于预期的安培时，电磁铁就会变得强大到足以吸引邻近的触点，从而导致短路。

您一定在想，这与微服务架构有什么关系。在我看来，这是高度相关的，正如我们将要看到的！

## 微服务架构中的级联故障

微服务架构已经很好地取代了单体架构，但是为了使我们的系统具有高度的弹性，我们还需要解决一些关键问题。

微服务的一个问题是级联故障。举一个例子来更好地理解它。

![Cascading Failures 级联故障](http://img.golang.space/shot-1626769998089.png)

在上图中，参与者调用我们的主服务，它依赖于上游服务ー A，B，C。现在假定，服务 A 是一个读取量较大的系统，它依赖于数据库。这个数据库有其自身的局限性，并且在过载时，可能导致连接重置。这个问题不仅会影响服务 A 的性能，还会影响主服务，因为`goroutine`会继续等待它，从而记录线程池。

这就是人们所说的“一个坏苹果糟蹋了整个桶”，喝了糟糕的果酒的人肯定会有同感。下面让我们用一个例子来验证这一点。

![img](http://img.golang.space/shot-1626770071027.png)

让我们构建一个 Netflixisc 应用程序。其中一个微服务负责提供feed页面的**电影服务**。此服务还依赖于**推荐服务**为用户提供适当的推荐。

```go
// Recommendation Service
func main() {
	logGoroutines()
	http.HandleFunc("/recommendations", recoHandler)
	log.Fatal(http.ListenAndServe(":9090", nil))
}

func logGoroutines() {
	ticker := time.NewTicker(500 * time.Millisecond)
	done := make(chan bool)
	go func() {
		for {
			select {
			case <-done:
				return
			case t := <-ticker.C:
				fmt.Printf("\n%v - %v", t, runtime.NumGoroutine())
			}
		}
	}()
}

func recoHandler(w http.ResponseWriter, r *http.Request) {
	a := `{"movies": ["Few Angry Men", "Pride & Prejudice"]}`
	w.Write([]byte(a))
}
```

**推荐服务**暴露一个路由接口 `/recommendations`，它返回一个推荐电影列表，同时每 500 毫秒打印一次 goroutine 的数量。

```go
// Movies App
type MovieResponse struct {
	Feed           []string
	Recommendation []string
}

func main() {
	http.HandleFunc("/movies", fetchMoviesFeedHandler)
	log.Fatal(http.ListenAndServe(":8080", nil))
}

func fetchMoviesFeedHandler(w http.ResponseWriter, r *http.Request) {
	mr := MovieResponse{
		Feed: []string{"Transformers", "Fault in our stars", "The Old Boy"},
	}
	rms, err := fetchRecommendations()
	if err != nil {
		w.WriteHeader(500)
	}
	mr.Recommendation = rms
	bytes, err := json.Marshal(mr)
	if err != nil {
		w.WriteHeader(500)
	}
	w.Write(bytes)
}

func fetchRecommendations() ([]string, error) {
	resp, err := http.Get("http://localhost:9090/recommendations")
	if err != nil {
		return []string{}, err
	}
	defer resp.Body.Close()
	body, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		return []string{}, err
	}
	var mvsr map[string]interface{}
	err = json.Unmarshal(body, &mvsr)
	if err != nil {
		return []string{}, err
	}
	mvsb, err := json.Marshal(mvsr["movies"])
	if err != nil {
		return []string{}, err
	}
	var mvs []string
	err = json.Unmarshal(mvsb, &mvs)
	if err != nil {
		return []string{}, err
	}
	return mvs, nil
}
```

**电影服务**暴露一个路由 `/movies`，它返回电影列表和推荐列表。为了获取推荐，它反过来调用上游的**推荐服务**。

通过此设置，让我们以每秒 100 个请求的速率访问电影服务，持续 3 秒钟。在 99% 的毫秒范围内，我们可以获得 100% 的成功。这是预期的，因为只提供静态数据。

现在，假设推荐服务的响应时间过长，并在 `recoHandler` 添加20秒的等待时间，然后重新进行测试。成功率会下降，而响应时间也会开始受到影响。此外，在测试期间阻塞在推荐服务上的 goroutine 数量将急剧增加。

推荐服务的停工时间影响了终端用户，因为本来可以提供给他的电影feed列表都没有提供。这正是级联故障对我们的系统造成的影响。

## 电路熔断器的救援

熔断器是一个非常简单但相当重要的概念，因为它可以让我们保持服务的高可用性。熔断器有三种状态:

- **Closed State** 关闭状态

  ![img](http://img.golang.space/shot-1626770169827.png)

关闭状态是指数据通过的时候连接处关闭的状态。这是我们的理想状态，其中上游服务正如预期的那样工作。

- **Open State** 开放状态

  ![img](http://img.golang.space/shot-1626770191947.png)

打开状态指的是由于上游服务未按预期响应而导致电路短路的状态。这种短路可以避免上游服务在已经挣扎的情况下不堪重负。此外，下游服务的业务逻辑可以更快地获得上游可用性状态的反馈，而无需等待上游的响应。

- **Half Open State** 半开状态

  ![img](http://img.golang.space/shot-1626770217913.png)

如果电路是打开状态，我们希望它在上游服务再次可用时立即关闭它。虽然你可以通过手动干预来实现，但首选的方法应该是在电路最后一次打开，让一些请求延迟之后通过电路，

如果这些请求请求上游服务成功，我们就可以安全地接通电路。

![img](http://img.golang.space/shot-1626770248090.png)

另一方面，如果这些请求失败，电路仍然处于打开状态。

熔断器的状态图如下:

![img](http://img.golang.space/shot-1626770277464.png)

1. 如果电路是关闭的，当故障超过配置的阈值，则可以打开

2. 如果电路是打开的，在一段睡眠时间延迟后，部分打开

3. 如果电路是半开的，它可以

   + 再次打开，如果允许通过的请求也失败了

   + 关闭，如果允许通过的请求成功响应

## 熔断器在 Golang 的应用

虽然有多个库可供选择，但最常用的是 [hystix](https://github.com/afex/hystrix-go/hystrix)。正如文档建议的那样，hystrix 是 Netflix 设计的一个延迟和容错库，用于隔离远程系统、服务和第三方库的访问，阻止级联故障，并在不可避免的故障发生的复杂分布式系统中实现恢复能力。

Hystrix 熔断器的实现取决于以下配置:

1. 超时 ー 上游服务响应的等待时间
2. 最大并发请求 ー 上游服务允许调用的最大并发
3. 请求容量阈值 ー 在熔断之前的请求数，断路器在需要更改状态时无法评估的请求数量
4. 睡眠窗口 ー 开放状态与半开放状态之间的延迟时间
5. 误差百分比阈值ー电路短路时的误差百分比阈值

接下来让我们在电影和推荐示例中使用它，并在获取推荐时实现熔断器模式。

```go
var downstreamErrCount int
var circuitOpenErrCount int

func main() {
  	downstreamErrCount = 0
	circuitOpenErrCount = 0
	hystrix.ConfigureCommand("recommendation", hystrix.CommandConfig{
		Timeout: 100,
		RequestVolumeThreshold: 25,
		ErrorPercentThreshold:  5,
		SleepWindow:            1000,
	})
	http.HandleFunc("/movies", fetchMoviesFeedHandlerWithCircuitBreaker)
	log.Fatal(http.ListenAndServe(":8080", nil))
}

func fetchMoviesFeedHandlerWithCircuitBreaker(w http.ResponseWriter, r *http.Request) {
	mr := MovieResponse{
		Feed: []string{"Transformers", "Fault in our stars", "The Old Boy"},
	}

	// 
	output := make(chan bool, 1)
	errors := hystrix.Go("recommendation", func() error {
		// talk to other services
		rms, err := fetchRecommendations()
		if err != nil {
			return err
		}
		mr.Recommendation = rms
		output <- true
		return nil
	}, func(err error) error {
    	// 写你的fallback(回退)逻辑
		return nil
	})

	select {
	case err := <-errors:
		if err == hystrix.ErrCircuitOpen {
			circuitOpenErrCount = circuitOpenErrCount + 1
		} else {
			downstreamErrCount = downstreamErrCount + 1
		}

	case _ = <-output:

	}

	bytes, err := json.Marshal(mr)
	if err != nil {
		w.WriteHeader(500)
	}
	fmt.Printf("\ndownstreamErrCount=%d, circuitOpenErrCount=%d", downstreamErrCount, circuitOpenErrCount)
	w.Write(bytes)
}
```

使用 Hystrix，您还可以在电路打开时实现回退逻辑。这种逻辑可能因情况而异。如果电路打开，则从缓存中获取。

使用这个更新的逻辑，让我们尝试以每秒100个请求的速率重新攻击 3 秒钟。

哇! ！100% 的成功率，在打开的情况下，我们只提供 Feed 和返回 0个推荐。此外，由于每当电路短路，我们不再调用上游服务，因此推荐服务不会不堪重负，阻塞的 goroutine 数量不会像以前那么多。

## 想了解更多？

我的建议:

1. [关于 Netflix Hystrix](https://github.com/Netflix/Hystrix/wiki)
2. [Hystrix 是怎样工作的？](https://github.com/Netflix/Hystrix/wiki/How-it-Works)
3. [Hystrix bucketing](https://raw.githubusercontent.com/wiki/Netflix/Hystrix/images/circuit-breaker-1280.png)

![image](https://user-images.githubusercontent.com/17313948/127777325-3bed5e5a-57b0-4800-a817-c48f1c9f3523.png)



## 原文链接

[Circuit Breakers in Golang](https://ioshellboy.medium.com/circuit-breakers-in-golang-1779da9b001)


