---
 layout: post
 title: GROMACS教程：蛋白质配体复合物
 categories:
 - 科
 tags:
 - GMX
---

* toc
{:toc}


<ul>
<li>翻译: 李继存<br>
参考译文: 王梦凡, <a href="http://blog.sciencenet.cn/home.php?mod=space&amp;uid=275626&amp;do=blog&amp;id=861648">自己翻译的Gromacs教程: Protein-Ligand动力学模拟</a></li>
</ul>

<figure>
<img src="/GMX/GMXtut-5_3htb_lig_surface.png" alt="" />
<figcaption></figcaption></figure>

## 概述

<p>本教程将指导新用户进行一次模拟, 模拟体系为一个蛋白质(T4溶菌酶L99A/M102Q)与配体的复合物. 本教程重点关注配体的处理方面, 并假定用户熟悉GROMACS的基本操作. 否则, 在进行本教程前请先参阅一些基本的教程材料.</p>

<p>本教程假定用户使用5.x系列的GROMACS.</p>

## 第一步: 准备蛋白质的拓扑

<p>我们必须下载待研究蛋白质的结构文件. 在本教程中, 我们将使用T4溶菌酶L99A/M102Q(PDB代码为3HTB). 请先到<a href="http://www.rcsb.org/pdb/home/home.do">RCSB网站</a>下载此蛋白晶体结构的PDB文本文件.</p>

<p>一旦下载了结构文件, 你可以使用一些可视化程序, 如VMD, Chimera, PyMOL等, 来查看蛋白质的结构. 查看过分子之后, 你就要将其中的结晶水, PO4, BME去掉. 注意, 这个过程并不总是合适的(例如, 对结合位点的水分子就不适用). 对本教程的目的而言, 我们不需要结晶水或其他配体, 而只关心名称为<code>JZ4</code>的配体.</p>

<p>如果你需要一个已经处理好的.pdb文件来检查自己的处理结果, 可以从<a href="/GMX/GMXtut-5_3HTB_clean.pdb">这里</a>下载. 我们现在面临的问题是, GROMACS提供的任何力场都不能识别JZ4配体. 因此, 如果你试图直接用<code>gmx pdb2gmx</code>处理JZ4的.pdb文件, 将会出现致命错误. 只有当构建单元的条目出现在相应力场的.rtp文件中时, <code>gmx pdb2gmx</code>才能自动组装其拓扑. 但对于JZ4配体不是这种情况, 所以我们将分两步来准备体系的拓扑:</p>

<ol>
<li>使用<code>gmx pdb2gmx</code>准备蛋白质的拓扑</li>
<li>使用外部工具准备配体的拓扑</li>
</ol>

<p>由于要独立地准备这两个拓扑, 我们必须将蛋白质和JZ4放到分开的坐标文件中. 使用下面的命令将配体从复合物中脱离出来并保存到新的文件中:</p>

<pre><code>grep JZ4 3HTB_clean.pdb &gt; JZ4.pdb
</code></pre>

<p>上面的命令将从<code>3HTB_clean.pdb</code>文件中抽取出含<code>JZ4</code>的行, 保存到<code>JZ4.pdb</code>文件中. 这样, 准备蛋白质的拓扑文件很简单, 在3HTB的结构中没有缺失的原子或残基, 因此只需要简单地运行下面的命令:</p>

<pre><code>gmx pdb2gmx -f 3HTB_clean.pdb -o 3HTB_processed.gro -water spc
</code></pre>

<p><code>gmx pdb2gmx</code>将会处理结构文件, 并提示选择力场</p>

<pre><code>Select the Force Field:
From '/usr/local/gromacs/share/gromacs/top':
 1: AMBER03 protein, nucleic AMBER94 (Duan et al., J. Comp. Chem. 24, 1999-2012, 2003)
 2: AMBER94 force field (Cornell et al., JACS 117, 5179-5197, 1995)
 3: AMBER96 protein, nucleic AMBER94 (Kollman et al., Acc. Chem. Res. 29, 461-469, 1996)
 4: AMBER99 protein, nucleic AMBER94 (Wang et al., J. Comp. Chem. 21, 1049-1074, 2000)
 5: AMBER99SB protein, nucleic AMBER94 (Hornak et al., Proteins 65, 712-725, 2006)
 6: AMBER99SB-ILDN protein, nucleic AMBER94 (Lindorff-Larsen et al., Proteins 78, 1950-58, 2010)
 7: AMBERGS force field (Garcia &amp; Sanbonmatsu, PNAS 99, 2782-2787, 2002)
 8: CHARMM27 all-atom force field (CHARM22 plus CMAP for proteins)
 9: GROMOS96 43a1 force field
10: GROMOS96 43a2 force field (improved alkane dihedrals)
11: GROMOS96 45a3 force field (Schuler JCC 2001 22 1205)
12: GROMOS96 53a5 force field (JCC 2004 vol 25 pag 1656)
13: GROMOS96 53a6 force field (JCC 2004 vol 25 pag 1656)
14: GROMOS96 54a7 force field (Eur. Biophys. J. (2011), 40,, 843-856, DOI: 10.1007/s00249-011-0700-9)
15: OPLS-AA/L all-atom force field (2001 aminoacid dihedrals)
</code></pre>

<p>在本教程中, 我们将使用GROMOS96 43A1联合原子力场, 因此在命令提示处键入<code>9</code>, 然后回车.</p>

## 第二步: 准备配体的拓扑

<p>现在我们必须处理配体. 但对于力场无法自动识别的一些物种, 应该如何得到其力场参数呢? 正确处理配体是分子模拟中最富挑战性的工作之一. 力场的作者们花费了多年时间来发展自洽的力场, 将一个新物种引入已有的力场框架中不是一件容易的工作, 得到的任何新物种的力场参数都必须与原始力场自洽, 以保证其合理性.</p>

<p>对OPLS, AMBER和CHARMM力场, 发展时常利用各种量子化学计算, 相关的主要文献说明了发展所需要的流程. 对GROMOS力场, 其参数化方法相对模糊, 依赖于对凝聚相行为的经验拟合. 也就是, 计算每一原子类型初始的电荷和Lennard-Jones参数, 考察其精确性, 再进行细化. 尽管最终的结果非常令人满意, 模拟流体可以代表其在实际世界中的对应物, 但发展力场的过程可能非常艰难, 且常令人沮丧.</p>

<p>出于上面的原因, 人们非常倾向于使用一些自动化的工具. 对每一个力场, 都存在一些方法或软件程序, 可以给出与各种力场兼容的参数, 但精确度并不相同. 下表就是一些例子:</p>

<table><caption></caption>
<tr>
<td rowspan="2" style="text-align:left;">AMBER</td>
<td style="text-align:center;"><a href=http://ambermd.org/#AmberTools">Antechamber</a></td>
<td style="text-align:center;">生成分子的GAFF力场</td>
</tr>
<tr>
<td style="text-align:center;"><a href="http://code.google.com/p/acpype/">ACPYPE</a></td>
<td style="text-align:center;">Antechamber的Python界面, 输出GROMACS拓扑</td>
</tr>
<tr>
<td style="text-align:left;">CHARMM</td>
<td style="text-align:center;"><a href="http://dx.doi.org/10.1002/jcc.21367">CGenFF</a></td>
<td style="text-align:center;">用于CHARMM的通用力场</td>
</tr>
<tr>
<td rowspan="2" style="text-align:center;">GROMOS87/GROMOS96</td>
<td style="text-align:center;"><a href="http://davapc1.bioch.dundee.ac.uk/prodrg">PRODRG 2.5</a></td>
<td style="text-align:center;">生成拓扑的自动化服务器</td>
</tr>
<tr>
<td style="text-align:center;"><a href="http://compbio.biosci.uq.edu.au/atb/">ATB</a></td>
<td style="text-align:center;">生成拓扑的一个新服务器</td>
</tr>
<tr>
<td rowspan="2" style="text-align:left;">OPLS-AA</td>
<td style="text-align:center;"><a href="http://www.gromacs.org/Downloads/User_contributions/Other_software">Topolbuild</a></td>
<td style="text-align:center;">将Tripos.mol2文件转换为拓扑</td>
</tr>
<tr>
<td style="text-align:center;"><a href="http://www.gromacs.org/Downloads/User_contributions/Other_software">TopolGen</a></td>
<td style="text-align:center;">一个Perl脚本, 用于将全原子的.pdb文件转换为拓扑<sup>注</sup></td>
</tr>
</table>

<p>注: TopolGen是我写的, 它的功能还非常有限, 不能从表面上完全信任其输出的值. 对OPLS-AA, 正确的电荷计算方法在文献中有非常完整的说明. 因此, 你应该使用TopolGen来创建拓扑的大致结构, 并自己进行正确的电荷计算.</p>

<p>根据这个表格, 看起来GROMOS参数化非常简单. 将你的坐标文件上传到PRODRG(或使用网站提供的JME编辑器绘制分子结构), 按下<code>presto!</code>就可以得到拓扑文件. 不幸的是, 尽管PRODRG是一个非常方便的工具, 但给出的拓扑中有许多不精确的地方, 即便在最好的情况下, 也会导致使用这种拓扑进行模拟时得到的结果存在问题. 我们彻底分析了这些效应. 如果你感兴趣, 可以查看我们的<a href="http://pubs.acs.org/doi/abs/10.1021/ci100335w">论文</a>.</p>

<p>在本教程中, 我们将使用PRODRG创建所用配体的JZ4的初始拓扑. 打开PRODRG网站, 上传你的<code>JZ4.pdb</code>文件. 服务器会提供一些选项, 用以确定如何处理配体:</p>

<ul>
<li><code>Chirality (yes/no)</code>: 保留输入分子的手性中心(<code>yes</code>), 或重新创建(<code>no</code>). 我们所用的配体本身无手性，因此选择默认设置.</li>
<li><code>Charges (full/reduced)</code>: 对凝聚态体系使用<code>full</code>电荷(43A1力场); 对真空模拟使用<code>reduced</code>电荷(43B1力场). 几乎对所有的情况都是有<code>full</code>电荷. 无论如何, 43B1力场的精确性存在争议.</li>
<li><code>EM (yes/no)</code>: 进行能量最小化(<code>yes</code>), 或不进行能量最小化而直接使用输入坐标(<code>no</code>). 本教程中, 我们希望根据已有的坐标来构建蛋白配体复合物, 从而不改变已知复合物中的原子位置, 因此不需要PRODRG对我们的分子进行 <strong>真空</strong> 中的能量最小化, 对这个选项使用<code>no</code>.</li>
</ul>

<p>我们将使用PRODRG给出的两个文件, 将<code>The GROMOS87/GROMACS coordinate file (polar/aromatic hydrogens)</code>的输出保存为文本文件, 命名为<code>jz4.gro</code>, 然后将<code>The GROMACS topology</code>的输出保存为文本文件<code>drg.itp</code>.</p>

<p>打开<code>drg.itp</code>文件, 让我们看一看JZ4拓扑文件中的<code>[ atoms ]</code>节:</p>

<pre><code>[ atoms ]
;   nr      type  resnr resid  atom  cgnr   charge     mass
     1       CH3     1  JZ4      C4     1    0.000  15.0350
     2       CH2     1  JZ4     C14     2    0.059  14.0270
     3       CH2     1  JZ4     C13     2    0.060  14.0270
     4         C     1  JZ4     C12     2   -0.041  12.0110
     5       CR1     1  JZ4     C11     2   -0.026  12.0110
     6        HC     1  JZ4     H11     2    0.006   1.0080
     7       CR1     1  JZ4      C7     2   -0.026  12.0110
     8        HC     1  JZ4      H7     2    0.006   1.0080
     9       CR1     1  JZ4      C8     2   -0.026  12.0110
    10        HC     1  JZ4      H8     2    0.007   1.0080
    11       CR1     1  JZ4      C9     2   -0.026  12.0110
    12        HC     1  JZ4      H9     2    0.007   1.0080
    13         C     1  JZ4     C10     3    0.137  12.0110
    14        OA     1  JZ4     OAB     3   -0.172  15.9994
    15         H     1  JZ4     HAB     3    0.035   1.0080
</code></pre>

<p>我马上就注意到此拓扑中有三处错误:</p>

<ol>
<li>电荷组2包含了整个苯环, 几乎是分子中的所有原子.</li>
<li>等价官能团(C-H)的电荷并不相同. 例如8号H原子的电荷为+0.006 e, 而12号H原子的电荷为+0.007 e.</li>
<li>这些原子上的电荷与所用力场中指定的电荷并不一致.</li>
</ol>

<p>那我们应该怎么办呢? 如果你读过上面提到的论文, 你可能已经知道了我会给出的建议: 以构建单元的方式根据等价官能团来指定电荷与电荷组. 对任何未知的基团, 执行论文中建议的电荷计算. 由于JZ4仅含有蛋白质中存在的一些官能团(憎水链, 芳香环以及醇), 这很容易完成, 我们可以对这些部分重新指定电荷与电荷组. 在本教程中我将不再涉及正确拓扑的验证, 那太费时了, 只需指出下面这点就足够了: 你必须说服审稿人, 让他相信你对待研究分子所用的参数是可信的. 它们应该能够重现一些众所周知的物理量, 如logP或 <span class="math">\(\D G_\text{hydr}\)</span> 等, 像GROMOS96文献中的那样. 如果对你的体系不能满足这些要求, 你可能需要重新考虑下你所选择的力场.</p>

<p>对<code>drg.itp</code>的<code>[ atoms ]</code>节, 我将构建如下的拓扑:</p>

<pre><code>[ atoms ]
;   nr      type  resnr resid  atom  cgnr   charge     mass
     1       CH3     1  JZ4      C4     1    0.000  15.0350
     2       CH2     1  JZ4     C14     2    0.000  14.0270
     3       CH2     1  JZ4     C13     2    0.000  14.0270
     4         C     1  JZ4     C12     2    0.000  12.0110
     5       CR1     1  JZ4     C11     3   -0.100  12.0110
     6        HC     1  JZ4     H11     3    0.100   1.0080
     7       CR1     1  JZ4      C7     4   -0.100  12.0110
     8        HC     1  JZ4      H7     4    0.100   1.0080
     9       CR1     1  JZ4      C8     5   -0.100  12.0110
    10        HC     1  JZ4      H8     5    0.100   1.0080
    11       CR1     1  JZ4      C9     6   -0.100  12.0110
    12        HC     1  JZ4      H9     6    0.100   1.0080
    13         C     1  JZ4     C10     7    0.150  12.0110
    14        OA     1  JZ4     OAB     7   -0.548  15.9994
    15         H     1  JZ4     HAB     7    0.398   1.0080
</code></pre>

<p>注意到, 等价官能团的电荷是相同的, 憎水组完全不带电, 电荷组的大小也很合适(2&#8211;3个原子). 拓扑的其他部分足够精确. PRODRG给出的键合参数通常是合理的, 现在我们就可以构建体系的拓扑并重新构建蛋白配体复合物了.</p>

<p><strong>创建复合物</strong></p>

<p>根据<code>gmx pdb2gmx</code>的输出, 我们已经有了一个名称为<code>3HTB_processed.gro</code>的文件, 它包含了经处理后与力场相容的蛋白质结构. 我们也有一个来自PRODRG的文件<code>jz4.gro</code>, 其中包含了所有需要的H原子. 将<code>3HTB_processed.gro</code>文件另存为一个待处理的新文件, 如<code>complex.gro</code>, 因为将JZ4添加到蛋白质就可以得到蛋白配体的复合物. 接下来, 复制<code>jz4.gro</code>的坐标部分粘贴到<code>complex.gro</code>文件中, 位于蛋白质原子最后一行的下面, 盒向量的前面. 原先的内容</p>

<pre><code>163ASN      C 1691   0.621  -0.740  -0.126
163ASN     O1 1692   0.624  -0.616  -0.140
163ASN     O2 1693   0.683  -0.703  -0.011
 5.99500   5.19182   9.66100   0.00000   0.00000  -2.99750   0.00000   0.00000   0.00000
</code></pre>

<p>会变为(绿色粗体为新添加的文本):</p>

<pre><code>163ASN      C 1691   0.621  -0.740  -0.126
163ASN     O1 1692   0.624  -0.616  -0.140
163ASN     O2 1693   0.683  -0.703  -0.011
   1JZ4  C4       1   2.429  -2.412  -0.007
   1JZ4  C14      2   2.392  -2.470  -0.139
   1JZ4  C13      3   2.246  -2.441  -0.181
   1JZ4  C12      4   2.229  -2.519  -0.308
   1JZ4  C11      5   2.169  -2.646  -0.295
   1JZ4  H11      6   2.135  -2.683  -0.199
   1JZ4  C7       7   2.155  -2.721  -0.411
   1JZ4  H7       8   2.104  -2.817  -0.407
   1JZ4  C8       9   2.207  -2.675  -0.533
   1JZ4  H8      10   2.199  -2.738  -0.621
   1JZ4  C9      11   2.267  -2.551  -0.545
   1JZ4  H9      12   2.306  -2.516  -0.640
   1JZ4  C10     13   2.277  -2.473  -0.430
   1JZ4  OAB     14   2.341  -2.354  -0.434
   1JZ4  HAB     15   2.369  -2.334  -0.528
   5.99500   5.19182   9.66100   0.00000   0.00000  -2.99750   0.00000   0.00000   0.00000
</code></pre>

<p>由于我们添加了15个原子到.gro文件中, 将<code>complex.gro</code>文件中第二行的数字增加15. 现在坐标文件中应该共有1708个原子.</p>

<p><strong>构建拓扑</strong></p>

<p>在体系拓扑中包含JZ4配体的参数很简单: 在<code>topol.top</code>中包含位置限制文件的后面插入一行, 内容为<code>#include &quot;drg.itp&quot;</code>. 包含位置限制的部分表示<code>Protein</code>分子类型节的结束. 原先的内容:</p>

<pre><code>; Include Position restraint file
#ifdef POSRES
#include &quot;posre.itp&quot;
#endif

; Include water topology
#include &quot;gromos43a1.ff/spc.itp&quot;
</code></pre>

<p>将变为:</p>

<pre><code>; Include Position restraint file
#ifdef POSRES
#include &quot;posre.itp&quot;
#endif

; Include ligand topology
#include &quot;drg.itp&quot;

; Include water topology
#include &quot;gromos43a1.ff/spc.itp&quot;
</code></pre>

<p>最后调整<code>[ molecules ]</code>部分, 因为新添加了一个分子到<code>complex.gro</code>文件中, 所以必须修改相应的部分</p>

<pre><code>[ molecules ]
; Compound        #mols
Protein_chain_A     1
JZ4                 1
</code></pre>

<p>拓扑和坐标文件现在就与体系的内容一致了.</p>

## 第三步: 定义单元盒子并添加溶剂

<p>这一步的工作流程与任何其他MD模拟一样. 我们将定义单元盒子并填充水分子</p>

<pre><code>gmx editconf -f complex.gro -o newbox.gro -bt dodecahedron -d 1.0
</code></pre>

<ul>
<li><p><code>-f</code>: 输入复合物的结构文件</p></li>
<li><p><code>-o</code>: 输出复合物的结构文件, 包含新盒子的信息</p></li>
<li><p><code>-bt</code>: 定义盒子形状: triclinic(三斜体), cubic(立方体), dodecahedron(十二面体), octahedron(八面体)</p></li>
<li><p><code>-d</code>: 定义盒子边缘距离分子边缘的距离(单位: nm), 通常不能小于0.85 nm</p>

<p>gmx solvate -cp newbox.gro -cs spc216.gro -p topol.top -o solv.gro</p></li>
<li><p><code>-cp</code>: 输入含盒子的结构文件</p></li>
<li><p><code>-cs</code>: 指定spc水盒子</p></li>
<li><p><code>-p</code>: 输出含溶剂的复合物拓扑文件</p></li>
<li><p><code>-o</code>: 输出含溶剂的复合物结构文件</p></li>
</ul>

## 第四步: 添加离子

<p>现在我们有了一个已经溶剂化的体系, 其中包含一个带电的蛋白质. <code>gmx pdb2gmx</code>的输出提示我们, 蛋白质的净电荷为+6 e(根据蛋白质的氨基酸组分计算所得). 如果你忽略了<code>gmx pdb2gmx</code>输出的这一信息, 可查看<code>topol.top</code>文件中<code>[ atoms ]</code>部分的最后一行, 其(部分)内容应为<code>qtot 6</code>. 由于生命体系中不存在净电荷, 我们必须在体系中添加离子.</p>

<p>使用<code>gmx grompp</code>整合.tpr文件, 可利用任何.mdp文件. 我使用了运行能量最小化的.mdp文件, 因为它只需要最少的参数, 因此最容易维护. 例如, 你可以使用这个<a href="/GMX/GMXtut-5_em.mdp">.mdp文件</a>.</p>

<pre><code>gmx grompp -f em.mdp -c solv.gro -p topol.top -o ions.tpr
</code></pre>

<p>将所得的.tpr文件传给<code>gmx genion</code>:</p>

<pre><code>gmx genion -s ions.tpr -o solv_ions.gro -p topol.top -pname NA -nname CL -nn 6
</code></pre>

<ul>
<li><code>-s ions.tpr</code>: 输入预处理得到的tpr文件</li>
<li><code>-o solv_ions.gro</code>: 输出添加了抗衡离子后的结构文件</li>
<li><code>-p topol.top</code>: 输出拓扑文件</li>
<li><code>-pname NA</code>: 添加的阳离子类型</li>
<li><code>-nname CL</code>: 添加的阴离子类型</li>
<li><code>-nn 6</code>: 添加的阴离子个数</li>
</ul>

<p>在以前的GROMACS版本中, 使用<code>-pname</code>和<code>-nname</code>指定的离子名称是依赖于力场的, 但自4.5版本开始, 离子名称已经标准化了: 指定的原子名称始终是大写的元素符号, 与<code>[ moleculetype ]</code>一致, 残基名称可以追加电荷符号(+/-), 也可以不追加. 如果遇到困难, 请参考<code>ions.itp</code>中的正确命名.</p>

<p>运行上面的命令后, 会提示<code>Select a continuous group of solvent molecules</code>时, 选择溶剂</p>

<p><code>Group    15 (            SOL) has 31341 elements</code></p>

<p>程序会自动将溶剂中的某些原子用CL来代替, <code>topol.top</code>文件中也会修改溶剂和抗衡离子的信息.</p>

<p><code>[ molecules ]</code>部分现在看起来应该这样:</p>

<pre><code>[ molecules ]
; Compound        #mols
Protein_chain_A     1
JZ4                 1
SOL             10441
CL                  6
</code></pre>

## 第五步: 能量最小化

<p>现在体系已经整合起来了, 我们可以使用<code>gmx grompp</code>并利用这个<a href="/GMX/GMXtut-5_em_real.mdp">输入参数文件</a>来创建一个二进制输出文件:</p>

<pre><code>gmx grompp -f em_real.mdp -c solv_ions.gro -p topol.top -o em.tpr
</code></pre>

<p>注意, 在上面的.mdp文件中, 我设置了<code>energygrps = Protein JZ4</code>. 检查蛋白质和配体之间非键相互作用的各种组分可能会有用, 特别是运行出错时.</p>

<p>当运行<code>gmx genbox</code>和<code>gmx genion</code>时, 确保你已经更新了<code>topol.top</code>, 否则, 你可能会得到非常多的错误信息(<code>number of coordinates in coordinate file does not match topology</code>等等).</p>

<p>现在就可以使用<code>gmx mdrun</code>来运行EM了:</p>

<pre><code>gmx mdrun -v -deffnm em
</code></pre>

<p>在我的机器上, 体系收敛相对较快:</p>

<pre><code>Steepest Descents converged to Fmax &lt; 1000 in 178 steps
Potential Energy  = -5.0933919e+05
Maximum force     =  9.1495343e+02 on atom 1266
Norm of force     =  5.1248310e+01
</code></pre>

<p>像在<a href="http://jerkwin.github.io/9999/10/31/GROMACS%E4%B8%AD%E6%96%87%E6%95%99%E7%A8%8B/#TOC1.7.2">溶菌酶教程</a>中一样, 你可以使用<code>gmx energy</code>模块监测势能的各个分量. 我就不在这里展示这些原理了, 你自己试试吧.</p>

<p>现在我们的体系已经处于能量最小点了, 我们可以开始真正的动力学了.</p>

## 第六步: NVT平衡

<p>平衡我们的蛋白质配体复合物与平衡任何其他蛋白质水溶液体系一样. 但对我们的情况需要有些特殊考虑:</p>

<ol>
<li>对配体施加限制</li>
<li>处理温度耦合组</li>
</ol>

<p><strong>限制配体</strong></p>

<p>PRODRG并不会为我们的配体生成一个类似<code>posre.itp</code>的文件, 但GROMACS提供的<code>gmx genrestr</code>模块可帮助我们完成这项工作. 利用PRODRG给出的<code>jz4.gro</code>文件运行<code>gmx genrestr</code></p>

<pre><code>gmx genrestr -f jz4.gro -o posre_jz4.itp -fc 1000 1000 1000
</code></pre>

<ul>
<li><code>-f jz4.gro</code>: 输入配体的结构文件</li>
<li><code>-o posre_jz4.itp</code>: 输出配体施加位置限制的itp文件</li>
<li><code>-fc 1000 1000 1000</code>: 位置限制的力常数(单位: kJ/mol-nm<sup>2</sup>)</li>
</ul>

<p>现在我们需要在拓扑文件中包含这些信息. 这可以采取几种不同的方法, 取决于我们要使用的条件. 如果当蛋白质被限制的时候, 我们也要简单地限制配体, 将下面几行添加到拓扑文件中的指定位置:</p>

<pre><code>; Include Position restraint file
#ifdef POSRES
#include &quot;posre.itp&quot;
#endif

; Include ligand topology
#include &quot;drg.itp&quot;

; Ligand position restraints
#ifdef POSRES
#include &quot;posre_jz4.itp&quot;
#endif

; Include water topology
#include &quot;gromos43a1.ff/spc.itp&quot;
</code></pre>

<p>位置很关键! 你必须将拓扑中调用<code>posre_jz4.itp</code>的语句放在指定的位置. <code>drg.itp</code>内的参数定义了配体的<code>[ moleculetype ]</code>部分, 分子类型的定义在包含水分子拓扑(<code>spc.itp</code>)处结束. 将<code>posre_jz4.itp</code>调用放于任何其他位置都会将位置限制参数施加到错误的分子类型上.</p>

<p>如果在平衡过程中你需要更多的控制, 即, 你想要独立地限制蛋白质和配体, 你可以使用不同的<code>#ifdef</code>块来控制配体的包含位置限制文件的位置, 像这样:</p>

<pre><code>; Include Position restraint file
#ifdef POSRES
#include &quot;posre.itp&quot;
#endif

; Include ligand topology
#include &quot;drg.itp&quot;

; Ligand position restraints
#ifdef POSRES_LIG
#include &quot;posre_jz4.itp&quot;
#endif

; Include water topology
#include &quot;gromos43a1.ff/spc.itp&quot;
</code></pre>

<p>对后一情况, 为 <strong>同时</strong> 限制蛋白质和配体, 我们需要在.mdp文件中指定<code>define = -DPOSRES -DPOSRES_LIG</code>. 如何处理你的体系取决于你自己. 上面的这些例子只是为了展示GROMACS提供的灵活性. 对一个标准的平衡过程, 同时限制蛋白质和配体可能足够了. 你自己的需求可能不同.</p>

<p><strong>热浴</strong></p>

<p>温度耦合的正确控制是非常敏感的事. 将每个分子类型耦合到其自身的热浴组并不是个好主意. 例如, 如果你使用</p>

<pre><code>tc_grps = Protein JZ4 SOL CL
</code></pre>

<p>你的体系可能会爆开, 因为对只有几个原子的组(即, JZ4和CL), 在控制其动能的涨落时, 温度耦合算法不够稳定. <strong>不要对体系中的每一单个物种使用独立的耦合.</strong></p>

<p>典型的方法是设定<code>tc_grps = Protein Non-Protein</code>并运行. 不幸的是, <code>Non-Protein</code>组也包含JZ4. 由于JZ4与蛋白质物理上结合非常紧密, 最好将它们一起考虑. 也就是说, 在温度耦合时将JZ4当作蛋白质的一部分. 同样的, 将我们插入的少数几个Cl-作为溶剂的一部分. 为此, 我们需要一个特殊的组, 其中包含蛋白质和JZ4. 我们可以使用<code>gmx make_ndx</code>模块来完成这个工作, 只要提供完整体系的任意坐标文件即可. 在这里, 我使用<code>em.gro</code>, 它是体系(能量最小化后)的输出结构:</p>

<pre><code>gmx make_ndx -f em.gro -o index.ndx
</code></pre>

<p>屏幕显示:</p>

<pre><code>...
  0 System              : 33037 atoms
  1 Protein             :  1693 atoms
  2 Protein-H           :  1301 atoms
  3 C-alpha             :   163 atoms
  4 Backbone            :   489 atoms
  5 MainChain           :   653 atoms
  6 MainChain+Cb        :   805 atoms
  7 MainChain+H         :   815 atoms
  8 SideChain           :   878 atoms
  9 SideChain-H         :   648 atoms
 10 Prot-Masses         :  1693 atoms
 11 non-Protein         : 31344 atoms
 12 Other               :    15 atoms
 13 JZ4                 :    15 atoms
 14 CL                  :     6 atoms
 15 Water               : 31323 atoms
 16 SOL                 : 31323 atoms
 17 non-Water           :  1714 atoms
 18 Ion                 :     6 atoms
 19 JZ4                 :    15 atoms
 20 CL                  :     6 atoms
 21 Water_and_ions      : 31329 atoms
</code></pre>

<p>使用下面的命令将<code>Protein</code>和<code>JZ4</code>组合并在一起, 其中的<code>&gt;</code>为<code>gmx make_ndx</code>的提示符:</p>

<pre><code>&gt; 1 | 13
&gt; q
</code></pre>

<p>现在我们可以设置<code>tc_grps = Protein_JZ4 Water_and_ions</code>来达到我们需要的<code>Protein Non-Protein</code>的效果了.</p>

<p>使用这个<a href="/GMX/GMXtut-5_nvt.mdp">.mdp文件</a>进行NVT平衡:</p>

<pre><code>gmx grompp -f nvt.mdp -c em.gro -p topol.top -n index.ndx -o nvt.tpr

gmx mdrun -deffnm nvt
</code></pre>

## 第七步: NPT平衡

<p>完成NVT平衡后, 使用这个<a href="/GMX/GMXtut-5_npt.mdp">.mdp文件</a>进行NPT平衡:</p>

<pre><code>gmx grompp -f npt.mdp -c nvt.gro -t nvt.cpt -p topol.top -n index.ndx -o npt.tpr

gmx mdrun -deffnm npt
</code></pre>

## 第八步: 成品MD

<p>完成前面两个阶段的平衡后, 体系已经在需要的温度和压力下平衡好了, 现在我们可以放开位置限制运行成品模拟收集数据了. 这个过程和我们以前见到的一样, 我们将使用检查点文件(在这种情况下它保留了压力耦合信息)运行<code>gmx grompp</code>. 我们将运行1 ns的MD模拟, 所用的.mdp可以在<a href="/GMX/GMXtut-5_md.mdp">这里</a>下载.</p>

<pre><code>gmx grompp -f md.mdp -c npt.gro -t npt.cpt -p topol.top -n index.ndx -o md_0_1.tpr

gmx mdrun -deffnm md_0_1
</code></pre>

<p><code>gmx mdrun</code>步最好在集群或多核工作站上并行运行, 因为可能需要几个小时才能完成. 并行运行的正确命令类似下面这样:</p>

<pre><code>gmx mdrun -nt X -deffnm md_0_1
</code></pre>

<p>其中<code>X</code>代表模拟中使用的处理器的数目. 默认情况下, <code>gmx mdrun</code>将会使用运行机器上所有可用的核, 因此如果不想使用所有的核, 你只需要明确地指定<code>X</code>的值即可. 在接近<code>gmx grompp</code>输出的最后, 你应该可以看到类似下面的内容:</p>

<pre><code>Estimate for the relative computational load of the PME mesh part: 0.34
</code></pre>

<p>PME负载的估计值暗示了应该使用多少核进行PME计算, 多少核进行PP计算. 请参考<a href="http://dx.doi.org/10.1021/ct700301q">GROMACS 4的论文</a>以及GROMACS手册了解更多信息. 对我们使用的八面体盒子, 最佳设置时PME负载为0.33(2:1 PP:PME, 我们很幸运!). 当执行<code>gmx mdrun</code>时, 程序会自动决定用于PP和PME计算的最佳处理数, 因此, 请确保你使用了合适的核数(<code>-nt X</code>的值)用于计算, 这样你才能获得最佳性能.</p>

## 第九步: 分析

<p>本教程将不再列出你要执行的所有类型的分析, 但会介绍一些可能的分析. 对JZ4这样的配体可能会形成氢键, 因此你可以使用<code>gmx hbond</code>来确定是否形成氢键. 在<code>md.mdp</code>文件中使用<code>energygrps</code>可以检查JZ4和蛋白质之间非键势能的各种组分, 蛋白质和配体之间的结合, 是静电相互作用占主导地位还是憎水相互作用占主导地位?</p>

## 总结

<p>你现在已经使用GROMACS运行了一个蛋白质配体复合物的分子动力学模拟. 不要认为本教程足够全面, 你还可以使用GROMACS完成更多类型的模拟(只举几个例子, 如自由能计算, 非平衡MD, 简正模式分析). 你也应该阅读文献以及GROMACS手册, 对这里提供的.mdp文件中的参数进行调整, 以提高运行效率与精确度.</p>

<p>如果你对改进这个教程有些建议, 如果你发现了错误, 或者你觉得有些地方不够清楚, 请给我发邮件<code>jalemkul@vt.edu</code>, 不要客气. 请注意: 这不是邀请你因为GROMACS的问题而给我发邮件. 我并不是作为一个私人家教或个人客服在为自己打广告. 那是<a href="http://lists.gromacs.org/mailman/listinfo/gmx-users">GROMACS用户邮件列表</a>的事. 我可能会在那里帮助你, 但那只是作为对整个社区的服务, 而不只针对最终用户.</p>

<p>祝你模拟愉快!</p>

