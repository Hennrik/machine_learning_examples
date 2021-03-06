# Machine learning using Scikit-learn
This Readme file illustrates how to implement several machine learning algorithms using Scikit-learn based on the [Mike's blog](https://xyclade.github.io/MachineLearning/).

* [Linear Regression](#linear-regression)
* [Support Vector Machine](#support-vector-machine)
  * [Gaussian Kernel](#gaussian-kernel)
  * [Polynomial Kernel](#polynomial-kernel)
* [Principal Component Analysis](#principal-component-analysis)
* [Naive Bayes](#naive-bayes)
* [K Nearest Neighbor](#k-nearest-neighbor)
* [Functional Programming Elements](#functional-programming-elements)

## Linear Regression
Importing and reading csv data
```python
dataLocation=r'C:\Users\Yanbo Huang\Desktop\Python exercises\OLS_Regression_Example_3.csv'
data=pd.read_csv(dataLocation)
```
Data visualization  

<img src="imgs/lr1.jpg" height="300" >


Transfering gender property to numeric data, _inch_ to _cm_ and _bound_ to _kg_
```python
data['Gender']=(data['Gender']!='Male').astype(int)
data['Height']*=2.54
data['Weight']*=0.45359237
```
Linear regression model building
```python
Features=data[['Gender','Height']].values
Label=data['Weight'].values
regr=linear_model.LinearRegression()
regr.fit(Features,Label)
```
Weight prediction of women and man with 170cm hight  
```python
featuresTest=[[0,170.0],[1,170.0]]
print 'Input features:\n',featuresTest
print 'predictions:\n',regr.predict(featuresTest)
```
Model coefficients and weight predicitons with unit in _kg_:
> Coefficients:   
[-8.78958164,  1.06736021]  
Input features:  
[[0, 170.0], [1, 170.0]]  
predictions:    
[ **79.1453856**, **70.35580396**]

## Support Vector Machine
### Gaussian Kernel
Data visualization:


<img src="imgs/svm1.jpg" height="300" >

Aparently, we can not use linear SVM classifier to solve this problem. One alternative way is to use Gaussian kernal to map the original data into some complex high dimensional space to make it sparateble.  

Here we show how to realize this.  

Splitting the original data into three parts: train,cross validation,and test:
```python
# split data into :train and test
train,test=train_test_split(data,test_size=0.2)
# split train into :train and cv 
train,cv=train_test_split(train,test_size=0.2)
```
> **Dataset discriptions:**  
_Train_ : Train the SVM model  
_CV_ :  Choose model perameters  
_Test_: Give reliable test error for reporting

Exacting features and labels from different datasets:
```python
features_train=train[['x','y']].values
label_train=train['label '].values
features_cv=cv[['x','y']].values
label_cv=cv['label '].values
features_test=test[['x','y']].values
label_test=test['label '].values
```
Giving a list of parameters:
```python
sigmas=[0.001,0.01,0.1,0.2,0.5,1.0,2.0,3.0,10.0]
Cs=[0.001,0.01,0.1,0.2,0.5,1.0,2.0,3.0,10.0,100.0,100.0]
gammas=1.0/(2*(np.array(sigmas)**2))
```
Choosing the corraton rate and gamma when getting lowest _false classified ratio_ in CV dataset:  
(**_Note_**: Here we avoid using training dataset to select our model parameters. Otherwise,it is eassy to overfit the data)
```python
FCR=[]  # false classified ratio
for p in Cs:
    for g in gammas:
        svc=svm.SVC(kernel='rbf',C=p,gamma=g).fit(features_train,label_train)
        label_predict=svc.predict(features_cv)
        falseRatio=float(sum((label_cv-label_predict)**2))/len(label_cv)
        FCR.append(falseRatio)
FCR=np.array(FCR).reshape(len(Cs),len(sigmas))
print 'minimum false classified ratio in CV dataset:{0:.4}% '.format(FCR.min()*100)
svm_C=Cs[np.array(np.where(FCR==FCR.min()))[0,0]]
svm_gamma=gammas[np.array(np.where(FCR==FCR.min()))[1,0]]
# FCR table with index in C and sigma
df = pd.DataFrame(FCR, index=Cs, columns=sigmas)
df.columns.name='C,S->'  
```
The **df** in the code stands for FCR table with index in **C** and **sigma**  
(**Note**: We got the table based on _CV dataset_ instead of _training dataset_)  
Here we got :  


<img src="imgs/svm2.jpg" height="300">
> minimum false classified ratio in CV dataset:  **5.0%**  

Then using model parameters which resulting this **5.0%** test error to train the model and make predication on our test dataset:
```python
svc=svm.SVC(kernel='rbf',C=svm_C,gamma=svm_gamma).fit(features_test,label_test)
label_predict=svc_final.predict(features_test)
falseRatio_test=float(sum((label_test-label_predict)**2))/len(label_test)
print 'final false classified ratio for this model is :{0:.4}%'.format(falseRatio_test*100)
```
Here we got :
> final false classified ratio for this model is :**5.8%**

### Polynomial Kernel
Data visualization  


<img src="imgs/svm3.jpg" height="300">  

<img src="imgs/svm4.jpg" height="300">

Using gaussian kernal we got:

<img src="imgs/svm5.jpg" height="300">

> minimum false classified ratio in test dataset:**19.16%** 

The ratio of falsely classified data in testing dataset is extremely  high by using gaussian kernal SVM.

Alternatively, we implemented polynomial kernel in our SVM model.   
```python
degrees=[2,3,4,5]
Cs=[0.001,0.01,0.1,0.2,0.5,1.0,2.0,3.0,10.0,100.0,100.0]

FCR=[]  # false classified ratio
# Gaussian kernal model
for c in Cs:
    for d in degrees:
        svc=svm.SVC(kernel='poly',degree=d,C=c).fit(features_train,label_train)
        label_predict=svc.predict(features_test)
        falseRatio=float(sum((label_test-label_predict)**2))/len(label_test)
        FCR.append(falseRatio)
FCR=np.array(FCR).reshape(len(Cs),len(degrees))
print 'minimum false classified ratio in test dataset:{0:.4}% '.format(FCR.min()*100)
svm_C=Cs[np.array(np.where(FCR==FCR.min()))[0,0]]
svm_degree=degrees[np.array(np.where(FCR==FCR.min()))[1,0]]

df = pd.DataFrame(FCR, index=Cs, columns=degrees)
df.columns.name='C,d->'
```
Here we got the FCR table and test error:  
<img src="imgs/svm6.jpg" height="300">
> minimum false classified ratio in test dataset:**0.0%** 

As we can see, the polynomial kernel is superior to gaussian kernel in this case.  

## Principal component analysis  

Get the example data:  
```python
dataLocation=r'C:\Users\Yanbo Huang\Desktop\Python exercises\PCA_Example_1.csv'
data=pd.read_csv(dataLocation)  
```
Reshape the data into 24 stocks 

```python
df=data.pivot(index='Date', columns='Stock', values='Close')
```
Train the PCA model which merges the 24 features into 1 single feature

```python
pca=PCA(n_components=1)
outputData=pca.fit_transform(df)
```
Plot the results,with the feature value on the vertical axis and the individual days on the horizontal axis.
<img src='imgs\pca1.jpg' height='300'>

Load Dow Jones Index for verification

```python
DJIdataLocation=r'C:\Users\Yanbo Huang\Desktop\Python exercises\PCA_Example_2.csv'
DJI=pd.read_csv(DJIdataLocation)
#sort data by date
DJI=DJI[['Date','Close']].sort('Date')['Close']
```
Normolize data

```python
DJI=(DJI-DJI.mean())/(DJI.max()-DJI.min())
pcaData=(outputData-outputData.mean())/(outputData.max()-outputData.min())
```

Plot both pac outputdata and DJI for comparision

<img src='imgs\pca2.jpg' height='300'>

## Naive Bayes
Get files from directory
```python
def getFilesFromDir (path):
    dir_content = os.listdir(path)
    dir_clean = filter(lambda x: (".DS_Store" not in x) and ("cmds" not in x), dir_content)
    msg = map(lambda x: getMessage(path + '/' + x), dir_clean)
    return msg
```
Get message from file
```python
def getMessage (file_list, path, amount_of_samples):
    fo = open (path)
    lines = fo.readlines()
    fo.close()
    #...
    #Regular expresssion process and keep words and spaces in the string
    lines = re.sub('3D', '', lines)
    lines = re.sub(r'([^\s\w]|_)+', '', lines)
    #...
```
Get stopwords
```python
def getStopWords (path):
    fo = open (path)
    lines = fo.readlines()
    lines = map(lambda x: str.replace(x, '\n', ''), lines)
    fo.close()
    return lines
```
Establish Term Document Matrix using sklearn.feature_extraction.text.TfidfVectorizer

Get ham/spam features, here take ham as example
```python
vector = TfidfVectorizer(stop_words = stop_words_given,
                         analyzer='word',
                         decode_error = 'ignore',
                         max_features = amountOfFeaturesToTake)
words_counts = vector.fit_transform(hamTrainDir)
words_counts = words_counts.toarray()
feature_ham = vector.get_feature_names()
```
Term Document Matrix
```python 
vector3 = TfidfVectorizer(decode_error = 'ignore',
                          vocabulary = feature_email)
words_counts3 = vector3.fit_transform(emailTrainDir)
words_counts3 = words_counts3.toarray()
```
Train: multinomial Naive Bayes with y is the corresponding label of the train set, in which ham label is 0, spam label is 1.
```python 
clf = MultinomialNB(alpha=1.0, class_prior=None, fit_prior=True).fit(words_counts3, y)
```
Ham/spam test, here take ham as example
```python
new_counts = vector3.transform(hamTestDir)
```
Ham/spam prediction and accuracy score, here take ham as example
```python
predicted = clf.predict(new_counts)
accu_score = clf.score(new_counts, y1)
```

Run the code several times and change amount of features to take, we get the following results:
###Ham Result
| Amount of Features to take        | Ham (Correct)           | Spam (Incorrect) |
| ----------------------------------|:-----------------------:| -----:|
| 50                                | 94.07%                  | 5.93% |
| 100                               | 96.50%                  | 3.50% |
| 200                               | 97.71%                  | 2.29% |
| 400                               | 98.14%                  | 1.86% |
###Spam Result
| Amount of Features to take        | Spam (Correct)          | Ham (Incorrect) |
| ----------------------------------|:-----------------------:| -----:|
| 50                                | 82.82%                  | 17.18%|
| 100                               | 89.48%                  | 10.52%|
| 200                               | 90.05%                  | 9.95% |
| 400                               | 93.27%                  | 6.73% |


## K Nearest Neighbor
Importing and reading csv data
```python
dataLocation=r'/Users/yuliang/Documents/MachineLearning-master/Example_Data/KNN_Example_1.csv'
data=pd.read_csv(dataLocation)
```
Scatter plot of original dataset
```python 
plt.scatter(data[data['Label']==0]['X'],data[data['Label']==0]['Y'],color='magenta',label='Alpha')
plt.scatter(data[data['Label']==1]['X'],data[data['Label']==1]['Y'],color='green',label='Beta')
```
Data visualization

<img src='imgs/knn.png' height='300'>

Split dataset using 2 fold
```python
train,test=train_test_split(data,test_size=0.5)
neigh = KNeighborsClassifier(n_neighbors=3,weights='distance')
```
Get features and labels from dataset
```python
features_train=train[['X','Y']].values
label_train=train['Label'].values
features_test=test[['X','Y']].values
label_test=test['Label'].values
```
Train model
```python
neigh.fit(features_train,label_train)
```
Use test dataset to calculate mean accuracy of trained model
```python
mean_accuracy=neigh.score(features_test,label_test)
```
Predict unkown points
```python
unknownPoint=np.array([[5.3,4.3]])
prediction=neigh.predict(unknownPoint)
```
result: 
the mean accuracy on the given test data and labels is 0.860000
Internet Service Provider Alpha

## Functional Programming Elements

The most of workload of this project lie on dataset pre-processing. In the pre-processing part, we implemented some functional programming concepts when dealing with operations ralated to *list*, such as *map*, *reduce*, *filter*, and *list comprehension*. Here we list some examples where we were thinking in functional programming. 

In the *svm_gaussian* part,when dealing with the kernal perameter *gamma* and *sigma*, we have: *gamma= 1/(2xsigma^2)*. We are supplied with a list of *sigma*. Here we used the function *map* to implify *gamma= 1/(2xsigma^2)* for every element in *sigma* list:
```python
gammas = map(lambda x: 1.0/(2.0*x**2), sigmas)
```
Similarly, the concept of **list comprehension** was used in *svm_poly* part:

```python
gammas = [ 1.0/(2.0*x**2) for x in sigmas ]
```

In both *svm_gaussian* and *svm_poly* parts, we used **reduce** function to find the min value of FCR:

```python
FCR_min = reduce(lambda a,b: a if (a < b) else b, FCR)
```

When realizing the *Naive Bayes* algorithm, the files given us have two uselessful file types, *.DS_store* and *.cmds*. Here, we used **filter** function to ignore these files when reading the initial datasets:

```python
dir_clean = filter(lambda x: (".DS_Store" not in x) and ("cmds" not in x), dir_content)
msg = map(lambda x: getMessage(path + '/' + x),dir_clean)
``` 
What's more, some parts in *Naive Bayes* implementation used **map** fucntion to deal with path operations for replecing *for* loop,such as:

```python
msg = map(lambda x: getMessage(path + '/' + x), dir_clean)
```

```python
lines = map(lambda x: str.replace(x, '\n', ''), lines)
```
