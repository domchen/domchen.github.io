---
author: dom
date: 2012-11-25 07:31:49+00:00
layout: post
title: 压缩测试:LZMA vs ZLIB
id: 388
tags:
- LZMA
- 压缩
---

Flash Player11.4之后提供了LZMA的压缩支持，网上一直说这种压缩格式压缩率比ZLIB高很多，解压快很多。我感觉好像不太科学，还是自己测试下比较靠谱。测试的时候同时加上了天地会上给的as版LZMA解压类(可以用于低版本的FP)。<!-- more -->

    
    
    
    
    package
    {
    	import flash.display.Sprite;
    	import flash.utils.ByteArray;
    	import flash.utils.CompressionAlgorithm;
    	import flash.utils.getTimer;
    
    	import org.flexlite.domUtils.FileUtil;
    	import org.flexlite.domUtils.StringUtil;
    
    	/**
    	 * 
    	 * @author DOM
    	 */
    	public class CompressTest extends Sprite
    	{
    		public function CompressTest()
    		{
    			var bytes:ByteArray = FileUtil.openAsByteArray("doc.json");
    			var t:int = getTimer();
    			bytes.compress();
    			trace("zlib压缩:"+(getTimer()-t)+"ms"+" size:"+StringUtil.toSizeString(bytes.length,2));
    			t = getTimer();
    			bytes.uncompress();
    			trace("zlib解压:"+(getTimer()-t)+"ms");
    			t = getTimer();
    			bytes.compress(CompressionAlgorithm.LZMA);
    			trace("lzma压缩:"+(getTimer()-t)+"ms"+" size:"+StringUtil.toSizeString(bytes.length,2));
    			t = getTimer();
    			bytes.uncompress(CompressionAlgorithm.LZMA);
    			trace("lzma解压:"+(getTimer()-t)+"ms");
    			bytes.compress(CompressionAlgorithm.LZMA);
    			t = getTimer();
    			bytes = LZMA.decode(bytes);
    			trace("LZMA.AS解压:"+(getTimer()-t)+"ms");
    		}
    	}
    }


输出结果：

zlib压缩:255ms size:5.08MB
zlib解压:12ms
lzma压缩:1974ms size:5.11MB
lzma解压:399ms
LZMA.AS解压:27381ms

这结果真让人大失所望，不知道是不是测试的有问题，没有更小，反而更大了。而且解压时间长了几十倍。as版的LZMA解压时间更是无法接受。还是继续用zlib吧。
