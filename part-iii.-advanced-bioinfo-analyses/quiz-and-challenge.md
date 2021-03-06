# Quiz & Challenge

{% hint style="info" %}
[e-Maize Challenge](http://emaize.imaze.org/) @ [iMAZE](http://www.imaze.org/)
{% endhint %}

## 机器学习实例 1：eMaize Challenge -- 通过基因型预测表型

### 0.背景简介

该通过基因型预测表型的实例来自我们在imaze上的 [eMaize challenge](http://emaize.imaze.org): eMaize问题要求我们以SNP作为特征，通过训练一个模型，对玉米的三个性状进行预测。 接下来的教程会展示从原始数据开始，如何对数据进行转换，存取，特征选择以及回归和后续分析的整个过程。本问题最基本的目标是使用6210个样本中的前4754个样本作为训练集，预测其他样本的性状

### I.上机指南

本任务依赖于python语言及jupyter notebook，所需工具已安装到虚拟机。以下指南的所有代码均可在4.Emaize/jupyter\_notebook/basic\_tutorial.ipynb 中找到。

使用方法：

* 打开终端，进入Bioinfo\_Lab/4.Emaize/ 文件夹
* 输入jupyter notebook，等待弹出窗口，或者手动复制粘贴终端显示的网址到浏览器。
* 点击jupyter\_notebook,再点击basic\_tutorial.ipynb,即可看到本部分的教程。按照相关指南一步一步运行即可。本部分接下来的内容与basic\_tutorial.ipynb中的内容一致。

**jupyter notebook基本使用指南：**

本教程使用jupyter notebook，可以让使用者获得更好的体验，方便对代码进行修改，以及对结果进行查看和分析

* 一段相关的代码在同一个代码框中书写
* 同时按住shift与enter即可运行选中的代码框的代码
* 仅仅按enter键具有回车的效果
* **使用上方的编辑栏：**

点击加号在两个代码框中间插入新的代码框，删除代码框点击剪刀，中止程序点击方框

```text
#导入必需的库
import warnings
warnings.filterwarnings('ignore')
import numpy as np
from sklearn.random_projection import SparseRandomProjection
from scipy.sparse import load_npz, save_npz
import scipy.stats
import pandas as pd
import sklearn
import h5py
from sklearn.metrics import r2_score
from scipy.stats.stats import pearsonr
import pickle
from sklearn.linear_model import LinearRegression
#import xgboost
#from xgboost.sklearn import XGBRegressor
from sklearn.linear_model import Ridge
from sklearn.kernel_ridge import KernelRidge
from sklearn import neighbors
from sklearn.ensemble import RandomForestRegressor
from sklearn.gaussian_process import GaussianProcessRegressor
from sklearn.gaussian_process.kernels import DotProduct
from tqdm import tqdm_notebook as tqdm
from IPython.display import display, Image
%pylab inline
```

#### 1.查看原始数据

**1.1 数据种类**

* genotype：SNP数据，每个位点可能有三种情况，如AA，AT，TT
* trait：共三种，trait1开花期，trait2株高，trait3产量，为连续值
* 原始数据中有6210个样本，每个样本SNP位点约为190万个,

因为计算资源的原因，这里仅仅选取其中的5000个SNP作为示例,因为数据量的原因，结果肯定不够理想

**1.2 数据格式**

txt存储格式不适合大数据读取的问题，对内存的占用过多。对于结构化的、能够存储为矩阵的数据，可以使用HDF5格式存取，内存占用小，读取速度快

**读取SNP数据，数据格式为HDF5**

**在命令行查看数据shape的方法为：**

* cd至文件路径下，输入：h5ls snp\_5000
* 若使用了新版h5py，可能出现无法打开的情况，此时输入HDF5\_USE\_FILE\_LOCKING=FALSE h5ls snp\_5000

```text
#使用h5py读取5000个SNP：
with h5py.File('data/snp_5000') as f:
snps = f['snp'][:]
#查看数据shape,h5py读取出的snps是一个矩阵，可以用.shape查看其shape
snps.shape
```

```text
#查看数据内容
snps
```

**读取性状数据**

使用numpy/pandas均可读取性状数据并显示，这里用pandas展示，真正计算时一般用numpy

```text
traits = pd.read_csv('data/pheno_emaize.txt',delimiter='\t')
#仅显示前五个,4754之后的样本的性状是未知的
traits.head()
```

```text
#pandas dataframe也可查看shape
print traits.shape
```

```text
#查看性状的分布情况
trait1 = np.array(traits['trait1'])[:4754]
trait2 = np.array(traits['trait2'])[:4754]
trait3 = np.array(traits['trait3'])[:4754]
fig, ax = plt.subplots(1,3, figsize=(15,3))
ax[0].hist(trait1,bins = 50)
ax[0].set_title('normalized trait1 value distribution')
ax[1].hist(trait2,bins = 50)
ax[1].set_title('normalized trait2 value distribution')
ax[2].hist(trait3,bins = 50)
ax[2].set_title('normalized trait3 value distribution')
```

**查看训练集与测试集的划分**

下图中彩色部分为训练集性状，白色部分为待预测性状 可以发现其划分方式并不随机，这会导致常规的机器学习方法出现一些问题，由于是基础介绍，这里不讨论如何解决这个问题。

```text
def generate_parent_table(phenotype_file):
phenotypes = pd.read_table(phenotype_file)
pedigree = phenotypes['pedigree'].str.split('_', expand=True)
pedigree.columns = ['f', 'X', 'm']
phenotypes = pd.concat([phenotypes, pedigree], axis=1)
phenotypes['number'] = np.arange(phenotypes.shape[0])
parent_table = phenotypes.pivot_table(values='number', index=['m'], columns=['f'], dropna=False)
male_ids = ['m%d' % i for i in range(1, parent_table.shape[0] + 1)]
female_ids = ['f%d' % i for i in range(1, parent_table.shape[1] + 1)]
parent_table = parent_table.loc[male_ids, female_ids]
return parent_table
phenotype_file = 'data/pheno_emaize.txt'
parent_table = generate_parent_table(phenotype_file)
phenotypes = pd.read_table('data/pheno_emaize.txt')
fig, ax = subplots(3,1, figsize=(20, 10))
for i in range(3):
trait = ['trait1','trait2','trait3'][i]
ax[i].matshow(np.take(np.ravel(phenotypes[trait].values), parent_table), cmap=cm.RdBu)
ax[i].set_title('Phenotypes of training data (%s)'%trait)
```

#### 2. 将SNP数据编码为向量

每个位点的碱基只有三种情况，不会出现更多碱基组合的可能，比如某位点仅有AA，AT，TT三种可能的情况 我们可以采取三种方式对其编码：

* 转化为0、1、2。找到minor allele frequency（MAF），即两种碱基（如A、T）中出现频率低的那个，以A作为MAF为例，则TT为0，AT为1，AA为2，这样可以突出MAF
* 转化为3-bit one hot vector,$\[1,0,0\]^T,\[0,1,0\]^T,\[0,0,1\]^T$这样可以保持三种向量在空间距离的一致
* 转化为2-bit vector,则AA，AT，TT分别编为$\[1,0\]^T,\[1,1\]^T,\[0,1\]^T$,不需要考虑MAF

我们采取第三种方式

```text
def convert_2bit(seq):
genotypes = np.zeros([6210,2])
a = seq[1].split('/')
for i in range(6210):
if seq[4:][i] == a[0] + a[0]:
genotypes[i] = np.array([0,1])
if seq[4:][i] == a[0] + a[1]:
genotypes[i] = np.array([1,0])
if seq[4:][i] == a[1] + a[1]:
genotypes[i] = np.array([1,1])
genotypes = genotypes.astype('int').T
return genotypes
```

**注意，接下来的步骤耗时14min**

真实计算时此步骤使用C加速计算，这里为了连贯性仅仅展示python的方法

可以跳过接下来的代码框步骤，直接使用处理好的结果，结果放在 /data/2bit\_geno

```text
#该代码框可跳过以节约时间，直接运行下一个代码框
geno_conv = convert_2bit(snps[1])
for i in tqdm(range(4999)):
geno_conv = np.concatenate((geno_conv,convert_2bit(snps[i+2])),axis =0)
```

```text
#读取处理成2bit格式的SNP
with h5py.File('data/2bit_geno') as f:
geno_conv = f['data'][:]
```

```text
#查看SNP的大致情况
fig, ax = plt.subplots(figsize=(5,10))
ax.matshow(geno_conv[:300,:200],cmap = cm.binary_r)
```

#### 3. 特征提取与降维

* 原数据每个样本有190万个SNP，转化为2bit coding后有大约380万个feature，大多数的feature可能是冗余的
* 过多的feature使得机器学习模型无法承受，一个考虑时间开销及效果的feature数量应该在几千至几万量级

**特征选择：**

特征选择的方法包括filter，wrapper和embedding三大类 我们使用过如下方法：

* Mutual information:劣势在于需要将连续的性状值离散化，损失信息
* ANOVA:通过p-value筛选feature，速度较慢，我们设计了加速ANOVA计算的算法。
* 基于模型的方法:基于广义线性模型或其他带有feature权重的机器学习模型，根据权重挑选feature

**降维：**

* PCA、SVD：劣势在于降维后的feature数量不能超过样本数量，一次性损失的feature过多
* Random projection:基于LSH的降维方式，速度较快

通过对问题的后续分析，我们发现对于预测绝大多数样本，基本的降维方法就已经够用 但是对于部分很难预测的样本，简单的特征选择方法也无法取得好的效果 我们根据后续开发的针对性的模型，设计了基于模型的特征选择方法，因为内容限制，不在这里使用。

**接下来分别使用ANOVA和Random projection演示特征选择和降维，对于后续的计算来说，选择其中一种就可以，也可以把不同的方法拼起来使用**

**我们会提供ANOVA、Random projection处理上面5000个snps后的数据，以及在完整数据集上用Random projection降维至10000个feature的数据。用于送入下一部分的回归模型。下面的三种方法的处理后的数据可以在feature\_selection文件夹下找到，存储格式为HDF5，可用h5py打开**

**ANOVA数据：**

feature\_selection/anova 包含三个性状各自的feature，大小为4000\*6210

**Random projection\(5000\)数据：**

feature\_selection/randomproj\_5000 从5000个SNPs降维得到，三个性状使用同一组feature,大小为1000\*6210

**Random projection\(whole SNPs\)数据：**

feature\_selection/randomproj\_whole 从所有SNPs降维得到，三个性状使用同一组feature，大小为10000\*6210

**3.1 ANOVA**

方差分析方法可以利用p值挑选feature 调用scipy.stats.f\_oneway,利用SNPs和性状可以很容易地计算出p-value，但是对于大量数据来说速度较慢 这里我们使用一种加速ANOVA计算的方法完成计算，相比于scipy.stats的方法可以提升计算速度数百倍。

```text
#加速ANOVA算法
def fast_anova_2bit(X, y):
y = y - y.mean()
y2 = y*y
N = X.shape[0]
SS_tot = np.sum(y2)
# 10, 01, 11
masks = [np.logical_and(X[:, 0::2], np.logical_not(X[:, 1::2])),
np.logical_and(np.logical_not(X[:, 0::2]), X[:, 1::2]),
np.logical_and(X[:, 0::2], X[:, 1::2])]
Ni = np.concatenate([np.sum(mask, axis=0) for mask in masks]).reshape((3, -1))
at_least_one = Ni > 0
SS_bn = [np.sum(y.reshape((-1, 1))*mask, axis=0) for mask in masks]
SS_bn = np.concatenate(SS_bn).reshape((3, -1))
SS_bn **= 2
SS_bn = np.where(at_least_one, SS_bn/Ni, 0)
SS_bn = np.sum(SS_bn, axis=0)
SS_wn = SS_tot - SS_bn
M = np.sum(at_least_one, axis=0)
DF_bn = M - 1
DF_wn = N - M
SS_bn /= DF_bn
SS_wn /= DF_wn
F = SS_bn/SS_wn

p_vals = np.ones(F.shape[0])
ind = np.nonzero(M == 2)[0]
if ind.shape[0] > 0:
p_vals[ind] = scipy.stats.f.sf(F[ind], 1, N - 2)
ind = np.nonzero(M == 3)[0]
if ind.shape[0] > 0:
p_vals[ind] = scipy.stats.f.sf(F[ind], 2, N - 3)
return F, p_vals
```

**注意X和y分别是什么**

我们需要输入进 fast\_anova\_2bit\(X, y\)的X是处理过的SNPs中前4754个样本的，y是trait1、trait2、trait3 因为需要分别预测三个性状，我们需要针对三个性状分别挑选特征

```text
geno_conv_train = geno_conv[:,:4754]
geno_conv_test = geno_conv[:,4754:]
print geno_conv_train.shape
print geno_conv_test.shape
```

```text
#分别计算三种性状下5000个SNPs做完ANOVA的p-value
F,pval_1 = fast_anova_2bit(geno_conv_train.T,trait1)
F,pval_2 = fast_anova_2bit(geno_conv_train.T,trait2)
F,pval_3 = fast_anova_2bit(geno_conv_train.T,trait3)
pval_1.shape
```

```text
fig, ax = plt.subplots(1,3, figsize=(15,3))
ax[0].hist(pval_1,bins = 50)
ax[0].set_title('5000 SNPs p-value for trait1 distribution')
ax[1].hist(pval_2,bins = 50)
ax[1].set_title('5000 SNPs p-value for trait2 distribution')
ax[2].hist(pval_3,bins = 50)
ax[2].set_title('5000 SNPs p-value for trait3 distribution')
```

```text
threshold1 = np.percentile(pval_1,40)
threshold2 = np.percentile(pval_2,40)
threshold3 = np.percentile(pval_3,40)
print 'threshold1: %f' %threshold1
print 'threshold2: %f' %threshold2
print 'threshold3: %f' %threshold3
```

```text
#返回符合条件的p-value的坐标，即可找到需要留下的SNPs的位置，注意每个SNPs占据两行
anova_index_1 = np.where(pval_1<threshold1)[0]
anova_index_1 = np.sort(np.concatenate((anova_index_1,anova_index_1 +1)))
anova_index_2 = np.where(pval_2<threshold2)[0]
anova_index_2 = np.sort(np.concatenate((anova_index_2,anova_index_2 +1)))
anova_index_3 = np.where(pval_3<threshold3)[0]
anova_index_3 = np.sort(np.concatenate((anova_index_3,anova_index_3 +1)))
```

```text
#根据p-value选取保留的SNPs，注意这一步要在所有的6210个样本上做
feature1_anova = np.take(geno_conv,anova_index_1,axis=0)
feature2_anova = np.take(geno_conv,anova_index_2,axis=0)
feature3_anova = np.take(geno_conv,anova_index_3,axis=0)
```

**3.2 Random projection**

Random projection 不依赖于性状，仅仅在原SNPs数据进行降维 Random projection可以使用scikit-learn下的sklearn.random\_projection模块计算 包括generate，transform和normalize等步骤 这里演示从5000个feature（10000行）降维

**3.2.1 generate**

产生一个稀疏矩阵

```text
#10000为操作前的feature个数：2*5000
X = np.zeros((2, 10000))
#确定降维后的个数，这里定为1000，使用sklearn random_projection 模块下的 SparseRandomProjection 函数
proj = SparseRandomProjection(1000)
proj.fit(X)
print proj.components_.shape
```

**3.2.2 transform**

```text
X= geno_conv.T
X_ = proj.transform(X)
```

**3.2.3 normalize**

对每个feature进行normalize，避免出现过大的值

```text
from sklearn.preprocessing import StandardScaler
normalized_feeature = StandardScaler().fit_transform(X_).T
```

最终大小为1000\*6210，结果在feature\_selection/randomproj\_5000

#### 4. 回归模型

这部分通过几个常用的机器学习模型对上一部处理过的feature进行拟合和预测 这里使用sklearn和xgboost提供的模块，这些模块具有很好的封装，使用风格统一，使用时可以查看其官方文档 这里不介绍具体的机器学习模型的算法原理，可以参考周志华老师的《机器学习》等书进行学习。

**4.1 机器学习模型**

接下来会使用一些常用的可以用于回归的机器学习模型，可以选择其中的一种或几种对feature\_selection/文件夹下的三种数据进行回归和预测。下面我们将几个模型列出，并且选择其中的一种作为示例，其他的模型可同理调用。

**4.2 评价指标**

我们使用$$r^2,pcc$$作为衡量预测结果的指标 $$r^2 = 1-\frac{SS_{res}}{SS_{tot}}$$ $$pcc = \frac{cov(X,Y)}{\sigma_X \sigma_Y} = \frac{E[(X-\mu_X)(Y-\mu_Y)]}{\sigma_X \sigma_Y}$$ 我们可以绘制结果的heatmap图，散点图等进行可视化。

**4.3 交叉验证**

交叉验证\(Cross validation\)可以帮助调参，寻找机器学习模型中的超参数 一般可以使用10折或者5折交叉验证，注意在最终预测时，使用调参后的模型在整个训练集上训练，这时不再交叉验证 因为交叉验证需要额外增加计算时间，因此这里只在整个训练集上训练一次，不再展示交叉验证的过程。

**如果深究的话，本问题还有其特殊性，可以设计特殊的交叉验证方式**

不同的样本具有关联性CV，有的样本可能来自同一亲本，而且训练集和测试集的划分并不是随机的 因此在真正解决这个问题的时候，需要考虑不同的抽样方式下的调参与训练，我们可以使用下图所示的几种抽样方式

```text
for method in ('random', 'by_female', 'by_male', 'cross'):
with h5py.File('data/cv_index.%s'%method, 'r') as f:
index_train = f['0/train'][:]
index_test = f['0/test'][:]
fig, ax = subplots(2, 1, figsize=(16, 6))
sampling_table = np.zeros(np.prod(parent_table.shape))
sampling_table[index_train] = 1
sampling_table = np.take(sampling_table, parent_table)
ax[0].matshow(sampling_table, cmap=cm.Greys)
ax[0].set_title('Training samples (%s)'%method)

sampling_table = np.zeros(np.prod(parent_table.shape))
sampling_table[index_test] = 1
sampling_table = np.take(sampling_table, parent_table)
ax[1].matshow(sampling_table, cmap=cm.Greys)
ax[1].set_title('Test samples (%s)'%method)
plt.tight_layout()
```

**4.4 准备数据**

sklearn的机器学习模型一般需要提供X和y以供模型训练，然后提供新的X，模型就可以预测新的y 通过前面的工作，我们获得了三种不同的X\(ANOVA、random projection\(5000/whole\)\)，我们还需要将X和y划分为训练集和测试集 注意我们需要分别对三个性状进行预测，因此ANOVA的X是三种

**评价模型的时候要注意，y\_true的部分值缺失**

**4.4.1 准备y**

先处理y，y 的train和test是统一的，不受方法影响

```text
#y 的train和test的统一的，不受方法影响
pheno_whole = pd.read_csv('data/emaize_pheno_whole',delimiter=',')
wholepheno = {}
for trait in ['trait1','trait2','trait3']:
wholepheno[trait] = np.array(pheno_whole[trait])
y_train = {}
y_test = {}
for trait in ['trait1','trait2','trait3']:
y_train[trait] = wholepheno[trait][:4754]
y_test[trait] = wholepheno[trait][4754:]
```

**4.4.2 准备X**

再处理X，使用ANOVA时X要区分不同性状，Random projection由于是与性状无关的降维方法，三种性状下的feature都一样 因为机器学习模型要求一般数据形式为sample\*feature，因此需要对结果转置

```text
def prepare_data(method):
if method == 'randomproj_5000':
with h5py.File('feature_selection/randomproj_5000') as f:
X_train = f['data'][:][:,:4754].T
X_test = f['data'][:][:,4754:].T
if method == 'randomproj_whole':
with h5py.File('feature_selection/randomproj_whole') as f:
X_train = f['X'][:][:4754,:]
X_test = f['X'][:][4754:,:]
if method == 'anova':
X_train = {}
X_test = {}
with h5py.File('feature_selection/anova') as f:
X_train['trait1'] = f['feature1'][:][:,:4754].T
X_test['trait1'] = f['feature1'][:][:,4754:].T
X_train['trait2'] = f['feature2'][:][:,:4754].T
X_test['trait2'] = f['feature2'][:][:,4754:].T
X_train['trait3'] = f['feature3'][:][:,:4754].T
X_test['trait3'] = f['feature3'][:][:,4754:].T
return X_train,X_test
```

选择三种方法之一作为X,注意ANOVA方法返回的X需要指明性状

```text
#查看anova方法的X_train X_test
X_train, X_test = prepare_data('anova')
print 'anova method X_train shape: %s' %(X_train['trait1'].shape,)
print 'anova method X_test shape: %s' %(X_test['trait1'].shape,)
```

```text
#查看random projection方法的X_train X_test
X_train, X_test = prepare_data('randomproj_5000')
print 'random projection method X_train shape: %s' %(X_train.shape,)
print 'random projection method X_test shape: %s' %(X_test.shape,)
```

**4.5 选择需要的机器学习模型**

接下来会提供多种机器学习模型，并且讲解其使用方法，可以选择自己喜欢的模型进行回归，也可以用sklearn或其他package提供的模型 模型包括：

* lr: Linear regression
* ridge: Ridge regression
* kr: Kernel Ridge regression
* rfr: Random Forest regression
* xgbr: XGBoost regression
* knr: K-nearest neigbour regression
* gpr: Gaussian Process regression

```text
def Model(model):
if model=='lr':
reg = LinearRegression()
#elif model=='xgbr':
# reg = XGBRegressor()
elif model=='ridge':
reg = Ridge()
elif model=='kr':
reg = KernelRidge(alpha = 10000, kernel = 'polynomial',degree = 3)
elif model=='knr':
reg = neighbors.KNeighborsRegressor(n_neighbors=4, algorithm='brute')
elif model=='rfr':
reg = RandomForestRegressor(n_estimators=10, criterion='mse', max_depth=12, n_jobs=5)
elif model=='gpr':
kernel = 1.0 * DotProduct(sigma_0=1.0)**4
reg = GaussianProcessRegressor(kernel = kernel, optimizer=None)
return reg
```

接下来可以使用三种特征（X）中的一种以及七种方法中的一种进行训练和预测 这里我们以randomproj\_5000作为特征，用Ridge作为回归模型演示 如果用anova得到的feature，需要注意不同性状的feature不一样

**某个机器学习模型的使用方法如下：reg.fit\(X,y\)用于拟合，reg.predict\(X\)用于预测。更多用法可以参考sklearn官方文档**

```text
X_train, X_test = prepare_data('randomproj_whole')
reg = Model('gpr')
y_predict = {}
y_predict_train = {}
for trait in tqdm(['trait1','trait2','trait3']):
reg.fit(X_train,y_train[trait])
y_predict[trait] = reg.predict(X_test)
y_predict_train[trait] = reg.predict(X_train)

X_test.shape
```

计算预测结果与真实值的$$r^2,pcc$$

```text
test_nonan = np.where(np.isnan(np.array(pheno_whole['trait1'])[4754:]) ==0)
pcc_train = {}
pcc_test = {}
r2_train = {}
r2_test = {}
for trait in ['trait1','trait2','trait3']:
pcc_test[trait] = pearsonr(y_predict[trait][test_nonan],np.array(pheno_whole[trait])[4754:][test_nonan])
pcc_train[trait] = pearsonr(y_predict_train[trait],y_train[trait])
r2_test[trait] = r2_score(y_predict[trait][test_nonan],np.array(pheno_whole[trait])[4754:][test_nonan])
r2_train[trait] = r2_score(y_predict_train[trait],y_train[trait])
```

```text
pcc_test
```

```text
pcc_train
```

```text
r2_test
```

```text
r2_train
```

可以看到预测结果并不是很好，在测试集上的pcc只有0.5左右。后续的分析可以发现，这是因为样本之间具有相关性导致的 具体的原因分析比较复杂，简单来说，因为这组测试集与训练集的样本的亲本之间亲缘关系较远，模型难以从SNPs得到的feature推断出亲本信息，导致预测结果较差。

绘制heatmap图观察预测结果 GPR具有很强的拟合能力，总可以在训练集上得到接近1的PCC

```text
fig, ax = plt.subplots(2,3, figsize=(15,10))
for i in range(3):
traits = ['trait1','trait2','trait3']
ax[0,i].scatter(y_predict[traits[i]][test_nonan],np.array(pheno_whole[traits[i]])[4754:][test_nonan])
ax[0,i].set_title('%s test set predict & true value plot' %traits[i])
line1 = [(-4, -4), (4, 4)]
(line1_xs, line1_ys) = zip(*line1)
ax[0,i].add_line(Line2D(line1_xs, line1_ys, linewidth=1, color='red'))
ax[0,i].set_xlim(left=-4, right=4)
ax[0,i].set_ylim(bottom=-4, top=4)
ax[1,i].scatter(y_predict_train[traits[i]],y_train[traits[i]])
ax[1,i].set_title('%s train set predict & true value plot' %traits[i])
ax[1,i].add_line(Line2D(line1_xs, line1_ys, linewidth=1, color='red'))
ax[1,i].set_xlim(left=-4, right=4)
ax[1,i].set_ylim(bottom=-4, top=4)
```

绘制完整真实值与预测值的heatmap图 从图中我们可以清晰地看出一个基本的模型的问题： 模型强烈地依赖已有信息进行预测，当未知样本的父本与已知训练集的亲缘关系较远时，模型只能依赖母本（横坐标）进行预测 导致预测的heatmap图有明显的与母本相关的特征，而实际上子代的性状更容易被父本主导

```text
wholepre = np.concatenate((y_predict['trait1'],y_predict['trait2'],y_predict['trait3'])).reshape(3,-1)
predictions = pd.DataFrame(wholepre.T)
predictions.columns = ['trait1', 'trait2', 'trait3']
predictions = predictions.set_index(np.arange(4754,6210))
def normalize_phenotype(x, range_pheno=4.0):
return (np.clip(x, -range_pheno, range_pheno) + range_pheno)/2.0/range_pheno
for trait in traits:
fig, ax = subplots(2, 1, figsize=(16, 6))
ax[0].matshow(np.take(np.ravel(normalize_phenotype(pheno_whole[trait].values)), parent_table), cmap=cm.RdBu_r)
ax[0].set_title('Phenotypes of whole true data (%s)'%trait)

trait_pred = np.full(phenotypes.shape[0], np.nan)
trait_pred[predictions.index.tolist()] = normalize_phenotype(predictions[trait].values)
ax[1].matshow(np.take(trait_pred, parent_table), cmap=cm.RdBu)
ax[1].set_title('Prediction on test data (%s 1)'%traits)
```

#### 后续分析

**不同样本具有不同的预测难度**

普通的机器学习模型在测试集上表现结果不好，但是通过多次的十字交叉抽样模拟，可以发现不同样本的预测难度不同，在大多数样本上，不需要专门设计的机器学习模型就足够表现很好

**样本之间具有关联性**

不服从一些基本的假设，比如线性模型下，残差并不是独立的，需要考虑问题的特殊性进行额外的设计。 由于存储空间和计算时间的限制，无法展示其他有效的方法，有兴趣的同学可以查找育种领域的其他模型进行尝试。

**复杂的机器学习模型并不一定有效**

育种领域目前最好的模型依然是线性模型，通过特殊的设计，可以考虑到亲缘关系、显著相关的SNP\(causal\)以及随机效应部分 而寻找合适的feature是预测结果好坏的决定性因素，至今没有非常好的方法。

我们通过模拟特殊的十字交叉抽样方式发现，虽然测试集的样本不好预测，但是大多数的样本使用简单的机器学习方法就可以在大多数样本上取得较好的结果 由于计算资源限制，下面直接展示模拟结果

我们使用一种特殊的十字交叉抽样，在训练集上抽样1000次，用来测试基本的机器学习模型结果 我们使用了2bit coding编码的SNPs,通过random projection降维至80000 然后使用Gaussian Process Regression作为回归模型

**具体展示结果见basic\_tutorial.ipynb相关部分**

不同父本的性状有显著差别，而子代的性状由于设计原因，主要由父本控制。 我们可以通过绘图查看不同父本的性状的变化

```text
male_index = np.ndarray([6210,]).astype('int')
for i in range(6210):
male_index[i] = int(np.array(phenotypes['pedigree'])[i].split('_')[2][1:])
male_trait1 = np.concatenate((male_index.reshape(1,-1),np.array(pheno_whole['trait1']).reshape(1,-1))).T
male_trait1_bysort = male_trait1[male_trait1[:,0].argsort()]
fig, ax = plt.subplots(figsize=(16,8))
ax.plot(male_trait1_bysort[:,1])
ax.set_title('different males have varied values')
```

以上内容简要地介绍了eMaize问题使用的一些基本的常用的机器学习方法，包括数据预处理、特征选择、降维、回归以及分析。本教程还顺便展示了一些python常用的工具包的使用，读者有时间可以慢慢体会其中的具体操作，因为jupyter notebook的可视化与交互性很强，读者可以方便地查看中间步骤的数据情况，更好地理解代码所进行的操作。 由于实际工作的步骤、数据量、变量等问题，还需要慎重考虑计算时间、任务管理等工作 想要预测较难预测的样本仅仅靠常规的机器学习方法并不够用，将机器学习应用于生物学问题时，不能简单套用模型，还需要根据问题进行针对性的设计，才有可能取得更好的结果。

