## 深入解读编解码算法
本文以go语言的编解码库实现为样例，展开分析几种常用的编解码算法。

### 概述

Go语言编解码库实现了ascii85、base32、base64、hex、binary等10+种编解码算法，本文将以go语言编解码库为样例，深入分析几种常用的编解码库算法。
下面将依次介绍ASCII类编码算法、二进制\16进制编码算法、pem证书编码、cvs编码。

### ASCII 编码类（ascii85,base32,base64)

Go语言支持ascii85（也叫base85），base32，base64等编码算法，主要用于二进制数据与可打印的ascii字符之间的编码转换。

#### 技术原理

ASCII85：包含85个可打印ASCII字符，使用5个ascii字符编码4个字节。对应到ASCII编码表，可见字符包括33（“！”）到117（“u”）。

算法核心代码如下，每次处理4个字节的二进制数据，然后对4字节的数据进行5次除以85取余数操作，余数+“！”得到编码后的ascii字符。

```go
// 拆分4个字节到uint32中，再重新包装到base85编码的5个字节中
var v uint32
switch len(src) {
default:
    v |= uint32(src[3])
    fallthrough
case 3:
    v |= uint32(src[2]) << 8
    fallthrough
case 2:
    v |= uint32(src[1]) << 16
    fallthrough
case 1:
    v |= uint32(src[0]) << 24
}

// 特殊情况，全零组编码为'z'而不是'!!!!!'
if v == 0 && len(src) >= 4 {
    dst[0] = 'z'
    dst = dst[1:]
    src = src[4:]
    n++
    continue
}
.
// 转化为base85编码的数据
for i := 4; i >= 0; i-- {
    dst[i] = '!' + byte(v%85) //取余数后+“！”使得编码后的ASCII字符在“！”到“u”之间
    v /= 85
}
```

base32：base32编码包含32个可打印ASCII字符，每个base32字符可用5个bit位表示（2^5=32），Go语言的base32算法实现了RFC
4648标准规范中，使用`encodeStd = "ABCDEFGHIJKLMNOPQRSTUVWXYZ234567"`
作为32个base32编码字符。

算法核心代码如下：每次编码5个字节的二进制数据 5*8 到8个ascii字符
，首先取src中最高位的数组的高5位与操作`0x1F`存储到b[7]，`enc.encode[b[7]&31]`与操作后映射到上面提到的ASCII字符集encodeStd，得到一个ASCII字符，以此类推向后编码。在对处理的中间数据做编码映射的时候，代码采用了循环展开的方式，提高编码性能。

```go
// 拆分5个8bit字节数据块到8个5-bit数据块中
switch len(src) {
    default:
    b[7] = src[4] & 0x1F
    b[6] = src[4] >> 5  // b[6]需要取src[4]的后3位 + src[3]的前2位
    fallthrough
    case 4:
    b[6] |= (src[3] << 3) & 0x1F
    b[5] = (src[3] >> 2) & 0x1F
    b[4] = src[3] >> 7  // b[4]需要取src[3]的后1位 + src[2]的前4位
    fallthrough
    case 3:
    b[4] |= (src[2] << 1) & 0x1F
    b[3] = (src[2] >> 4) & 0x1F // b[3]需要取src[2]的后4位 + src[1]的前1位
    fallthrough
    case 2:
    b[3] |= (src[1] << 4) & 0x1F
    b[2] = (src[1] >> 1) & 0x1F
    b[1] = (src[1] >> 6) & 0x1F // b[1]取src[1]的后2位 + src[1]的前3位
    fallthrough
    case 1:
    b[1] |= (src[0] << 2) & 0x1F
    b[0] = src[0] >> 3
}

// 使用base32 字母表编码5-bit数据块
size := len(dst)
if size >= 8 {
    // 循环展开，提高编码性能
    dst[0] = enc.encode[b[0]&31]
    dst[1] = enc.encode[b[1]&31]
    dst[2] = enc.encode[b[2]&31]
    dst[3] = enc.encode[b[3]&31]
    dst[4] = enc.encode[b[4]&31]
    dst[5] = enc.encode[b[5]&31]
    dst[6] = enc.encode[b[6]&31]
    dst[7] = enc.encode[b[7]&31]
} else {
    for i := 0; i < size; i++ {
        dst[i] = enc.encode[b[i]&31]
    }
}
```

base64：base64编码包含64个可打印ASCII字符，每个base64字符可用6个bit位表示（2^6=64），Go语言的base64算法实现了RFC
4648标准规范中，使用`encodeStd =
"ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"` 
作为编码的可打印ASCII字符。

算法核心代码如下：每次编码3个字节的二进制数据 3*8 到 4个ascii字符，首先把3个byte存储到一个unit变量中，然后每次取6位输出一个ascii字符，最终得到4个ascii字符。

```go
for si < n {
    // 转换3个字节的数据块（3*8-bit）到 4个字节的数据块（4*6-bit)
    val := uint(src[si+0])<<16 | uint(src[si+1])<<8 | uint(src[si+2])

    dst[di+0] = enc.encode[val>>18&0x3F]
    dst[di+1] = enc.encode[val>>12&0x3F]
    dst[di+2] = enc.encode[val>>6&0x3F]
    dst[di+3] = enc.encode[val&0x3F]

    si += 3
    di += 4
}
```

#### 应用

ascii85主要应用于Adobe的PostScript（文档的打印件） 、PDF文件格式、Git使用的二进制文件补丁编码等。

base32、base64的应用于处理文本数据，表示、传输、存储一些二进制数据等。


### 二进制/十六进制编码

Go语言支持二进制/十六进制编码算法，用于数字和字节序列之间的互相转换。

#### 技术原理

hex：十六进制编码算法编码每个字节为2个16进制字符，每个16进制字符用4个bit位表示，编码字符集为`hextable = "0123456789abcdef"`

算法核心代码如下：每次编码一个byte的数据为2个16进制符号，循环直到编码完成。

```go
func Encode(dst, src []byte) int {
	j := 0
	for _, v := range src { 
		//循环src源数组，每次编码一个byte为2个16进制符号，直到跳出循环
		dst[j] = hextable[v>>4]
		dst[j+1] = hextable[v&0x0f]
		j += 2
	}
	return len(src) * 2
}
```

binary：二进制包实现了数字与字节序列之间转换，支持bool, int8, uint8, int16, unit16,int32,unint32,int64,unint64,float32, float64的数据类型与字节序列[]byte的互相转换。

[小端序]([https://zh.wikipedia.org/wiki/%E5%AD%97%E8%8A%82%E5%BA%8F#%E5%B0%8F%E7%AB%AF%E5%BA%8F](https://zh.wikipedia.org/wiki/字节序#小端序)) uint64长度数据的编码转换代码如下： Uint64(b []byte)函数通过移位+或操作解码8byte字节序列到uint64类型的数据，PutUint64(b []byte, v uint64)函数编码unit64数字到8位byte数组中。

```go
func (littleEndian) Uint64(b []byte) uint64 {
	_ = b[7] // 边界检查
	return uint64(b[0]) | uint64(b[1])<<8 | uint64(b[2])<<16 | uint64(b[3])<<24 |
		uint64(b[4])<<32 | uint64(b[5])<<40 | uint64(b[6])<<48 | uint64(b[7])<<56 // 小端序低地址位存放低位字节
}

func (littleEndian) PutUint64(b []byte, v uint64) {
	_ = b[7] // 边界检查
	b[0] = byte(v)
	b[1] = byte(v >> 8)
	b[2] = byte(v >> 16)
	b[3] = byte(v >> 24)
	b[4] = byte(v >> 32)
	b[5] = byte(v >> 40)
	b[6] = byte(v >> 48)
	b[7] = byte(v >> 56)
}
```

#### 应用

二进制/十六进制编码广泛应用于计算机、通信数据传输、数字电路等领域。


### pem证书编码

#### 技术原理

pem：Go语言实现了pem数据编码算法，主要应用在TLS密钥和证书中。

Go语言中一个pem数据结构包括Type 、headers、Bytes3个字段，分别表示pem证书的类型、头部、内容。

```go
type Block struct {
	Type    string            // The type, taken from the preamble (i.e. "RSA PRIVATE KEY").
	Headers map[string]string // Optional headers.
	Bytes   []byte            // The decoded bytes of the contents. Typically a DER encoded ASN.1 structure.
}
```

Go语言的pem包实现了pem证书与Block数据结构之间的互相转换，pem.Decode方法解析pem数据到Block块，核心代码分解如下：

```go
func Decode(data []byte) (p *Block, rest []byte) {
    // 识别pem数据开始标志“-----BEGIN”
	rest = data
	if bytes.HasPrefix(data, pemStart[1:]) {
		rest = rest[len(pemStart)-1 : len(data)]
	} else if i := bytes.Index(data, pemStart); i >= 0 {
		rest = rest[i+len(pemStart) : len(data)]
	} else {
		return nil, data
	}

    // 获取pem数据类型
	typeLine, rest := getLine(rest)
	if !bytes.HasSuffix(typeLine, pemEndOfLine) {
		return decodeError(data, rest)
	}
	typeLine = typeLine[0 : len(typeLine)-len(pemEndOfLine)]

	p = &Block{
		Headers: make(map[string]string),
		Type:    string(typeLine), // 插入pem数据类型
	}
    
	// 获取数据数据的头部key-value
	for {
		if len(rest) == 0 {
			return nil, data
		}
		line, next := getLine(rest)

		i := bytes.IndexByte(line, ':')
		if i == -1 {
			break
		}

		key, val := line[:i], line[i+1:]
		key = bytes.TrimSpace(key)
		val = bytes.TrimSpace(val)
		p.Headers[string(key)] = string(val)
		rest = next
	}
    
    ......// 验证数据尾部"-----END"


    //使用base64编码pem数据内容，存储到block.Bytes数组中
	base64Data := removeSpacesAndTabs(rest[:endIndex])
	p.Bytes = make([]byte, base64.StdEncoding.DecodedLen(len(base64Data)))
	n, err := base64.StdEncoding.Decode(p.Bytes, base64Data)
	if err != nil {
		return decodeError(data, rest)
	}
	p.Bytes = p.Bytes[:n]
	_, rest = getLine(rest[endIndex+len(pemEnd)-1:])

	return
}
```
pem.Encode方法编码block数据块到pem数据，感兴趣可以自行查看源码 [encoding/pem/pem.go](https://github.com/golang/go/blob/master/src/encoding/pem/pem.go) 


### CSV编码介绍
CSV：Go语言的encoding/csv包用于读写逗号分隔值（comma-separated value,
[csv](https://en.wikipedia.org/wiki/Comma-separated_values)）
的数据，csv文件格式常应用于表格数据的交换 。cvs包实现了(r *Reader)
Read、(r *Reader) ReadAll、(r *Reader) Write、(r *Reader) WriteAll
方法用于处理逗号分隔值数据的读写。

样例程序如下：输入是逗号分隔值数据，使用csv.NewReader声明一个reader指针r，调用r.Read()读取每一行的数据输出到record 字符串数组中并打印。

```go
func ExampleReader() {
	in := `first_name,last_name,username
"Rob","Pike",rob
Ken,Thompson,ken
"Robert","Griesemer","gri"  // 输入的逗号分隔字符串
`
	r := csv.NewReader(strings.NewReader(in))

	for {
		// 使用r.Read()方法读取读取数据输出到字符串数组中
        record, err := r.Read() 
		if err == io.EOF {
			break
		}
		if err != nil {
			log.Fatal(err)
		}

		fmt.Println(record)
	}
	// Output:
	// [first_name last_name username]
	// [Rob Pike rob]
	// [Ken Thompson ken]
	// [Robert Griesemer gri]
}
```
csv包的读写方法调用底层bufio包的ReadSlice接口做数据读写，再对特殊符号做了处理，如逗号“，”，换行符 “\r\n”|"\n"，这里不展开分析，感兴趣可以自行查看源码 [encoding/csv](https://github.com/golang/go/tree/master/src/encoding/csv) 。
