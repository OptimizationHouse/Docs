# #**image package**

​      作用：image包实现了一个基本的2D图像库. 

~~~go
func NewNRGBA

func NewNRGBA(r Rectangle) *NRGBA
~~~

​       NewNRGBA返回一个新的带有给定边界的NRGBA图像。NRGBA是一个内存中的图像，它的At方法返回颜色的NRGBA值。

~~~go
type NRGBA struct {

    Pix []uint8

    Stride int

    Rect Rectangle

}

func (*NRGBA) Set

func (p *NRGBA) Set(x, y int, c color.Color)  //设定指定位置的color。

func Rect

func Rect(x0, y0, x1, y1 int) Rectangle  //Rect是Rectangle的缩写{Pt(x0, y0), Pt(x1, y1)}.
~~~

# #**Image/draw包用法:**

​        draw 包仅仅定义了一个操作：通过可选的蒙版图（mask image），把一个原始图片绘制到目标图片上，这个操作是出奇的灵活，可以优雅和高效的执行很多常见的图像处理任务，比如使用nil蒙版调用DrawMask，其代码如下：

~~~go
func Draw(dst Image, r image.Rectangle, src image.Image, sp image.Point, op Op)

func DrawMask(dst Image, r image.Rectangle, src image.Image, sp image.Point,

mask image.Image, mp image.Point, op Op)
~~~

​        第一个函数Draw是没有使用蒙版mask的调用方法，它内部其实就是调用的mask为 nil的方法，它的参数描述如下：

​        dst 是绘图的背景图，r是背景图的绘图区域，src 是要绘制的图，sp是src对应的绘图开始点(绘制的大小 r变量定义)，mask 是绘图时用的蒙版，控制替换图片的方式。mp是绘图时蒙版开始点（绘制的大小r变量定义了)

​        下图就是几个相关的例子：

​          mask 蒙版是渐变

​                  <img src="C:\Users\Daug\Desktop\image\image\images\1.png" alt="1" style="zoom:80%;" /> 

​          给一个矩形填充颜色:

​          使用 Draw方法的逻辑效果图：

​                  <img src="C:\Users\Daug\Desktop\image\image\images\2.png" alt="2" style="zoom:80%;" />    

​        代码：

~~~go
  m := image.NewRGBA(image.Rect(0, 0, 640, 480))

  blue := color.RGBA{0, 0, 255, 255}

  draw.Draw(m, m.Bounds(), &image.Uniform{blue}, image.ZP, draw.Src)
~~~

​        拷贝图片的一部分效果特效如下：

​                <img src="C:\Users\Daug\Desktop\image\image\images\3.png" alt="3"   />             

​        相关代码：

~~~go
  r := image.Rectangle{dp, dp.Add(sr.Size())}  // 获得更换区域

  draw.Draw(dst, r, src, sr.Min, draw.Src)
~~~

​       如果是复制整个图片，则更简单：

~~~go
  sr = src.Bounds()     // 获取要复制图片的尺寸

  r := sr.Sub(sr.Min).Add(dp)  // 目标图的要剪切区域

  draw.Draw(dst, r, src, sr.Min, draw.Src)
~~~

​       图片滚动效果如下图:

​                 ![4](C:\Users\Daug\Desktop\image\image\images\4.png) 

​      假设我们需要把图片 m 上移20个像素.

​      相关代码:

~~~go
  b := m.Bounds()

  p := image.Pt(0, 20)

  // 注意，尽管第二个参数是b，但由于剪切，有效矩形变小了。

  draw.Draw(m, b, m, b.Min.Add(p), draw.Src)

  dirtyRect := b.Intersect(image.Rect(b.Min.X, b.Max.Y-20, b.Max.X, b.Max.Y))
~~~

​       把一个图片转成RGBA格式，效果图:

​            <img src="C:\Users\Daug\Desktop\image\image\images\5.png" alt="5" style="zoom:80%;" /> 

​        相关代码:1

~~~go
  b := src.Bounds()

  m := image.NewRGBA(b)

  draw.Draw(m, b, src, b.Min, draw.Src)
~~~

​        通过蒙版画特效效果图

​                <img src="C:\Users\Daug\Desktop\image\image\images\6.png" alt="6" style="zoom:80%;" /> 

相关代码

~~~go
 type circle struct {

    p image.Point

    r int

  }

  func (c *circle) ColorModel() color.Model {

    return color.AlphaModel

  }

 func (c *circle) Bounds() image.Rectangle {

    return image.Rect(c.p.X-c.r, c.p.Y-c.r, c.p.X+c.r, c.p.Y+c.r)

  }

  

 func (c *circle) At(x, y int) color.Color {

    xx, yy, rr := float64(x-c.p.X)+0.5, float64(y-c.p.Y)+0.5, float64(c.r)

    if xx*xx+yy*yy < rr*rr {

      return color.Alpha{255}

    }

    return color.Alpha{0}

 } 

  draw.DrawMask(dst, dst.Bounds(), src, image.ZP, &circle{p, r}, image.ZP, draw.Over)
~~~

注意,一个image对象只需要实现下面几个就可,这也就是Go接口强大的地方.

~~~go
 type Image interface {

    // 返回图像的颜色模型。

    ColorModel() color.Model

    // Bounds返回一个可以返回非零颜色的域。

    // 边界不一定包含点(0,0)。

    Bounds() Rectangle

    // 返回(x, y)处像素的颜色。

    // At(Bounds().Min.X, Bounds().Min.Y) 返回网格左上角的像素的值。

    // At(Bounds().Max.X-1, Bounds().Max.Y-1) 返回右下角的像素的值。

    At(x, y int) color.Color

 }
~~~

​        画一个字体

​        效果图，画一个蓝色背景的字体。

​                 <img src="C:\Users\Daug\Desktop\image\image\images\7.png" alt="7" style="zoom:80%;" /> 

​        相关伪代码：

~~~go
 src := &image.Uniform{color.RGBA{0, 0, 255, 255}}

  mask := theGlyphImageForAFont()

  mr := theBoundsFor(glyphIndex)

  draw.DrawMask(dst, mr.Sub(mr.Min).Add(p), src, image.ZP, mask, mr.Min, draw.Over)
~~~

# #**image/color包用法：**

​        Color包实现了一个基本的颜色库：color是一个接口，它定义了可以被视为颜色的任何类型的最小方法集:可以转换为红色、绿色、蓝色和alpha值的颜色。转换可能是有损的，例如从CMYK或YCbCr颜色空间转换。其代码演示如下：

~~~go
type Color interface {

  RGBA() (r, g, b, a uint32)

}
~~~

​       RGBA返回字母预先乘上的红色、绿色、蓝色和颜色的alpha值。每个值的范围都在[0,0xFFFF]内，但是由uint32表示，因此乘以一个直到0xFFFF的混合因子不会溢出。

​       关于返回值有三个重要的点：首先，红色，绿色和蓝色是先预乘alpha：完全饱和的红色也是25%透明，由RGBA返回75%。第二，容器有16位:100%的红色是由RGBA返回一个65535的r的值，而不是255，所以从CMYK或YCbCr转换没有损耗。第三，返回的类型是uint32，将最大值设置为65535以保证两个值相乘不会溢出。

​        Image/color包还定义了许多实现color接口的具体类型。例如，RGBA是一个结构体，它代表了经典的“每个容器8位颜色”，代码如下:

~~~go
type RGBA struct {

  R, G, B, A uint8

}
~~~

​        不过要注意的是：RGBA的R字段是8位字符预乘范围[0,255]的颜色。RGBA通过将该值乘以0x101来满足颜色接口，从而在范围[0,65535]内生成一个16位的颜色。类似地，NRGBA结构类型表示一种8位的没有alpha预乘的颜色，就像PNG图像格式所使用的那样。当直接操作'NRGBA '的字段时，其值没有预乘alpha，但当调用RGBA方法时，返回值要预乘alpha。

# #**Image/palette包用法：**

​       调色板这个包为提供为图像库提供了一个标准的调色板。

Image/color/palette/gen.go：运行程序时会生成palette .go文件。

Image/color/palette/generate.go：在生成palette.go文件时会自动运行。

Image/color/palette/palette.go：定义了两种调色板——Plan9和WebSafe：

​       Plan9是一个256色的调色板，将24位RGB空间分割成4×4×4的细分，每个子立方体有4个阴影。与WebSafe相比，其想法是通过将颜色立方体切成更少的单元来降低颜色分辨率，并使用额外的空间来增加强度分辨率。这样就得到了16个灰色阴影(4个灰色子立方体，每个子立方体中有4个样本)，每种原色和副色的13个阴影(3个子立方体，4个样本加上黑色)，并合理地选择了覆盖颜色立方体其余部分的颜色。其优点是可以更好地表示连续色调。

​       WebSafe是一个216色的调色板，由于Netscape Navigator的早期版本而流行起来。它也被称为网景颜色多维数据集。

# #**Image/gif包用法：**

​       gif包实现了gif图片的解码及编码。

~~~go
func Decode(r io.Reader) (image.Image, error)   //Decode从r中读取一个GIF图像，然后返回的image.Image是第一个嵌入的图。

func DecodeConfig(r io.Reader) (image.Config, error)  //DecodeConfig不需要解码整个图像就可以返回全局的颜色模型和GIF图片的尺寸。

type Config struct {

  ColorModel   color.Model

  Width, Height int

}

//Config返回图像的颜色model和尺寸
func Encode(w io.Writer, m image.Image, o *Options) error  //将图片m按照gif模式写入w中 

type Options struct {

	// NumColors是图片中使用颜色的最大值，它的范围是1-256

	NumColors int

	// Quantizer经常被用来通过NumColors产生调色板，palette.Plan9 被用来替代nil Quantizer

	Quantizer draw.Quantizer

	// Drawer i用于将源图片转化为期望的调色板， draw.FloydSteinberg 用来替代一个空 Drawer.

	Drawer draw.Drawer

}
func EncodeAll(w io.Writer, g *GIF) error  //将图片按照帧与帧之间指定的循环次数和时延写入w中

type GIF struct {

  Image   []*image.Paletted // 连续的图片

  Delay   []int       // 连续的延迟时间，每一帧单位都是百分之一秒，delay中数值表示其两个图像动态展示的时间间隔

  LoopCount int        // 循环次数，如果为0则一直循环。

}

func DecodeAll(r io.Reader) (*GIF, error) //DecodeAll 从r上读取一个GIF图片，并且返回顺序的帧和时间信息
~~~

# #**image/jpeg包用法：**

​        jpeg包实现了jpeg图片的编码和解码。

~~~go
func Decode(r io.Reader) (image.Image, error)  //Decode读取一个jpeg文件，并将他作为image.Image返回

  func DecodeConfig(r io.Reader) (image.Config, error)  //无需解码整个图像，DecodeConfig变能够返回整个图像的尺寸和颜色（Config具体定义查看gif包中的定义）

  func Encode(w io.Writer, m image.Image, o *Options) error  //按照4:2:0的基准格式将image写入w中，如果options为空的话，则传递默认参数

type Options struct {
	Quality int
}//Options是编码参数，它的取值范围是1-100，值越高质量越好

type FormatError //用来报告一个输入不是有效的jpeg格式

type FormatError string

func (e FormatError) Error() string 

type Reader //不推荐使用Reader

type Reader interface {
	io.ByteReader
	io.Reader
}

type UnsupportedError 

func (e UnsupportedError) Error() string  //报告输入使用一个有效但是未实现的jpeg功能
~~~

# #**image/png包用法：**

​       image/png实现了png图像的编码和解码。

​       png和jpeg实现方法基本相同，都是对图像进行了编码和解码操作。

~~~go
func Decode(r io.Reader) (image.Image, error)   //Decode从r中读取一个图片，并返回一个image.image，返回image类型取决于png图片的内容

func DecodeConfig(r io.Reader) (image.Config, error)  //无需解码整个图像变能够获取整个图片的尺寸和颜色

func Encode(w io.Writer, m image.Image) error  //Encode将图片m以PNG的格式写到w中。任何图片都可以被编码，但是哪些不是image.NRGBA的图片编码可能是有损的。

type FormatError

func (e FormatError) Error() string     //FormatError会提示一个输入不是有效PNG的错误。

~~~

 由此可见，png和jpeg使用方法类似，只是两种不同的编码和解码方式。

# #**应用如下：**

## 1.生成图片：

~~~go
package main

import "image"import "image/color"import "image/png"import "os"

func main() {

  //生成一个100X50的图像。

  img := image.NewRGBA(image.Rect(0, 0, 100, 50))

  // 在 (2, 3)处画一个红点。

  img.Set(2, 3, color.RGBA{255, 0, 0, 255})

  // 保存到out.png

  f, _ := os.OpenFile("out.png", os.O_WRONLY|os.O_CREATE, 0600)

  defer f.Close()

  png.Encode(f, img)

}
~~~

运行结果，就是生成了一张png文件。

## 2.生成复杂色彩的图片：

~~~go
package main

import (

  "fmt"

  "image"

  "image/color"

  "image/png"

  "math"

  "os"

)

type Circle struct {

  X, Y, R float64

}

func (c *Circle) Brightness(x, y float64) uint8 {

  var dx, dy float64 = c.X - x, c.Y - y

  d := math.Sqrt(dx*dx+dy*dy) / c.R

  if d > 1 {

    return 0

  } else {

    return 255

  }

}

func main() {

  var w, h int = 280, 240

  var hw, hh float64 = float64(w / 2), float64(h / 2)

  r := 40.0

  θ := 2 * math.Pi / 3

  cr := &Circle{hw - r*math.Sin(0), hh - r*math.Cos(0), 60}

  cg := &Circle{hw - r*math.Sin(θ), hh - r*math.Cos(θ), 60}

  cb := &Circle{hw - r*math.Sin(-θ), hh - r*math.Cos(-θ), 60}

  m := image.NewRGBA(image.Rect(0, 0, w, h))

  for x := 0; x < w; x++ {

    for y := 0; y < h; y++ {

      c := color.RGBA{

        cr.Brightness(float64(x), float64(y)),

        cg.Brightness(float64(x), float64(y)),

        cb.Brightness(float64(x), float64(y)),

        255,

      }

      m.Set(x, y, c)

    }

  }

  f, err := os.OpenFile("rgb.png", os.O_WRONLY|os.O_CREATE, 0600)

  if err != nil {

    fmt.Println(err)

    return

  }

  defer f.Close()

  png.Encode(f, m)

}
~~~

运行结果： 
                    <img src="C:\Users\Daug\Desktop\image\image\images\8.png" alt="8" style="zoom:150%;" />

## 3.获取一张图片的尺寸

~~~go
package main

import (

  "fmt"

  "image"

  _ "image/jpeg"

  _ "image/png"

  "os"

)

func main() {

  width, height := getImageDimension("rgb.png")

  fmt.Println("Width:", width, "Height:", height)

}

func getImageDimension(imagePath string) (int, int) {

  file, err := os.Open(imagePath)

  defer file.Close()

  if err != nil {

    fmt.Fprintf(os.Stderr, "%v\n", err)

  }

 

  image, _, err := image.DecodeConfig(file)

  if err != nil {

    fmt.Fprintf(os.Stderr, "%s: %v\n", imagePath, err)

  }

  return image.Width, image.Height

}
~~~

