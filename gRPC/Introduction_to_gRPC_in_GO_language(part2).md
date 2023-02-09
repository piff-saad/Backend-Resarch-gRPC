<div dir="rtl">
<h2>
جریان RPC سمت کلاینت:
</h2>
اکنون می خواهیم کمی دقیق تر به این موضوع بپردازیم:
"متد استریم سمت کلاینت RecordRoute، که در آن جریانی از اطلاعات را از کلاینت دریافت می‌کنیم و یک RouteSummary را با اطلاعاتی درباره آن‌ها برمی‌گردانیم."
 همانطور که می بینید، این بار متد اصلاً پارامتر درخواستی ندارد. در عوض، یک جریان RouteGuide_RecordRouteServer دریافت می‌کند، که سرور می‌تواند برای خواندن و نوشتن پیام‌ها از آن استفاده کند - می‌تواند پیام‌های کلاینت را با استفاده از متد Recv()  دریافت کند و با استفاده از متد SendAndClose()  پاسخ تکی خود را برگرداند.

```go
func (s *routeGuideServer) RecordRoute(stream pb.RouteGuide_RecordRouteServer) error {
  var pointCount, featureCount, distance int32
  var lastPoint *pb.Point
  startTime := time.Now()
  for {
    point, err := stream.Recv()
    if err == io.EOF {
      endTime := time.Now()
      return stream.SendAndClose(&pb.RouteSummary{
        PointCount:   pointCount,
        FeatureCount: featureCount,
        Distance:     distance,
        ElapsedTime:  int32(endTime.Sub(startTime).Seconds()),
      })
    }
    if err != nil {
      return err
    }
    pointCount++
    for _, feature := range s.savedFeatures {
      if proto.Equal(feature.Location, point) {
        featureCount++
      }
    }
    if lastPoint != nil {
      distance += calcDistance(lastPoint, point)
    }
    lastPoint = point
  }
}
```
در بدنه متد، از متد Recv() RouteGuide_RecordRouteServer برای خواندن مکرر درخواست‌های کلاینت استفاده می‌کنیم تا زمانی که دیگر پیامی وجود نداشته باشد: سرور باید خطای بازگشتی از Recv() را پس از هر فراخوانی بررسی کند. اگر همچنان this مقدارش صفر باشد، جریان همچنان خوب است و می تواند به خواندن ادامه دهد. اگر io.EOF باشد، جریان پیام به پایان رسیده است و سرور می تواند RouteSummary خود را برگرداند. اگر مقدار دیگری داشته باشد، خطای as is  را برمی گردانیم تا توسط لایه gRPC به وضعیت RPC ترجمه شود.

<h2>
جریان دو طرفه RPC:
</h2>
 
```go
func (s *routeGuideServer) RouteChat(stream pb.RouteGuide_RouteChatServer) error {
  for {
    in, err := stream.Recv()
    if err == io.EOF {
      return nil
    }
    if err != nil {
      return err
    }
    key := serialize(in.Location)
                ... // look for notes to be sent to client
    for _, note := range s.routeNotes[key] {
      if err := stream.Send(note); err != nil {
        return err
      }
    }
  }
}
```
این بار یک جریان RouteGuide_RouteChatServer دریافت می کنیم که، مانند مثال client-side streaming می تواند برای خواندن و نوشتن پیام ها استفاده شود. با این حال، این بار در حالی که کلاینت هنوز در حال نوشتن پیام در جریان پیام خود است، مقادیر را از طریق متد استریم برمی گردانیم.

سینتکس خواندن و نوشتن در اینجا بسیار شبیه به متد client-side streaming ما است، با این تفاوت که سرور به جای SendAndClose ()  از تابع Send()  استریم استفاده می کند زیرا چندین پاسخ را می نویسد. اگرچه هر طرف همیشه پیام های طرف مقابل را به ترتیبی که نوشته شده است دریافت می کند، هم مشتری و هم سرور می توانند به هر ترتیبی بخوانند و بنویسند - جریان ها کاملاً مستقل عمل می کنند.

 <h2>
راه اندازی سرور:
</h2>
هنگامی که همه متد‌های خود را پیاده‌سازی کردیم، همچنین باید یک سرور gRPC راه‌اندازی کنیم تا مشتریان بتوانند واقعاً از خدمات ما استفاده کنند. قطعه زیر نشان می دهد که چگونه این کار را برای سرویس RouteGuide خود انجام می دهیم:

```go
flag.Parse()
lis, err := net.Listen("tcp", fmt.Sprintf("localhost:%d", *port))
if err != nil {
  log.Fatalf("failed to listen: %v", err)
}
var opts []grpc.ServerOption
...
grpcServer := grpc.NewServer(opts...)
pb.RegisterRouteGuideServer(grpcServer, newServer())
grpcServer.Serve(lis)
```
 
 <h2>
ایجاد یک stub:
</h2>
برای فراخوانی متد های سرویس، ابتدا باید یک کانال gRPC برای ارتباط با سرور ایجاد کنیم. این را با ارسال آدرس سرور و شماره پورت به grpc.Dial() به صورت زیر ایجاد می کنیم:

```go
var opts []grpc.DialOption
...
conn, err := grpc.Dial(*serverAddr, opts...)
if err != nil {
  ...
}
defer conn.Close()
```
هنگامی که کانال gRPC راه اندازی شد، برای انجام RPC به یک کلاینت نیاز داریم. ما آن را با استفاده از متد NewRouteGuideClient ارائه شده توسط بسته pb تولید شده از مثال فایل .proto دریافت می کنیم.
```go
client := pb.NewRouteGuideClient(conn)
```

 <h1>
توابع و متد های سرویس فراخوانی:
</h1>

متد RPC ساده:

فراخوانی ساده RPC GetFeature تقریباً به آسانی فراخوانی یک متد محلی است.
```go
feature, err := client.GetFeature(context.Background(), &pb.Point{409146138, -746188906})
if err != nil {
  ...
}
```

متد RPC استریم سمت سرور:

در اینجا ما متد streaming سمت سرور را ListFeatures می نامیم که جریانی از ویژگی های جغرافیایی را برمی گرداند.
```go
rect := &pb.Rectangle{ ... }  // initialize a pb.Rectangle
stream, err := client.ListFeatures(context.Background(), rect)
if err != nil {
  ...
}
for {
    feature, err := stream.Recv()
    if err == io.EOF {
        break
    }
    if err != nil {
        log.Fatalf("%v.ListFeatures(_) = _, %v", client, err)
    }
    log.Println(feature)
}
```
ما از متد RouteGuide_ListFeaturesClient's Recv() برای خواندن مکرر پاسخ های سرور به یک شی بافر پروتکل پاسخ (در این مورد یک ویژگی) استفاده می کنیم تا زمانی که دیگر پیامی وجود نداشته باشد: کلاینت باید خطای بازگشتی از Recv()  را بعد از هر فراخوانی بررسی کند. اگر خطایی نبود، جریان همچنان خوب است و می تواند به خواندن ادامه دهد. اگر io.EOF باشد، جریان پیام به پایان رسیده است. در غیر این صورت باید یک خطای RPC وجود داشته باشد که از طریق err منتقل می شود.


جریان دو طرفه RPC:

نهایتا به جریان دو طرفه RPC RouteChat ()  نگاه می کنیم. همانطور که در مورد RecordRoute، ما فقط به متد یک متن پاس می دهیم و جریانی را دریافت می کنیم که می توانیم از آن برای نوشتن و خواندن پیام ها استفاده کنیم. با این حال، این بار در حالی که سرور هنوز در حال نوشتن پیام به جریان پیام خود است، مقادیر را از طریق متد استریم خود برمی گردانیم.
```go
stream, err := client.RouteChat(context.Background())
waitc := make(chan struct{})
go func() {
  for {
    in, err := stream.Recv()
    if err == io.EOF {
      // read done.
      close(waitc)
      return
    }
    if err != nil {
      log.Fatalf("Failed to receive a note : %v", err)
    }
    log.Printf("Got message %s at point(%d, %d)", in.Message, in.Location.Latitude, in.Location.Longitude)
  }
}()
for _, note := range notes {
  if err := stream.Send(note); err != nil {
    log.Fatalf("Failed to send a note: %v", err)
  }
}
stream.CloseSend()
<-waitc
```

سینتکس برای خواندن و نوشتن در اینجا بسیار شبیه به متد استریم سمت سرویس گیرنده ما است، با این تفاوت که وقتی فرخوانی خود را به پایان رساندیم، از متد ()CloseSend استریم استفاده می کنیم. اگرچه هر طرف همیشه پیام های طرف مقابل را به ترتیبی که نوشته شده دریافت می کند، هم مشتری و هم سرور می توانند به هر ترتیبی بخوانند و بنویسند - جریان ها کاملاً مستقل عمل می کنند.
 
</dv>
