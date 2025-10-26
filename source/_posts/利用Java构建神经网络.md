---
title: 利用Java构建神经网络
date: 2016-12-19 20:11:42
tags:
---

## 简介

神经网络模型也被称为人工神经网络(Artificial Neural Network,ANN),它是一类非线性模型，用来在给定输入变量集合的情况下，对输出进行预测。其最早的例子可以追溯到20世纪40年代早期，当时用它来解决分类问题。可以说，它是一类非线性的回归方法。最近几年在深度学习中应用很广泛，本篇文章抱着学习其原理的心态，参考书籍，力图用Java语言实现一个神经网络。

由于网上有太多介绍神经网络的文章，这里对它的原理这种关键点描述，不在展开他们的细节，直接探究其实现方法，如果不了解这些知识点请自行Google。
* 多层前向传播网络
* 激活函数
* 反向传播算法(Backpropagation Algorithm)

<!-- more -->

## 建立ANN

### 构造激活函数
这里主要使用Simgod和tanh函数，主要是因为他们的范围可控，并且导数易求(神经网络中的导数往往要简单到可以由f(x)求得，主要是因为f(x)的值常常被记录下来，利用将极大提高效率)。也可以自己实现一个激活函数。


```java
package com.chengsluo.ann;
//将加权和映射到某个固定范围，如[-1,1];
public abstract class Activation {
	//激活函数
	public abstract double f(double x);
	//激活函数的导数，这是在反向传播过程中必须要用到的。
	public abstract double df(double x);

	//后面的一部分是Sigmod函数的具体实现

	//sigmod(k,x)函数的实现1/(1+e^(-k*x))，其中k控制函数值从正变为负的变化速率
	//可以推到其导数为k*f(x)*(1-f(x))
	public static Activation logit(final double k) {
		return new Activation() {
			@Override
			public double f(double x) {
				return 1.0/(1.0 + Math.exp(-k*x));
			}

			@Override
			public double df(double x) {
				x = f(x);
				return k*x*(1.0-x);
			}
		};
	}
	public static Activation sigmoid(double k) {
		return logit(k);
	}
	//默认k=1.0
	public static Activation logit() {
		return logit(1.0);
	}
	public static Activation sigmoid() {
		return logit();
	}
	//双曲正切函数
	public static Activation tanh() {
		return new Activation() {
			@Override
			public double f(double x) {
				return Math.tanh(x);
			}
			//(tanh(x))'=1-tanh(x)^2;
			@Override
			public double df(double x) {
				x = f(x);
				return 1.0 - x*x;
			}
		};
	}
}
```

### 构建单层

这里在构建网络层的时候，为了简化起见，给每一个层配置了相同的激活函数，这常常是默认的做法。

```java
package com.chengsluo.ann;
import java.util.Random;
public class Layer {

	//打印方法
	public static String printVec(double[] x) {
		StringBuilder sb = new StringBuilder();
		sb.append("{");
		for(int i=0;i<x.length;i++) { if(i > 0) sb.append(",");sb.append(x[i]); }
		sb.append("}");
		return sb.toString();
	}

	double[]   v;
	double[][] w;
	double[]   bW = null;

	Activation fn;

	//构造函数
	/**
	 * @param units 本层节点个数
	 * @param inputs 与每个节点连接的输入节点的个数
	 * @param fn 本层所用的激活函数
	 * @param bias 是否使用bias
	 *
	 * */
	public Layer(int units,int inputs,Activation fn,boolean bias) {
		this.fn = fn;
		v   = new double[units];
		w = new double[units][];
		for(int i=0;i<v.length;i++) w[i] = new double[inputs];
		if(bias)
			bW = new double[units];
		err = new double[units];
	}
	//返回本层网络节点数
	public int units() { return v.length; }

	public double[] value()  { return v; }
	double[]   err;
	//记录反向传播时，本层节点的所有误差的平方和
	double     E;

	public double[] errors() { return err; }

	public double[] feed(double[] x) {
		for(int i=0;i<v.length;i++) {
			double[] W = w[i];
			v[i] = bW != null ? bW[i] : 0.0;
			for(int j=0;j<W.length;j++)
				v[i] += W[j]*x[j];
			//将当前节点值更新为权重乘以对应输入的总和.
			//并用激活函数归一化
			v[i] = fn.f(v[i]);
		}
		//这里的v[i]是上一层节点共同作用的结果。影响率为w[j].
		return v;
	}

	public double[] backprop(double[] s) {
		double[] out = new double[w[0].length];
		E = 0;
		for(int i=0;i<v.length;i++) {
			err[i] = fn.df(v[i])*s[i];
			//期望值与真实值的偏差->s[i]与此点的导数的乘积
			//同等偏差情况下，导数越大，误差err越大
			E += err[i]*err[i];
			double[] W = w[i];
			//根据前一层节点对本节点的贡献率来给前一层节点的对应位置制造反向传播的输入
			for(int j=0;j<W.length;j++) out[j] += W[j]*err[i];
		}
		//这里的out[i]是本层所有节点共同作用的结果
		return out;
	}
	/* backprop与feed不同的地方是：
	 *
	 * feed传入的是函数值，输出的也是函数值
	 * backprop传入的是偏差值，但会经过df处理，
	 * 按照复合函数的链式法则，处理后的结果仍然是内层函数的偏差值，
	 * 所以可以继续传递到前一层。
	 *
	 * */

	public double error() { return E; }

	public double[] update(double[] o,double r) {
		for(int i=0;i<v.length;i++) {
			if(bW != null)
				bW[i] += r*err[i];

			double[] W = w[i];
			for(int j=0;j<W.length;j++)
				W[j] += r*err[i]*o[j];
			//权重的增加值为:上一层的输出与连接位的对应误差err再乘以学习率的乘积的和。
		}
		//误差越大权重每轮增加越大，学习率越大，权重增加越快。
		return v;
	}

	//随机构建初始值[-1,1]
	public void initialize(Random rng) {
		for(int i=0;i<v.length;i++) {
			for(int j=0;j<w[i].length;j++)
				w[i][j] = 2*rng.nextDouble() - 1;
			bW[i] = 2*rng.nextDouble() - 1;
		}

	}

	public void initialize() {
		initialize(new Random());
	}


}
```

### 组织成网络

```java
package com.chengsluo.ann;

import java.util.ArrayList;

public class NeuralNetwork {

	int inputUnits = 0;
	ArrayList<Layer> layers = new ArrayList<Layer>();
	double learningRate = 0.2;

	public NeuralNetwork learningRate(double l) {
		learningRate = l;
		return this;
	}

	public NeuralNetwork inputs(int inputUnits) {
		this.inputUnits = inputUnits;
		return this;
	}

	public NeuralNetwork momentumLayer(int units, Activation fn, boolean bias) {
		int inputs = (layers.size() == 0) ? this.inputUnits : layers.get(layers.size() - 1).units();
		layers.add(new MomentumLayer(units, inputs, fn, bias));
		return this;
	}

	//默认bias为空
	public NeuralNetwork momentumLayer(int units, Activation fn) {
		return momentumLayer(units, fn, true);
	}

	//默认激活函数为tanh();
	public NeuralNetwork momentumLayer(int units) {
		return momentumLayer(units, Activation.tanh());
	}

	public NeuralNetwork layer(int units, Activation fn, boolean bias) {
		//获取上一层网络的节点数
		int inputs = (layers.size() == 0) ? this.inputUnits : layers.get(layers.size() - 1).units();
		layers.add(new Layer(units, inputs, fn, bias));
		return this;
	}

	public NeuralNetwork layer(int units, Activation fn) {
		return layer(units, fn, true);
	}

	public NeuralNetwork layer(int units) {
		return layer(units, Activation.tanh());
	}

	//从前到后，将前一层的输出反馈给本层作输入,返回最后一层的feed结果
	public double[] feed(double[] x) {
		for (Layer l : layers)
			x = l.feed(x);
		return x;
	}

	//获取最后一层的Value
	public double[] output() {
		return layers.get(layers.size() - 1).value();
	}

	//将每一层的错误累加起来
	public double error() {
		double E = 0;
		for (Layer l : layers)
			E += l.error();
		return E;
	}

	public NeuralNetwork train(double[] x, double[] y) {
		//首先前向传播值
		double[] out = feed(x);
		//计算最终误差
		double[] s = new double[out.length];
		for (int i = 0; i < y.length; i++)
			s[i] = y[i] - out[i];
		//打印每层的偏差向量
		System.out.println(Layer.printVec(y)+" - "+Layer.printVec(out)+" => "+Layer.printVec(s));
		//反向传播错误率，同时更新了节点错误率
		for (int i = layers.size() - 1; i >= 0; i--)
			s = layers.get(i).backprop(s);
		//按照错误率和前一层值更新本层本层对应节点权重
		for (Layer l : layers)
			x = l.update(x, learningRate);
		return this;
	}

	public NeuralNetwork initialize() {
		for (Layer l : layers)
			l.initialize();
		return this;
	}

	//获取一个新的神经网络对象
	public static NeuralNetwork build() {
		return new NeuralNetwork();
	}

}

```

### 提升性能(利用冲量构造单层)

这里层的实现不在使用前面讲的基本方法来更新权值，而是使用单独的矩阵来存储权重delta值。

```java
package com.chengsluo.ann;
//使用冲量层来构建进行权值更新

//这会使得权重在同一方向的收敛速度加快
public class MomentumLayer extends Layer {

	double dW[][];
	double dbW[];
	double m = 0.1;

	public MomentumLayer(int units, int inputs, Activation fn, boolean bias) {
		super(units, inputs, fn, bias);
		dW = new double[units][];
		for (int i = 0; i < v.length; i++)
			dW[i] = new double[inputs];
		if (bias)
			dbW = new double[units];

	}

	@Override
	//在权重的更新过程中，使用该权重的新delta值与旧delta值的加权平均
	//其中的权值为参数m
	public double[] update(double[] o, double r) {
		for (int i = 0; i < v.length; i++) {

			if (bW != null) {
				dbW[i] = (1 - m) * r * err[i] + m * dbW[i];
				bW[i] += dbW[i];
			}

			double[] W = w[i];
			for (int j = 0; j < W.length; j++) {
				dW[i][j] = (1 - m) * r * err[i] * o[j] + m * dW[i][j];
				W[j] += dW[i][j];
			}
		}
		return v;
	}

}

```

## 测试ANN

### 训练一个异或模型

```java
package com.chengsluo.ann;

import org.junit.Test;

public class XorTest {

	public static final class Obs {
		public double[] x;
		public double[] y;
		
		public Obs(double[] x,double[]y) {
			this.x = x;
			this.y = y;
		}
	}
	
	public static Obs obs(double[] x,double[] y) {
		return new Obs(x,y);
	}

	@Test
	public void test() {
		Obs[] xorData = new Obs[]{
				obs(new double[]{-1,0},new double[]{0}),
				obs(new double[]{1,0},new double[]{1}),
				obs(new double[]{0,1},new double[]{1}),
				obs(new double[]{1,1},new double[]{0}),				
		};
		
		
		NeuralNetwork nn = NeuralNetwork.build().inputs(2).layer(3).layer(1);
		NeuralNetwork bad= NeuralNetwork.build().inputs(2).layer(1);
		
		System.out.println("nn\tbad");
		nn.initialize();
		bad.initialize();
		double lastErr = Double.POSITIVE_INFINITY;
		for(int i=0;i<1000;i++) {
			double err = 0.0;
			double errBad = 0.0;
			for(Obs x : xorData) {
				nn.train(x.x,x.y);
				bad.train(x.x,x.y);
				err += nn.error();
				errBad += bad.error();
			}
			System.out.println(err+"\t"+errBad);
			if(Math.abs(lastErr - err) < 1e-6) break;
			lastErr = err;
		}
	}

}

```

### 时间序列测试

```java
package com.chengsluo.ann;

import java.util.Random;
import org.junit.Test;

public class TimeSeriesTest {
	@Test
	public void test() {
		int k = 14;
		
		double[] x = new double[k];
		double[] y = new double[1];
		double   t = 0.0;
		Random twist = new Random();
		NeuralNetwork       nn    = NeuralNetwork.build().inputs(k).layer(5).layer(1).initialize();
		for(int i=0;i<1000;i++) {
			double base = Math.sin(t);
			y[0] = base + 0.5*twist.nextGaussian();
			if(i >= x.length) {
				nn.train(x, y);
				System.out.println(nn.output()[0]+"\t"+y[0]+"\t"+base);
				for(int j=0;i<x.length-1;j++) x[i] = x[i+1];
				x[x.length-1]=y[0];
			} else {
				x[i] = y[0];
			}
			t += 2.0*Math.PI/60.0;
		}
		
		
	}

}

```
###　冲量层加速测试

```java
package com.chengsluo.ann;

import org.junit.Test;

public class MomentumComparisonTest {
	public static final class Obs {
		public double[] x;
		public double[] y;

		public Obs(double[] x, double[] y) {
			this.x = x;
			this.y = y;
		}
	}

	public static Obs obs(double[] x, double[] y) {
		return new Obs(x, y);
	}

	@Test
	public void test() {
		Obs[] xorData = new Obs[] { obs(new double[] { -1, 0 }, new double[] { 0 }),
				obs(new double[] { 1, 0 }, new double[] { 1 }), obs(new double[] { 0, 1 }, new double[] { 1 }),
				obs(new double[] { 1, 1 }, new double[] { 0 }), };

		NeuralNetwork[] nn = new NeuralNetwork[10];
		for (int i = 0; i < 5; i++) {
			nn[i] = NeuralNetwork.build().inputs(2).layer(3).layer(1);
		}
		for (int i = 5; i < 10; i++) {
			nn[i] = NeuralNetwork.build().inputs(2).momentumLayer(3).momentumLayer(1);
		}
		double err[] = new double[10];
		for (int i = 0; i < 10; i++)
			nn[i].initialize();
		for (int j = 0; j < 300; j++) {
			for (int i = 0; i < 10; i++) {
				NeuralNetwork n = nn[i];
				err[i] = 0.0;
				for (Obs x : xorData) {
					n.train(x.x, x.y);
					err[i] += n.error();
				}
			}
			StringBuilder sb = new StringBuilder();
			for (int i = 0; i < 10; i++) {
				if (i > 0)
					sb.append("\t");
				sb.append(err[i]);
			}
			System.out.println(sb.toString());
		}

	}

}

```

## 参考资料

拜伦·埃利斯. 实时分析[M]. 机械工业出版社, 2016.

CS231n: Convolutional Neural Networks for Visual Recognition
http://cs231n.stanford.edu

SourceCode
https://github.com/chengsluo/code-java/tree/master/ArtificialNeuralNetwork