# 속도비교
이성령  
2017년 7월 21일  


```r
data(Sonar)
indexTrain <- createDataPartition(1:nrow(Sonar), p = .7, list = F)
training <- Sonar[ indexTrain, ]
testing  <- Sonar[-indexTrain, ]
#10-fold cross validation 을 5번 반복하여 가장 좋은 후보의 파라미터 그리드를 찾게 해주는 일종의 장치를 만드는 코드이다.
fitControl <- trainControl(method = "repeatedcv", number = 10, repeats = 5)

rf_fit <- train(Class ~ ., data = training, method = "rf", trControl = fitControl, verbose = F)
```

```
## Loading required package: randomForest
```

```
## Warning: package 'randomForest' was built under R version 3.4.1
```

```
## randomForest 4.6-12
```

```
## Type rfNews() to see new features/changes/bug fixes.
```

```
## 
## Attaching package: 'randomForest'
```

```
## The following object is masked from 'package:dplyr':
## 
##     combine
```

```
## The following object is masked from 'package:ggplot2':
## 
##     margin
```

```r
predict(rf_fit, newdata = testing) %>% confusionMatrix(testing$Class)
```

```
## Confusion Matrix and Statistics
## 
##           Reference
## Prediction  M  R
##          M 28  3
##          R  6 23
##                                          
##                Accuracy : 0.85           
##                  95% CI : (0.7343, 0.929)
##     No Information Rate : 0.5667         
##     P-Value [Acc > NIR] : 2.679e-06      
##                                          
##                   Kappa : 0.6987         
##  Mcnemar's Test P-Value : 0.505          
##                                          
##             Sensitivity : 0.8235         
##             Specificity : 0.8846         
##          Pos Pred Value : 0.9032         
##          Neg Pred Value : 0.7931         
##              Prevalence : 0.5667         
##          Detection Rate : 0.4667         
##    Detection Prevalence : 0.5167         
##       Balanced Accuracy : 0.8541         
##                                          
##        'Positive' Class : M              
## 
```

```r
#아래 코드는 mtry 의 후보를 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 로 바꾸어 설정하고 이 중에서 채택해 보는 코드이다.
customGrid <- expand.grid(mtry = 1:10)

rf_fit2 <- train(Class ~ ., data = training, method = "rf", trControl = fitControl, tuneGrid = customGrid, verbose = F)

#수동으로 튜닝파라미터 조합개수를 늘려볼 필요가 있다. 이땐 train() 함수의 tuneLength 인자를 이용하면 된다.
fitControl <- trainControl(method = "repeatedcv", number = 10, repeats = 5, search = "random")

getDoParWorkers()
```

```
## [1] 1
```

```r
# 속도 비교 해보기 일반 train
time <- system.time({
  train(Class ~ ., data = training, method = "rf", trControl = fitControl, tuneGrid = customGrid, verbose = F)
})
# 병렬 붙이고 train 
registerDoParallel(detectCores()-1)
getDoParWorkers()
```

```
## [1] 3
```

```r
time2 <- system.time({
  k2 <- train(Class ~ ., data = training, method = "rf", trControl = fitControl, tuneGrid = customGrid, verbose = F)
})
customGrid <- expand.grid(mtry = 1:10)
fitControl <- trainControl(method = "repeatedcv", number = 10, repeats = 5)
#foreach + 병렬 train 
library(foreach)
time3 <- system.time(model<- foreach(mtry2=1:10,number2=10,a=5,.combine=c,.packages="caret",.multicombine = TRUE) %dopar% 
                        { train(Class ~ ., data = training, method = "rf", 
                                    trControl =trainControl(method = "repeatedcv", number=number2,repeats= a), 
                                     tuneGrid =expand.grid(mtry =mtry2),verbose = F)
                        }
                      )
model
```

```
## Random Forest 
## 
## 148 samples
##  60 predictor
##   2 classes: 'M', 'R' 
## 
## No pre-processing
## Resampling: Cross-Validated (10 fold, repeated 5 times) 
## Summary of sample sizes: 134, 133, 133, 132, 133, 133, ... 
## Resampling results:
## 
##   Accuracy   Kappa    
##   0.8293333  0.6559737
## 
## Tuning parameter 'mtry' was held constant at a value of 1
```

```r
time3
```

```
##    user  system elapsed 
##    0.09    0.01    9.56
```

```r
k2
```

```
## Random Forest 
## 
## 148 samples
##  60 predictor
##   2 classes: 'M', 'R' 
## 
## No pre-processing
## Resampling: Cross-Validated (10 fold, repeated 5 times) 
## Summary of sample sizes: 134, 134, 133, 132, 133, 133, ... 
## Resampling results across tuning parameters:
## 
##   mtry  Accuracy   Kappa    
##    1    0.8307738  0.6579267
##    2    0.8240000  0.6442741
##    3    0.8175476  0.6309736
##    4    0.8239167  0.6444238
##    5    0.8202024  0.6360802
##    6    0.8186786  0.6334046
##    7    0.8268690  0.6501862
##    8    0.8176310  0.6310750
##    9    0.8104881  0.6171207
##   10    0.8133571  0.6224512
## 
## Accuracy was used to select the optimal model using  the largest value.
## The final value used for the model was mtry = 1.
```

```r
k <- rbind(time,time2,time3)
timetable <- k[,1:3]
rownames(timetable) <- c("일반 train","병렬 설정","병렬+foreach")
timetable
```

```
##              user.self sys.self elapsed
## 일반 train       77.65     0.08   77.81
## 병렬 설정         0.93     0.03   33.93
## 병렬+foreach      0.09     0.01    9.56
```

