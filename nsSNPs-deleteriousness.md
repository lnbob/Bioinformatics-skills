<a name="content">目录</a>

[nsSNPs致病性分析](#title)
- [1. 自己写脚本分析蛋白质保守性](#self-build-script-for-conservation)







<h1 name="title">nsSNPs致病性分析</h1>

<a name="self-build-script-for-conservation"><h2>1. 自己写脚本分析蛋白质保守性 [<sup>目录</sup>](#content)</h2></a>

<p align="right">——仅以此纪念自己半路夭折的工具开发计划</p>

初衷：

> SNV有很多类型，其中落在编码区域的非同义突变 (nsSNPs) 会带来氨基酸序列的改变，很可能带来不良的影响，但是其影响程度与其**所处的位置**以及**氨基酸的理化性质**有关
> 
> 若非同义替换的发生位置是蛋白质的功能或结构的核心区域则更可能带来不良的影响，但是也不尽然，若突变前后的氨基酸的理化性质是否相近，比如分子大小相近，极性也很相似，那么也很可能是一个良性的变异
> 
> 因此氨基酸位置的保守性能很大程度上反映其所处位置的重要性

想法：

> **目的**：分析目的基因编码蛋白质序列各个氨基酸位点的保守性
> 
> **实现方法**：（1）获取与目的蛋白同源的蛋白质序列集合——从NCBI的Homogene数据库中获得某个基因（蛋白质）的同源基因（蛋白质）；（2）基于这些序列之间的多序列比对结果，推算出各个位点的保守性评分（计算方法在下文中说明）——分值在0-1之间，分值越低保守性越低；

1. **获取同源蛋白质序列及多序列比对结果**

	从NCBI的Homogene数据库中获得某个基因（蛋白质）的同源基因（蛋白质），获取这些同源蛋白的多序列比对结果，直接将网页上的多序列比对结果复制黏贴成纯文本`homogene.msa`

	<p align="center"><img src=./picture/nsSNPs-deleteriousness-HomoloGene-msa.png width=600 /></p>

	<p align="center">HomoloGene中的同源蛋白序列以多序列比对</p>
	
	将`homogene.msa`转换成FASTA格式：
	
	```
	# 读入上一步复制黏贴直接得到的原始多序列比对结果文件，除了空行之外，
	# 	每行都由四列组成（以任意空字符为分隔符），其中第一列为序列Id，
	# 	第三列为序列组成，则将这些序列读入并保存成哈希，以序列Id为key，
	# 	以序列组成为value
	# 文件读取结束后，即将每条序列以fasta格式输出

	$ perl -ne \
	'chomp;
	next if(/^$/);	# 跳过空行
	@F=split /\s+/;
	$Seq{$F[0]}.=$F[2];
	END{
		foreach $key (keys(%Seq)){
			print ">$key\n$Seq{$key}\n"
		}
	}' \
	homogene.msa >homogene.msa.fasta
	```

2. **计算保守性分值**

	用python脚本`ConservativeAnalysis.py`基于`homogene.msa.fasta`计算目标蛋白质序列各个位点的保守性，保守性的计算公式如下：

	<p align="center"><img src=./picture/conservative_formula-1.png width=600 /></p>

	这个公式其实用到了信息熵的思想，本质上是将总信息熵（或称最大信息熵）log<sub>2</sub>N，减去观测到的信息熵`-∑Plog2P`，若某一个位点的氨基酸组成多样性高，则保守性低，其观测信息熵就大，算出来的R值就小

	还可以用以下两种方式表示保守性：

	> - 相对保守性：
	> 
	> <p align="center"><img src=./picture/conservative_formula-2.png height=80 /></p>
	> 
	> - 保守性分位数：
	> <p align="center"><img src=./picture/conservative_formula-3.png height=80 /></p>

	Python脚本代码如下（脚本对上面提到的3中保守性分值都进行了计算）：

	```
	from Bio import SeqIO
	import sys
	import numpy as np
	import math
	
	# 用Biopython读入fasta序列
	def loadAASeq(infile):
		seq = {}
		for i in SeqIO.parse(infile,'fasta'):
			seq[i.id] = list(i.seq)
		return seq
	
	# 构建20种氨基酸和“-”gap占位符的字典，value为这些符号的编号
	def AAHash():
		AARow = {"A":0,
		"R":1,
		"N":2,
		"D":3,
		"C":4,
		"Q":5,
		"E":6,
		"G":7,
		"H":8,
		"I":9,
		"L":10,
		"K":11,
		"M":12,
		"F":13,
		"P":14,
		"S":15,
		"T":16,
		"W":17,
		"Y":18,
		"V":19,
		"-":20}
		return AARow
	
	# 对目标序列某一个位点的氨基酸组成进行频数统计
	def countFreq(seq,gaps,posi):
		'''
		seq:	[char] 目标序列名
		gaps:	[int] posi位置之前，目标序列存在的gap数
		posi:	[int] 当前位置,0-base
		'''
		if Seq[seq][posi] == '-':
			gaps += 1
		else:
			for key in Seq.keys():
				aa = Seq[key][posi]
				row = int(AA[aa])
				col = posi-gaps
				Count[row,col] += 1
		return gaps
	
	# 计算保守性分位数
	def Quantile(Vec):
		Vlen = len(Vec)
		SortIndex = Vec.argsort()
		
		# 计算每个数的秩序，当有多个数的值相等时，则用它们的最大秩序表示，如[1,2,2,5]，其中的两个2并列第3
		# 	主要思路为：若当前位置的值与上一个位置的值相同，说明存在数值并列情况，记下这个位置；
		#		若当前位置的值与上一个位置的值不相同，对之前的位置生成它们的秩序
		#	注意考虑第一个位置和最后一个位置的特殊情况：第一个位置没有更前的位置，最后一个位置没有更后的位置
		VRank = np.zeros(Vlen)
		forwardIndex = list()
		forwardValue = 0
		for i in range(0,Vlen):
			if Vec[SortIndex[i]] == forwardValue:
				forwardIndex.append(SortIndex[i])
				# 考虑最后一个位置的情况
				if i == Vlen-1:
					for j in forwardIndex:
						VRank[j] = i+1
			else:
				if len(forwardIndex) >= 2:
					for j in forwardIndex:
						VRank[j] = i
				else:
					# 考虑第一个位置的情况
					if i > 0:
						VRank[SortIndex[i-1]] = i
				# 考虑最后一个位置的情况
				if i == Vlen-1:
					VRank[SortIndex[i]] = i+1
				forwardIndex = [SortIndex[i]]
				forwardValue = Vec[SortIndex[i]]
		
		# 计算分位数
		PercValue = VRank*100/Vlen
		return PercValue
	
	if __name__ == '__main__':
		'''
		脚本共有两个输入参数：
		1. fasta格式的多序列比对结果
		2. 目标序列的id
		'''
		in_msa = sys.argv[1]
		nseq = sys.argv[2]
		Seq = loadAASeq(in_msa)
		AA = AAHash()
		seqlenAll = len(Seq[nseq]) # 多序列比对得到的比对整体的长度
		seqlenInt = seqlenAll - Seq[nseq].count('-') # 目标序列的氨基酸序列长度
		Count = np.zeros((21,seqlenInt),dtype = float) + 0.000001 # 加上一个很小的数是为了防止后面在执行对数计算时报错
		gaps = 0
		# 对每个位点进行频数统计，统计目标序列每个氨基酸位点的氨基酸组成频数分布，得到一个矩阵，行为20中氨基酸加“-”gap占位符，列为各个氨基酸位点	
		for i in range(0,seqlenAll):
			gaps = countFreq(nseq,gaps,i)
		colSum = sum(Count[...,0])
		# 每个位点氨基酸组成的频率
		CountRate = Count/colSum
		# 计算每个位点的保守性
		Shannon = -(CountRate*np.log2(CountRate)).sum(axis=0)
		Rindex = math.log2(21)-Shannon
		# 相对保守性
		Rrelat = (Rindex - Rindex.min())/(Rindex.max()-Rindex.min())
		# 保守性分位数，使用自定义的Quantile函数
		Rquant = Quantile(Rindex)
	```

	由于脚本中添加了详细的注释，这里就不展开讲解了

