---
title: Golang系列(35)-GZIP压缩/解压
date: 2022-11-29 22:28:28
tags: golang
---

在网络传输过程中，如果遇到体量较大的数据，我们一般采用压缩算法，实现减少带宽占用的做法。

## GZIP
>gzip是GNUzip的缩写，最早用于UNIX系统的文件压缩。
GZIP 的核心是 Deflate,Deflate 是一个同时使用 LZ77 与 Huffman Coding 的算法。

## 原理
gzip 对于要压缩的文件，首先使用LZ77算法的一个变种进行压缩，对得到的结果再使用Huffman编码的方法进行压缩。
### LZ77
核心思路是如果一个串中有两个重复的串，那么只需要知道第一个串的内容和后面串相对于第一个串起始位置的距离 + 串的长度。

比如： ABCDEFGABCDEFH → ABCDEFG(7,6)H。7 指的是往前第 7 个数开始，6 指的是重复串的长度，ABCDEFG(7,6)H 完全可以表示前面的串，并且是没有二义性的。

### Huffman
核心思路是通过构造 Huffman Tree 的方式给字符重新编码（核心是避免一个叶子的路径是另外一个叶子路径的前缀），以保证出现频路越高的字符占用的字节越少。

## 实现
下面介绍的是在golang里如何使用gzip实现解压缩操作
<!--more-->

```go
package compressor
// CompressorProvider describes a component that can provider compressors for the std methods.
type CompressorProvider interface {
	// AcquireGzipWriter Returns a *gzip.Writer which needs to be released later.
	// Before using it, call Reset().
	AcquireGzipWriter() *gzip.Writer

	// ReleaseGzipWriter Releases an acquired *gzip.Writer.
	ReleaseGzipWriter(w *gzip.Writer)

	// AcquireGzipReader Returns a *gzip.Reader which needs to be released later.
	AcquireGzipReader() *gzip.Reader

	// ReleaseGzipReader Releases an acquired *gzip.Reader.
	ReleaseGzipReader(r *gzip.Reader)
}

// SyncPoolCompressors 基于sync.Pool实现对象池，减少GC压力
type SyncPoolCompressors struct {
	writerPool *sync.Pool
	readerPool *sync.Pool
}

func NewSyncPoolCompressors() CompressorProvider {
	return &SyncPoolCompressors{
		readerPool: &sync.Pool{
			New: func() any {
				return new(gzip.Reader)
			}},
		writerPool: &sync.Pool{
			New: func() any {
				return new(gzip.Writer)
			}},
	}
}

func (s *SyncPoolCompressors) AcquireGzipWriter() *gzip.Writer {
	return s.writerPool.Get().(*gzip.Writer)
}

func (s *SyncPoolCompressors) ReleaseGzipWriter(w *gzip.Writer) {
	s.writerPool.Put(w)
}

func (s *SyncPoolCompressors) AcquireGzipReader() *gzip.Reader {
	return s.readerPool.Get().(*gzip.Reader)
}

func (s *SyncPoolCompressors) ReleaseGzipReader(r *gzip.Reader) {
	s.readerPool.Put(r)
}

type GzipCompressor struct {
	provider CompressorProvider
}

func (c *GzipCompressor) Compress(dst io.Writer, src []byte) (err error) {
	// not compress empty bytes.
	if len(src) == 0 {
		return
	}

	w := c.provider.AcquireGzipWriter()
	w.Reset(dst)
	defer c.provider.ReleaseGzipWriter(w)

	return runtime.Call(func() error {
		_, err := w.Write(src)
		return err
	}, w.Flush, w.Close)
}

func (c *GzipCompressor) Decompress(dst io.Writer, src []byte) (err error) {
	r := c.provider.AcquireGzipReader()
	defer c.provider.ReleaseGzipReader(r)

	if err = r.Reset(bytes.NewReader(src)); err != nil {
		return
	}
	_, err = io.Copy(dst, r)
	return
}

func New() *GzipCompressor {
	return &GzipCompressor{provider: NewSyncPoolCompressors()}
}
```

测试如下：
```go
package compressor

func TestCompress(t *testing.T) {
	source := []byte("hello,world")
	input := bytes.NewBuffer([]byte{})

	// 压缩数据
	err := Compress(input, source)
	assert.Nil(t, err)

	// 压缩成功后解压缩
	output := bytes.NewBuffer([]byte{})
	err = Decompress(output, input.Bytes())
	assert.Nil(t, err)

	// 对比结果
	assert.Equal(t, source, output.Bytes())
}
```

