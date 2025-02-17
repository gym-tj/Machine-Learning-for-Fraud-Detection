# 信用卡诈骗强化学习
##  解决的关键问题
&nbsp;　　关键问题通过利用信用卡的历史交易数据，进行机器学习构建信用卡反欺诈预测模型，提前发现客户信用卡被盗刷的事件。

&nbsp;　　问题现状：检测支付卡交易中的欺诈模式是一个非常困难的问题。随着支付卡交易产生的数据量不断增加，我们很难检测交易数据集中的欺诈模式，这些模式通常具有大量样本、多个维度和在线更新的特征，所以，通过机器学习的方法进行模型训练，得到一个能够直接识别信用卡诈骗的模型十分有社会意义。
</font>![](https://cdn.nlark.com/yuque/0/2023/png/40852163/1703431008731-c4f6b7ef-9de1-4cdf-b391-a63f37f4670a.png)

## 数据分析与数据处理
&nbsp;　　通过机器学习数据集包含由欧洲持卡人于2013年9月使用信用卡进行的数据。此数据集显示两天内发生的交易。特征V1至V28是经过PCA处理后的数据，原始数据未公开，而特Time、Amount的数据规格与其他特征差别较大，需要对其做特征缩放，将特征缩放至同一个规格。

&nbsp;　　通过热力图展示了信用卡正常用户（NonFraud）和被盗刷用户（Fraud）的数据特征之间的相关性的区别，是通过皮尔逊相关系数计算了正常用户和被盗刷用户的数据特征之间的相关性的。颜色越深，代表皮尔逊相关系数绝对值越大意味着这两个特征之间相关性越大。
![](https://cdn.nlark.com/yuque/0/2023/png/40852163/1703088466314-54e019e0-2ebd-4578-8bb1-a2a5f70f173b.png)![](https://cdn.nlark.com/yuque/0/2023/png/40852163/1703087860580-830cb147-0ce5-4672-8ca3-8672fda47de6.png)

&nbsp;　　可以看到Normal和Fraud数据的相关性热力图有很大的区别，计算正常和欺诈数据的相关性矩阵，计算两者每个特征的平均差异，得到影响最大的18个特征进行后面的训练。<font style="color:rgb(51, 51, 51);">      284,807笔交易中有492笔被盗刷，可以看出数据集非常不平衡，积极的类（被盗刷）占所有交易的0.172</font>。![](https://cdn.nlark.com/yuque/0/2023/png/40852163/1703089668613-8c87bf02-c02a-472f-b56b-933f45bace24.png)
&nbsp;　　如果直接进行模型训练的话，使用无论是随机森林还是logist模型以及支持向量机都会出现虽然得分很高，性能报告优秀，但是输出混淆矩阵发现并不能诊断识别异常数据。   如下（未经过SMOTE方法平衡样本训练出来的随机森林）  
  &nbsp;　　![](https://cdn.nlark.com/yuque/0/2023/png/40852163/1703425267857-abd6a7c1-a456-4bbf-9d49-3b19fdc455a8.png)
&nbsp;　　

```plain
precision    recall  f1-score   support

           0       1.00      1.00      1.00     85296
           1       0.88      0.62      0.73       147

    accuracy                           1.00     85443
   macro avg       0.94      0.81      0.86     85443
weighted avg       1.00      1.00      1.00     85443

[[85283    13]
 [   56    91]]
```

   于是要进行不平衡数据处理，通过SMOTE方法平衡样本（相当于变相加大被盗刷的数据的学习）但是会有模型过拟合的风险，所以在最后探索了一种将正常数据进行训练，然后将异常数据作为异常值的无监督学习的方法。

## 比较多种模型
  LogisticRegression

      

```python
from sklearn.model_selection import GridSearchCV
from sklearn.linear_model import LogisticRegression

#网格搜索调优参数
param_grid_1 = {'C': [1, 10, 100, 1000,],
                            'penalty': [ 'l1', 'l2']}

# 确定模型LogisticRegression，和参数组合param_grid ，cv指定5折
grid_search_1 = GridSearchCV(LogisticRegression(),  param_grid_1, cv=5) 

# 使用训练集学习算法
grid_search_1.fit(X_train, y_train) 
results_1 = pd.DataFrame(grid_search_1.cv_results_) 
best_1 = np.argmax(results_1.mean_test_score.values)
print("Best parameters: {}".format(grid_search_1.best_params_))
print("Best cross-validation score: {:.5f}".format(grid_search_1.best_score_))

# 在测试集上进行预测
y_pred_log = grid_search_1.predict(X_test)

```

```plain


precision    recall  f1-score   support

0       0.94      0.98      0.96     85172
1       0.98      0.93      0.96     85417

accuracy                           0.96    170589
macro avg       0.96      0.96      0.96    170589
weighted avg       0.96      0.96      0.96    170589

[[83845  1327]
[ 5733 79684]]
```


RandomForestClassifier

```python
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import classification_report
from sklearn.model_selection import GridSearchCV

# 定义随机森林分类器
rfc = RandomForestClassifier(random_state=42)

# 定义参数搜索空间
param_grid = {
    'n_estimators': [5, 10, 20],
}

# 构建网格搜索模型
grid_search = GridSearchCV(rfc, param_grid, cv=5)
grid_search.fit(X_train, y_train)
best_model = grid_search.best_estimator_

# 在测试集上进行预测
y_pred_rfc = best_model.predict(X_test)

```



得到的混淆矩阵为，并进行可视化

```plain
[[85152    20]
 [    3 85414]]
```

![](https://cdn.nlark.com/yuque/0/2023/png/40852163/1703347921641-2aacd7b6-a7e1-48cf-ac94-ea4a81647c09.png)

```plain
precision    recall  f1-score   support

           0       1.00      1.00      1.00     85172
           1       1.00      1.00      1.00     85417

    accuracy                           1.00    170589
   macro avg       1.00      1.00      1.00    170589
weighted avg       1.00      1.00      1.00    170589
```

  
 

  LinearSVC

```python
from sklearn.svm import LinearSVC
from sklearn.metrics import classification_report
from sklearn.model_selection import GridSearchCV

# 定义线性支持向量机模型
svc = LinearSVC(random_state=42, dual=False)
# 定义参数搜索空间
param_grid = [
    {'kernel':['rbf'],'gama':[1,0.1,0.1,0.01],'C':[1,10,100,1000]},
    {'kernel':['linear'],'C':[0.1,1,10,100]},
    {'kernel':['poly'],'gama':[10,1,0.1,0.01],'C':[0.001,0.01,0.1,1,10],
    'degree':[2,4,6]}
]

# 构建网格搜索模型
grid_search = GridSearchCV(svc, param_grid, cv=5)

# 在训练集上进行参数搜索和训练
grid_search.fit(X_train, y_train)

# 获取最佳模型
best_model = grid_search.best_estimator_

# 在测试集上进行预测
y_pred_svc = best_model.predict(X_test)

# 评估模型性能
print(classification_report(y_test, y_pred_svc))
```

```plain
            precision    recall  f1-score   support

           0       0.93      0.99      0.96     85172
           1       0.99      0.92      0.95     85417

    accuracy                           0.95    170589
   macro avg       0.96      0.95      0.95    170589
weighted avg       0.96      0.95      0.95    170589
array([[84014,  1158],
       [ 6645, 78772]], dtype=int64)
```

## 反思解决问题的过程

&nbsp;　　在这个项目中，我们要处理一个异常样本极少的二元问题，如果直接用原始的数据进行模型训练，会发现虽然模型的性能报告很优秀，但是从混淆矩阵中可以看出由于异常样本本来就少，大多数还被预测为正常值了，这个模型就失去其功能了，无法解决我们的问题。于是想到既然这个样本少，能不能加大对异常数据的学习，于是通过SWOTE方法平衡正负样本来实现进行数据的模型训练，发现果然数据的混淆矩阵大大改善，异常值也能被识别为异常了。但是这个方法的前提是SWOTE生成的异常数据和原异常数据至少有相关性，即，其生成的数据仍不能失真。

&nbsp;　　图中数字为诈骗数据转化前和转化后特征之间的互信息（Mutual Information）矩阵，互信息是一种非参数的方法，用于衡量两个变量之间的关联程度。它可以捕捉到任何类型的关系，包括线性和非线性关系。通过绘制出被诈骗数据的互信息矩阵的方式说明至少经过SWOTE处理后的数据没有太失真，可能会有原数据的重复或者再现，但是那也比失真好。



&nbsp;　　假设原始数据为original_data，SMOTE处理后的数据为smote_data（互信息变量关联矩阵）</font>
![](https://cdn.nlark.com/yuque/0/2023/png/40852163/1703349652197-06e4a0cf-7aaf-482b-a4f4-2f844e157901.png)![](https://cdn.nlark.com/yuque/0/2023/png/40852163/1703349663439-2f1116a5-283f-468b-9535-7b8c365d0543.png)

## 改进思路
&nbsp;　　之前的三个模型的处理思路都是将被诈骗和正常标为1和0两类，进行聚类分析，但是在经过老师点播后发现：既然这是个小概率事件，那么可以将被诈骗的数据作为异常数据处理。这样就不用刻意加大对诈骗数据的学习，具有可以有效防止模型过拟合的优点，而且，在操作过程中发现，这样做还有一个优点，那就是可以识别出可能之前没有出现过的诈骗新的行式，更具有社会意义。

&nbsp;　　例如新了解到的异常森林就是一种用于检测异常值的无监督学习算法但是在实际操作中发现，可能是因为数据特征太多，或者是正常数据的方差过大，导致无监督学习正常数据得到的模型并不能识别出异常数据或者得到的数据混淆矩阵还不如随机森林，说明还存在改进空间和优化。

```python
from sklearn.ensemble import IsolationForest
from sklearn.metrics import recall_score

# 创建异常森林模型
clf = IsolationForest(n_estimators=100, contamination=0.05, random_state=42)

# 使用异常森林模型进行训练和预测
clf.fit(X)
y_pred = clf.predict(X)

# 计算召回率
recall = recall_score(y, y_pred, average='weighted')  # 或者 average='macro'

# 输出召回率
print("Recall: ", recall)
from sklearn.metrics import confusion_matrix

# 计算混淆矩阵
confusion_mat = confusion_matrix(y, y_pred)  # y为真实标签

# 输出混淆矩阵
print("Confusion matrix:")
print(confusion_mat)
```

```plain
[[85283    13]
 [   46    81]]
```
