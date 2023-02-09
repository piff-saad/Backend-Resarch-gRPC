<div dir="rtl">

<h1>
مقدمه  gRPC در زبان GO.
</h1>

این تحقیق مقدمه اولیه برنامه نویس Go را برای کار با gRPC ارائه می دهد. با بررسی این مثال، یاد خواهید گرفت که چگونه:
1) یک سرویس را در یک فایل .proto تعریف کنید.
2) کد سرور و کلاینت را با استفاده از کامپایلر بافر پروتکل تولید کنید.
3) از Go gRPC API برای نوشتن یک سرویس گیرنده و سرور ساده برای سرویس خود استفاده کنید.

توجه داشته باشید که مثال در این آموزش از نسخه proto3 زبان بافر پروتکل استفاده می کند: می توانید در راهنمای زبان proto3 و راهنمای کد تولید شده Go اطلاعات بیشتری کسب کنید.

<h2>
چرا از gRPC استفاده میکنیم؟
</h2>

مثال ما یک برنامه ساده نقشه برداری روت است که به کلاینت ها امکان می دهد اطلاعاتی در مورد ویژگی های روت خود دریافت کنند، خلاصه ای از روت خود را ایجاد کنند و اطلاعات روت مانند به روز رسانی ترافیک را با سرور و سایر کلاینت ها مبادله کنند.
با gRPC می‌توانیم سرویس خود را یک بار در یک فایل .proto تعریف کنیم و کلاینت‌ها و سرورها را به هر یک از زبان‌های پشتیبانی‌شده gRPC تولید کنیم، که به نوبه خود می‌توانند در محیط‌هایی از سرورهای داخل یک مرکز داده بزرگ گرفته تا دستگاه تبلت خود اجرا شوند. ارتباط بین زبان ها و محیط های مختلف توسط gRPC برای شما انجام می شود. ما همچنین از تمام مزایای کار با بافرهای پروتکل، از جمله سریال‌سازی کارآمد، IDL ساده و به‌روزرسانی آسان رابط برخوردار هستیم.

<h2>
تعریف سرویس
</h2>

اولین قدم ما (همانطور که از مقدمه gRPC می دانید) تعریف سرویس gRPC و درخواست روش و انواع پاسخ با استفاده از بافرهای پروتکل است.
برای تعریف یک سرویس، یک سرویس با نام را در فایل .proto خود مشخص می کنید:

```go
service RouteGuide {
   ...
}
```

سپس متدهای rpc را در تعریف سرویس خود تعریف می کنید و نوع درخواست و پاسخ آنها را مشخص می کنید. gRPC به شما امکان می دهد چهار نوع متد سرویس را تعریف کنید که همه آنها در سرویس RouteGuide استفاده می شوند:
1)  یک RPC ساده که در آن کلاینت درخواستی را به سرور ارسال می‌کند و منتظر می‌ماند تا پاسخ بازگردد، درست مانند یک فراخوانی تابع عادی.

```go
// Obtains the feature at a given position.
rpc GetFeature(Point) returns (Feature) {}
```

2) یک RPC استریم سمت سرور که در آن کلاینت درخواستی را به سرور ارسال می کند و استریمی از داده برای خواندن دنباله ای از پیام ها دریافت می کند. کلاینت از استریم برگشتی می خواند تا زمانی که پیام دیگری وجود نداشته باشد. همانطور که در مثال ما می بینید، با قرار دادن کلمه کلیدی stream قبل از تایپ پاسخ، یک متد استریم سمت سرور را مشخص می کنید.

```go
// Obtains the Features available within the given Rectangle.  Results are
// streamed rather than returned at once (e.g. in a response message with a
// repeated field), as the rectangle may cover a large area and contain a
// huge number of features.
rpc ListFeatures(Rectangle) returns (stream Feature) {}
```


3) یک RPC استریم سمت کلاینت که در آن کلاینت دنباله ای از پیام ها را می نویسد و دوباره با استفاده از استریم ارائه شده به سرور ارسال می کند. هنگامی که کلاینت نوشتن پیام ها را به پایان رساند، منتظر می ماند تا سرور همه آنها را بخواند و پاسخ خود را برگرداند. شما با قرار دادن کلمه stream قبل از تایپ درخواست، یک متد استریم سمت کلاینت را مشخص می کنید.

```go
// Accepts a stream of Points on a route being traversed, returning a
// RouteSummary when traversal is completed.
rpc RecordRoute(stream Point) returns (RouteSummary) {}
```
4) یک استریم RPC دو طرفه که در آن هر دو طرف دنباله ای از پیام ها را با استفاده از استریم خواندن و نوشتن ارسال می کنند. این دو استریم به طور مستقل عمل می‌کنند، بنابراین کلاینت‌ها و سرورها می‌توانند به هر ترتیبی که دوست دارند بخوانند و بنویسند: برای مثال، سرور می‌تواند منتظر بماند تا تمام پیام‌های کلاینت را قبل از نوشتن پاسخ‌هایش دریافت کند، یا می‌تواند متناوب یک پیام را بخواند و سپس یک پیام بنویسد. یا ترکیب دیگری از خواندن و نوشتن. ترتیب پیام ها در هر استریم حفظ می شود. شما این نوع روش را با قرار دادن کلمه کلیدی stream قبل از درخواست و پاسخ مشخص می کنید.

```go
// Accepts a stream of RouteNotes sent while a route is being traversed,
// while receiving other RouteNotes (e.g. from other users).
rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}
```

فایل .proto ما همچنین حاوی تعاریف تایپ پیام بافر پروتکل برای همه انواع درخواست و پاسخ مورد استفاده در روش‌های سرویس ما است - برای مثال در اینجا آمده است:

```go
// Points are represented as latitude-longitude pairs in the E7 representation
// (degrees multiplied by 10**7 and rounded to the nearest integer).
// Latitudes should be in the range +/- 90 degrees and longitude should be in
// the range +/- 180 degrees (inclusive).
message Point {
  int32 latitude = 1;
  int32 longitude = 2;
}
```


<h1>
ساخت کد سرور و کلاینت
</h1>

در مرحله بعد باید رابط های سرویس گیرنده و سرور gRPC را از تعریف سرویس .proto خود تولید کنیم. ما این کار را با استفاده از پروتکل کامپایلر بافر protoc با پلاگین مخصوص gRPC Go انجام می دهیم.

<h2>
ساخت سرور
</h2>

ابتدا بیایید نحوه ایجاد سرور RouteGuide را در این مثال بررسی کنیم.
دو بخش در ایجاد سرور برای اینکه سرویس RouteGuide ما کار عملکرد خود را داشته باشد وجود دارد:
1) پیاده سازی رابط سرویس ایجاد شده از تعریف سرویس ما: انجام "کار" واقعی سرویس ما.
2) اجرای یک سرور gRPC برای گوش دادن به درخواست های کلاینت‌ ها و ارسال آنها به اجرای سرویس مناسب.

<h2>
پیاده سازی RouteGuide
</h2>

همانطور که می بینید، سرور ما دارای یک نوع ساختار routeGuideServer است که رابط RouteGuideServer تولید شده را پیاده سازی می کند:

```go
type routeGuideServer struct {
        ...
}
...

func (s *routeGuideServer) GetFeature(ctx context.Context, point *pb.Point) (*pb.Feature, error) {
        ...
}
...

func (s *routeGuideServer) ListFeatures(rect *pb.Rectangle, stream pb.RouteGuide_ListFeaturesServer) error {
        ...
}
...

func (s *routeGuideServer) RecordRoute(stream pb.RouteGuide_RecordRouteServer) error {
        ...
}
...

func (s *routeGuideServer) RouteChat(stream pb.RouteGuide_RouteChatServer) error {
        ...
}
...
```

<h2>
یک RPC ساده
</h2>

routeGuideServer تمام متد های خدمات ما را پیاده سازی می کند. بیایید ابتدا ساده ترین نوع، GetFeature را بررسی کنیم، که فقط یک پوینت از کلاینت را دریافت می کند و اطلاعات فیچر مربوطه را از پایگاه داده خود در یک فیچر به کلاینت برمی گرداند.

```go
func (s *routeGuideServer) GetFeature(ctx context.Context, point *pb.Point) (*pb.Feature, error) {
  for _, feature := range s.savedFeatures {
    if proto.Equal(feature.Location, point) {
      return feature, nil
    }
  }
  // No feature was found, return an unnamed feature
  return &pb.Feature{Location: point}, nil
}
```

این متد یک شی کانتکست برای RPC و درخواست بافر پروتکل Point کلاینت ارسال می شود. یک شی بافر پروتکل فیچر را با اطلاعات پاسخ و یک خطا برمی گرداند. در این متد، فیچر را با اطلاعات مناسب پر می‌کنیم، و سپس آن را همراه با خطای nil برمی‌گردانیم تا به gRPC بگوییم که کار با RPC تمام شده است و این فیچر می‌تواند به کلاینت بازگردانده شود.
جریان(استریم) RPC سمت سرور
حالا بیایید به یکی از RPC های جریانی خود نگاه کنیم. ListFeatures یک RPC استریم سمت سرور است، بنابراین باید چندین فیچر را به کلاینت خود بازگردانیم.

```go
func (s *routeGuideServer) ListFeatures(rect *pb.Rectangle, stream pb.RouteGuide_ListFeaturesServer) error {
  for _, feature := range s.savedFeatures {
    if inRange(feature.Location, rect) {
      if err := stream.Send(feature); err != nil {
        return err
      }
    }
  }
  return nil
}
```

همانطور که می بینید، به جای دریافت آبجکت های درخواست و پاسخ ساده در پارامترهای متد، این بار یک شی درخواست (مستطیلی که کلاینت ما می خواهد فیچر ها را در آن پیدا کند) و یک شی RouteGuide_ListFeaturesServer ویژه برای نوشتن پاسخ ها دریافت می کنیم.
در متد، ما هر تعداد شی Feature را که باید برگردانیم پر می کنیم و با استفاده از متد Send() آن را در RouteGuide_ListFeaturesServer می نویسیم. در نهایت، مانند RPC ساده ما، یک خطای nil برمی‌گردانیم تا به gRPC بگوییم که نوشتن پاسخ‌ها را به پایان رسانده‌ایم. اگر در این فراخوانی خطایی رخ دهد، یک خطای غیر nil را برمی‌گردانیم. لایه gRPC آن را به وضعیت RPC مناسب برای ارسال روی سیم ترجمه می کند.

ادامه بحث gRPC و جریان RPC سمت کلاینت در فایل مربوط به پارت دوم آمده است...

</div>
